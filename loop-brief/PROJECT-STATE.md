# Loop Brief — Project State & Handoff

A working, deployed FL2 cross-team alignment tool: a RAG pipeline that drafts a
weekly pre-meeting brief from multiple (mocked) work sources, with a
human-in-the-loop review interface. Built as a portfolio piece demonstrating
production-shaped agentic-workflow architecture.

> **Status in one line:** End-to-end pipeline is live and demonstrable. Data
> sources are realistic fixtures (not live integrations). Retrieval, synthesis,
> the human-review UI, and a hardened backend proxy are all functional and
> deployed.

---

## The scenario

Fictional B2B SaaS company **Loopline** (customer onboarding/success tool, ~80
people). Three engineering teams — **Anchor** (core product), **Relay**
(integrations), **Signal** (data/analytics) — plus stakeholders in Marketing,
Customer Support, and Finance. They hold a recurring **Wednesday cross-team sync
called "The Loop."** The tool prepares the brief for that meeting.

Current data window: **Sprint 12 / Week 47** (mid-sprint, Wed Nov 20 framing).
Only Sprint 12 data is ingested so far.

The fixtures encode one main story thread (Relay webhook schema delay → blocks
Anchor's ANC-204 → surfaces in CS tickets, Slack, retro) plus two minor threads
(Signal/Finance churn-dashboard timing squeeze; Marketing/Relay Salesforce v2
launch-date risk).

---

## Architecture (what lives where)

```
Visitor → ancatrif.com/loop-brief/  (frontend page, GitHub Pages)
            │  default: loads an embedded real sample brief instantly (free, no backend)
            │  "Try it live" ON → calls the proxy:
            ▼
          loopline-brief-proxy.onrender.com/brief  (Express proxy on Render, free tier)
            │  hides n8n URL, locks CORS to ancatrif.com, rate-limits, validates input
            ▼
          n8n webhook  (ancatrif.app.n8n.cloud/webhook/loopline-brief)
            │  the actual pipeline:
            │   1. 5 fixed standing queries
            │   2. embed each (Voyage AI)
            │   3. retrieve top-6 each from Supabase pgvector
            │   4. tag + de-duplicate (~23 unique chunks)
            │   5. (+ optional facilitator observation) build prompt
            │   6. Claude synthesizes structured brief (lead/watch/fyi + citations)
            │   7. parse + attach sources, respond
            ▼
          brief JSON returns up the chain to the page
```

### Components & where they live
- **Frontend:** `ancatrif.github.io` repo → `loop-brief/index.html`. Single
  self-contained HTML file (React via CDN + Babel). Served at
  `ancatrif.com/loop-brief/`.
- **Backend proxy:** `loopline-brief-proxy` repo (private) → deployed as a Render
  Web Service. `server.js` (Express). Env vars on Render hold the secrets.
- **n8n workflow:** "Loopline Brief Agent" on n8n Cloud. Webhook-triggered,
  request/response style (Webhook node receives, Respond-to-Webhook node returns).
- **Vector store:** Supabase project, `loop_chunks` table (pgvector, 1024-dim),
  RLS enabled + locked (backend-only, service role bypasses). Similarity search
  via `match_loop_chunks` RPC.
- **Embeddings:** Voyage AI (`voyage-3.5`), Anthropic's recommended partner.
- **LLM:** Claude (`claude-sonnet-4-6`) for synthesis.
- **Fixtures:** `loopline-sprint12-fixtures` (JSON + markdown), 5 sources:
  Jira ×3 teams, Confluence (retros ×3 + decision log), Intercom, Marketing
  calendar, Slack. 40 chunks total after chunking.
- **Ingestion scripts:** `loopline-rag` project — `chunkers.js`, `ingest.js`,
  `query.js`, `brief-agent.js` (original local version of the agent), `dry-run.js`.

### Local working folder
Desktop → `FL2 Alignment Agent/` contains `loopline-rag/` and
`loopline-sprint12-fixtures/` as siblings (relative paths depend on this).

---

## How the pipeline "thinks" (two layers)
- **Retrieval (mechanical):** 5 standing queries, each embedded, each pulls top-6
  semantically-nearest chunks from pgvector. De-duplicated to ~23. Decides what's
  *available* to reason over.
- **Synthesis (judgment):** Claude receives all ~23 numbered chunks, picks the
  single most urgent (Lead Item), groups the rest (Watch List / FYI), and cites
  which chunk numbers support each claim. Decides what's *important*. The optional
  facilitator **observation** influences THIS layer only — it re-prioritizes among
  already-retrieved chunks (it does not change retrieval).

**The 5 standing queries:** cross-team blockers/dependencies at risk;
client-impacting bugs/issues this week; upcoming external commitments;
unresolved retro action items from last sprint; decisions affecting multiple teams.

---

## The UI (facilitator review)
- Opens empty; **Generate brief** runs a short simulated "working" sequence then
  shows the brief. In sample mode this replays a real embedded run (instant, free,
  honest — it genuinely was produced by the pipeline). Live mode calls the proxy.
- **Editable** lead/watch/fyi cards (click any title or body). Badge flips to
  "Edited."
- **Citation pills** read e.g. `Jira · ANC-204`, `Confluence · Relay retro` —
  click to expand the full source chunk (traceability).
- **Observation box + "Regenerate with note"** — live mode only; gated in sample
  mode (steering runs the real pipeline by design — viewing is free, changing is
  live).
- **Refresh data** — live only; `?` tooltip warns it pulls fresh data and replaces
  edits. Destructive actions (refresh/regenerate after editing) warn before
  discarding edits.
- **Data window dropdown** — Sprint 12 selectable; Sprints 11/10 shown as
  "(no data yet)".
- **Approve & save** → reveals a Confluence-style "published page" preview of the
  final edited brief (demo-safe; real delivery not yet wired).
- **Version history left rail** (session-scoped) — each generate / steer / refresh /
  approve creates a timestamped, expandable entry showing why it exists. Entry
  shape is deliberately one Google-Sheet-row wide, for easy future persistence.

---

## Key decisions & rationale (so they're not re-litigated)
- **Voyage over OpenAI embeddings:** Anthropic's recommended partner; generous
  free tier; keeps the stack in the Claude ecosystem.
- **New Supabase key format** (`sb_secret_…`, not JWT): must be sent on the
  `apikey` header ONLY, never `Authorization: Bearer` (that triggers JWT parsing
  and fails). In n8n, set node Authentication to "None" and add `apikey` +
  `Content-Type` as manual headers — do NOT attach a Header-Auth credential (it
  injects an unwanted Authorization header).
- **Supabase "auto-expose new tables" OFF + RLS on, no public policy:** backend-only
  table; service role bypasses RLS. Had to explicitly
  `grant … to service_role` after disabling auto-expose.
- **Proxy pattern (Render):** hides n8n URL, CORS-locks to ancatrif.com,
  rate-limits (default 5/IP/hour), validates input. Frontend never sees n8n.
- **Sample-by-default + "Try it live" toggle:** instant, costless, abuse-proof
  default; live pipeline only on opt-in. Sidesteps Render free-tier cold start
  (~30–50s wake) for casual visitors.
- **Approve = the publish gate:** the human review is what turns a machine draft
  into a published artifact ("attribute the output"). Published version is the
  facilitator's *edited* brief, not raw model output.
- **Storage philosophy (minimum-necessary):** version history / run logs →
  Google Sheets (append-only, human-readable, fits the volume). Test fixtures /
  golden set / prompt → version-controlled files in the repo (version with the
  code). Don't provision a database for this volume.

---

## Known issues / tech debt
- **Prompt drift:** the live prompt lives inside the n8n "Build Claude Prompt"
  node (has observation handling + JSON output). The original `brief-agent.js`
  has an OLDER prose-format prompt. n8n node is the source of truth, but it's NOT
  version-controlled — should be extracted to a repo file (`prompts/…`).
- **n8n `$input.item` vs `$input.all()`:** the "Extract Embedding" and "Tag Chunks
  With Query" nodes MUST use `$input.all()` (loop over all 5 queries), not
  `$input.item` (first only) — earlier bug that collapsed 30 chunks → 1. Fixed
  live; export/back up the working workflow JSON so the fix isn't lost.
- **`Math.floor(i / 6)` in Tag Chunks** assumes match_count = 6. If match_count
  changes, that mapping must change too.
- **Retrieval recall:** top-6-per-query means a chunk ranking 7th everywhere could
  be missed. Non-issue at 40 chunks; at scale, raise match_count or add re-ranking
  (Voyage `rerank-2`: retrieve wide ~20, re-rank narrow to ~6).
- **n8n error handling:** malformed Claude JSON could fail the workflow
  ungracefully — add a fallback in "Parse & Attach Sources."

---

## Roadmap (priority order)
1. **Evals** — golden set (JSON in repo) testing retrieval (right chunks surface)
   + synthesis (correct lead item, real citations only, observation steering works).
   Runs on-demand when the *system changes* (prompt/chunking/model/data) — a
   pre-deploy regression gate, NOT a cadence or per-call thing. Highest portfolio
   value (most projects can't measure themselves; on-brand given HR-bot evals).
2. **Approval → email/Confluence delivery loop** — second n8n flow that publishes
   the approved brief. Closes the narrative.
3. **Minimal error handling** in the n8n parse step (graceful failure on bad JSON).
4. **Lightweight run log** — reuse the Google Sheets append for basic observability.
5. **More sprint data** — ingest Sprint 10/11 to light up the dropdown + show
   cross-sprint patterns. (Adding data is straightforward: same fixture schema →
   ingest; pipeline unchanged.)
6. **Proxy unit tests** — turn the manual checks (bad origin→403, bad sprint→400,
   rate limit→429, observation truncation) into an automated file. Deterministic,
   security-relevant, cheap.

### Deliberate NON-goals (scoped out on minimum-necessary grounds — defensible in interview)
- **User login / auth:** single-facilitator workspace; approval already gated by
  the review action. Would only be needed when approval becomes a *permissioned*
  action requiring *named attribution* in the audit trail — then use Supabase Auth
  or Clerk, not roll-your-own.
- Real database for versions; CI/CD; containerization; autoscaling; leaving free
  tiers. All over-engineering for the actual requirement.

---

## How to talk about it (honest framing)
"A working end-to-end RAG pipeline with a hardened proxy, deployed and live, using
realistic fixture data in place of live source integrations. Retrieval, synthesis,
and the human-in-the-loop review/approve flow are fully functional." Lead with the
human-in-the-loop and provenance story (every claim cited; the human approves before
anything publishes; the published artifact is the edited version). The
minimum-necessary-governance reasoning — including the deliberate non-goals — is
itself the senior signal.

> Reliability note for demos: the live pipeline depends on Voyage, Supabase, n8n,
> Render, and Anthropic all being up and within free tiers. Record a Loom of a clean
> live run (wake the Render service first to avoid cold-start lag on camera) as the
> durable primary demo; keep the live tool available as "and yes, it really runs."
