# KaamOS Lite — Implementation Plan

Senior-grade portfolio project (doubles as a startup MVP), solo, ~6 weeks @ ~12–15 hrs/wk, near-zero cost.
Read `CLAUDE.md` first — especially **§2b resolved design decisions**. Progress lives here: tick `- [ ]`.

**Invariant:** the LLM never decides. The core workflow is deterministic (Temporal); the LLM extracts
typed objects, answers with citations, and runs a bounded read-only copilot. Everything is audited.

---

## How to start

### Day 0 — no code
- [ ] Talk to 2–3 contractors. Collect ONE anonymized procurement WhatsApp thread + ONE handwritten quote
      photo. (Makes the demo real; starts the optional startup track.) *If none: the golden set falls back
      to fully synthetic — see Phase 2.*

### Session 1 — reconcile docs (plan mode, no code)
- [x] Claude reads `CLAUDE.md`, this file, and `docs/*`; lists any remaining contradictions/gaps; applies
      agreed edits.
- [x] Put files in the layout `CLAUDE.md` §6 expects (root `CLAUDE.md` + `IMPLEMENTATION_PLAN.md`, `docs/*.md`,
      `.claude/commands/next-task.md`, import stubs in `backend/`, `backend/ai/`, `frontend/`); `git init`.
      Commit `docs: initial project context`.

### Session 2 — ADRs (turn decisions into durable context)
- [ ] `0001-architecture` (modular monolith + Temporal + RLS; LLM not source of truth)
- [ ] `0002-llm-autonomy-boundary` (deterministic workflow; copilot = read-only tools only; no `plan()`)
- [ ] `0003-rls-mechanics` (`SET LOCAL app.org_id` in a txn; `tenant_context()` for workers; two DB roles)
- [ ] `0004-auth` (cookie session + CSRF; browser → Next route handlers → DRF; `Org`/`User`/`InboundRoute` global;
      the Meta webhook is the one direct, signature-verified exception)
- [ ] `0005-org-resolution` (`InboundRoute` for WhatsApp; simulator org from session, never the body; signed
      token embeds `org_id` for approvals; suppliers per-org)
- [ ] `0006-policy-single-source` (policy owns thresholds; `Decision` = allow/require_approval/deny + `reasons[]`;
      approval → re-eval → allow; token identifies / session authorizes; `system` actor can't approve)
- [ ] `0007-data-shape` (`Decimal(12,2)`; line items end to end; single-supplier award; `Document`/`DocumentChunk`+`page`;
      role enum + code perm-map)
- [ ] `0008-temporal-vs-celery` (Temporal for the workflow on its own DB; Celery from P2 for OCR/voice, then benchmark/embeddings)
- [ ] `0009-eval-strategy` (recorded cassettes in CI, miss = fail, re-record in same PR; metrics ramp; classifier data held out)
- [ ] `0010-demo-llm-and-host` (local Ollama for the public demo; free tier = dev-synthetic only; Oracle ARM 24 GB host;
      cost figures from prod prices × recorded tokens)
- [ ] Commit `docs: foundational ADRs`.

### Sessions 3+ — build, one task per session, via `/next-task`
Plan mode → approve → edit → `make test`+`make lint` (paste) → tick box → commit → `/clear`.

---

## Phase overview

| Phase | Name | Week | Goal |
|---|---|---|---|
| **0** | Foundations & scaffolding | 1 | Repo, Compose (app-pg + temporal-pg), CI, ADRs, import files, green CI |
| **1** | Multi-tenant core & domain | 1 | Tenant-isolated model + seed; RLS *proven* (incl. a vector table) |
| **2** | Channel, intake & extraction (incl. OCR) | 2 | Simulator + cascade + OCR + minimal gateway + eval harness |
| **3** | Governance | 3 | Tool registry + policy engine (+reasons, re-eval) + gateway hardening |
| **4** | Durable workflow + approvals + dashboard | 3–4 | Temporal flow + approvals + owner dashboard/approval UI |
| **5** | RAG, guardrails & audit | 5 | Cited RAG + copilot + injection defense + per-org hash chain + audit UI |
| **6** | Observability, polish & showcase | 6 | Traces, cost dashboard, weekly report, deploy, README, video |

Buffer: weeks 7–8. **Cut order:** drop RAG (P5a) before policy/approvals/Temporal.
**Expect Phase 1 to be the slow one** (Django RLS, two roles, `SET LOCAL`, isolation test). All of Week 1 is normal.

---

## Phase 0 — Foundations & scaffolding
- [ ] `docker-compose.yml`: **app-postgres** (pgvector), **temporal-postgres** (separate DB/user), redis,
      temporal, temporal-ui, backend, **temporal-worker**, frontend. (celery-worker added in P2.)
- [ ] `Makefile`: up/down/migrate/seed/evals/evals-record/demo/test/lint.
- [ ] Django project (settings split, **`ATOMIC_REQUESTS=True`**), DRF, empty apps, `.env.example`.
- [ ] Next.js placeholder.
- [ ] Import files (`backend/CLAUDE.md`, `backend/ai/CLAUDE.md`, `frontend/CLAUDE.md`), empty `docs/NEXT.md`,
      `.claude/commands/next-task.md`.
- [ ] CI: lint + type-check + pytest. (Eval gate added in P2.)

**DoD:** `make up` runs everything; `make test` green in CI; Temporal UI reachable. (From here on, the
`make test`/`make lint` DoD bar applies.)

---

## Phase 1 — Multi-tenant core & domain
- [ ] **Global (no RLS):** `Organization`, `User(+phone, role enum)`, `InboundRoute`. Static `ROLE_PERMISSIONS`
      map in code. Auth looks up `User` by email before any org context.
- [ ] **Tenant (org FK, RLS):** `Site`, `Supplier`, `Sku`, `SkuAlias`, `MaterialRequest`+`RequestLine`,
      `RFQ`+`RFQInvite`, `Quote`+`QuoteLine`, `PurchaseOrder`+`POLine`, `GRN`+`GRNLine`, `Invoice`,
      `QuoteBenchmark`, `PolicyRule`, `Approval` (shapes per BACKEND §2).
- [ ] **Empty + RLS on:** `Document`, `DocumentChunk`(`page`, `embedding vector(1024)`) — filled in P5.
- [ ] **`AuditEvent` (all columns) + service-layer emission** (no hash chain yet); `trace_id` = app
      correlation id (UUID per inbound/workflow). `SystemAuditEvent` (global).
- [ ] **RLS:** enable on all tenant tables + `Document*`; policy `current_setting('app.org_id', true)`;
      `SET LOCAL` via middleware (requests) + `tenant_context()` (workers); app role `NOBYPASSRLS` + migration role.
- [ ] **Seed:** 2 tenants, 6 sites, 12 suppliers (8 approved, per-org), ~40 SKUs+aliases, a simulator
      `InboundRoute` per tenant, and **full synthetic MaterialRequest→RFQ→Quote→PO→GRN chains for 90 days**
      (`origin="seed"`, with some late GRNs) → compute `QuoteBenchmark` (30-day median) and
      `reliability_score`. Money = `Decimal(12,2)`.

**DoD:** RLS isolation test — tenant A can't read/write tenant B via **ORM and via a raw pgvector query on
`DocumentChunk`**; no context → zero rows. Seed loads clean.

---

## Phase 2 — Channel, intake & extraction (incl. OCR)
- [ ] `channels/`: `ChannelAdapter` + **`POST /channels/simulator`** (session-authenticated; org from the
      session, never the body) + a Meta Cloud API adapter behind a flag (**`/channels/whatsapp/webhook`**,
      `X-Hub-Signature-256` verified, org via `InboundRoute`); **persona responder** (scripted auto-replies:
      fast/slow/overpriced/handwritten) with a manual toggle. Test: a body-supplied org id is ignored.
- [ ] **`celery-worker`** added to Compose; media tasks (OCR, voice) run there, never in a request.
- [ ] **Minimal model gateway** (`ai/gateway/`): LiteLLM call + per-task config + **PII redaction**. Demo
      mode = **Ollama qwen2.5:7b**.
- [ ] **Extraction cascade:** rules + alias dict + fuzzy match → classifier → gateway LLM (Pydantic-validated,
      **multi-line `items[]`**) → human queue.
- [ ] **OCR / multimodal (here, not P5):** PaddleOCR/Tesseract + budget-VLM fallback for **quote photos and
      invoice photos** → `extract_quote` / `extract_invoice`; whisper.cpp for voice notes. Low confidence →
      human queue ("Needs review"), non-blocking.
- [ ] **Frontend:** login page, simulator UI + a minimal request view with `NeedsReview` (intake visible end-to-end).
- [ ] **Golden set v1** (~200: RU/Urdu/English, typos, transcripts, **quote photos**): synthetic, seeded from
      the Day-0 thread (fallback = fully synthetic); founder-corrected. **Classifier training data held out**
      from the eval set. **Eval harness + CI gate** with **recorded LLM cassettes** (no live key).
- [ ] **Eval metrics (P2 set):** extraction F1, SKU-match, intent, % resolved without LLM, cost/msg.

**DoD:** metrics reported in CI; a regressing PR fails the gate; the simulator creates a real multi-line
`MaterialRequest`; a handwritten quote photo extracts into `QuoteLine`s.

---

## Phase 3 — Governance
- [ ] **Tool registry** (`apps/tools/`): the **canonical tool list in BACKEND §4**; typed fns + metadata
      (`side_effect`, `required_permission`, `idempotency`, `reversible` — **no approval threshold**);
      generates the copilot tool schema; execution guard (no `external`/`financial` without a prior `allow`);
      `system` actor permissions.
- [ ] **Policy engine** (`apps/policy/`): the **5 canonical seed rules in BACKEND §5** (benchmark rule
      evaluated per line); `Decision` = allow/require_approval/deny + `reasons[]`; decision log;
      **re-evaluation with an approval present → allow**.
- [ ] **Ranking** (`rank_quotes`, pure): complete quotes only, meets-`needed_by` first, delivered cost,
      tie-breaks, `needed_by_missed` flag (BACKEND §8). Unit-tested incl. partial quotes + `lead_time_days=None`.
- [ ] **Gateway hardening:** per-org token+PKR budgets, prompt + exact/semantic caching, per-org cost accounting.
- [ ] **Eval metrics (add):** tool-call correctness + **policy-violation rate (must be 0)**.

**DoD:** every tool + every policy rule has a test; a decision-log row per evaluation; an `external`/`financial`
tool cannot run without a prior `allow` (tested).

---

## Phase 4 — Durable workflow + approvals + dashboard
- [ ] **Approvals app:** `Approval` (subject, diff, `reasons`, expiry, `workflow_id`); **signed-token** links
      (token embeds `org_id`+`approval_id`+expiry); `POST /approvals/{token}/decide` verifies token **+ requires a
      logged-in session of the required role in the same org** → sets context → records `decided_by` →
      **signals Temporal by `workflow_id`**; single-use (409 if not pending); expiry auto-rejects.
      Tests: wrong role / wrong org / logged-out / replayed decide are all refused.
- [ ] **Temporal workflow:** validate → approved suppliers → **fan out RFQs** (`RFQInvite` + code) →
      `await quote_received signals OR deadline_at (4h)` → record quotes → **rank** → policy → (await approval →
      **re-check → allow**) → PO (+lines, `expected_delivery_at`) → GRN → **per-line three-way match
      (±2% qty / ±1% amount)** → close. **No complete quotes → `stalled` + notify owner**. Idempotent activities;
      `tenant_context()`; deterministic workflow code.
- [ ] **Quote correlation:** inbound supplier reply → `RFQInvite` via reply context → code → single open
      invite → else human queue; then signal the workflow (BACKEND §8). Tested with a supplier holding 2 open RFQs.
- [ ] **Benchmark + reliability recompute:** nightly Celery beat task (also a management command).
- [ ] **Frontend:** owner dashboard (request list/detail, `QuoteComparisonTable` incl. partial quotes, `PolicyFlags`)
      + approval page (login-gated decide).

**DoD:** kill the worker mid-run → it resumes + completes; replay test passes; the full cement flow runs
end-to-end on seed data, pausing on the approval signal; approving via the link resumes it.

---

## Phase 5 — RAG, guardrails & audit
- [ ] **RAG:** ingest `Document`s → OCR (reuse P2) → chunk → bge-m3 → `DocumentChunk` → hybrid BM25+vector →
      bge-reranker → answer with **citations (`title`+`page`)**; **refuse** when unsupported.
- [ ] **Copilot (`ai.assist`):** read-only (`side_effect="none"`) tool calls through the registry + guardrails.
- [ ] **Guardrails:** schema validation; allow-list; **instruction/data separation**; bounds; bank-account
      hard stop; **both injection tests** (extraction path + copilot path, AI §8) blocked + logged.
- [ ] **Audit hardening:** **per-org hash chain** (ordered by `seq`) + `verify` command (per-org) + tamper test.
- [ ] **Frontend:** audit timeline UI + `AskPanel` in the dashboard (copilot + cited RAG).

**DoD:** RAG cites + refuses; copilot can't call a write tool; both injection tests blocked; per-org chain
verifies + tamper detected.

---

## Phase 6 — Observability, polish & showcase
- [ ] OpenTelemetry traces across every step (trace_id now the OTel id); Langfuse; Sentry. (§10 "traces for
      every feature" bar takes effect now.)
- [ ] **Cost dashboard** (cost/workflow, % per cascade tier, tokens by task) + **weekly report** (`/reports/weekly`).
- [ ] Deploy (Oracle Always Free ARM 24 GB; fallback Hetzner CX43 16 GB); guest owner login (Ollama);
      `make demo` nightly reset. Cost figures = prod-model prices × recorded token counts.
- [ ] README (pitch + GIF + 5-min run), architecture + sequence diagrams, ADRs complete.
- [ ] **3-minute video** (script below) + 2 deep-dive posts.

**DoD:** anyone can `docker compose up && make seed` and run the demo; video + posts live; README links both.

---

## The 3-minute demo video
0:00 problem (messy WhatsApp) · 0:20 live flow (simulator | dashboard; multi-item request → RFQs → 3 replies
incl. handwritten → ranked table w/ citations) · 1:20 red policy flag → one-tap approval → audit entry →
Temporal UI paused on the signal · 1:50 kill the worker → resumes; block a supplier injection · 2:20
architecture + eval dashboard (F1, 0 policy violations, ~$0.002/workflow, % no-LLM) · 2:50 "open source, $0 VM."

## Showcase checklist
- [ ] Public repo (README + diagrams + ADRs) · [ ] live seeded demo · [ ] 3-min video · [ ] 2 posts ·
      [ ] 6-post LinkedIn series · [ ] sent to ~20 CTOs/hiring managers · [ ] (optional) 3 contractor pilots.

## Milestone gate
A stranger can, in 5 minutes: clone → `make up && make seed && make demo`, run the cement flow, watch it
survive a worker crash, see a blocked injection, and read the cost dashboard. Ship in Week 6 regardless of polish.
