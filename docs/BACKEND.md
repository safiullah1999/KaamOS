# Backend Subsystem — Context (`backend/`)

Read `CLAUDE.md` first (esp. §2b resolved decisions). This governs Django/DRF, the domain model,
workflows, and governance. **Backend is the source of truth.**

**Tenant key:** `org_id` everywhere (FK field `org`). **Money:** `Decimal(12,2)` PKR.

---

## 1. App structure (modular monolith)

```
apps/
├── tenancy/       # Organization, User(+phone), InboundRoute; ROLE_PERMISSIONS map; RLS mw; tenant_context()
├── catalog/       # Sku, SkuAlias, Supplier, Site
├── procurement/   # MaterialRequest+RequestLine, RFQ+RFQInvite, Quote+QuoteLine, PurchaseOrder+POLine,
│                  # GRN+GRNLine, Invoice, QuoteBenchmark, ranking + three-way match (pure functions)
├── policy/        # PolicyRule, PolicyEngine (re-evaluates with approvals), decision log
├── approvals/     # Approval, signed-token links, Temporal signal delivery
├── tools/         # typed tool registry + execution guard
├── audit/         # AuditEvent (append-only; per-org hash chain from Phase 5) + SystemAuditEvent
└── channels/      # ChannelAdapter, simulator endpoint, whatsapp webhook, persona responder (demo),
                   # media Celery tasks (OCR / voice transcription)
ai/                # see docs/AI.md
workflows/         # Temporal workflow + activities
```

Monolith only. Split a service only when deploy cadence diverges (not in this project).

---

## 2. Data model

**Global tables (NOT under org-RLS — needed before an org context exists):**
```
Organization(id, name, ...)
User(id, org, email, phone, role, password_hash)      role ∈ {owner, manager, engineer, viewer}  (enum)
InboundRoute(id, org, channel, business_number_id)    # maps a received WhatsApp business number -> org
```
Roles map to permissions via a **static `ROLE_PERMISSIONS` dict in code** (e.g. `procurement.send_rfq`).
No `Role`/`Permission` tables.

**Tenant tables (org FK -> col `org_id`, RLS on):**
```
Site(id, org, name, area)
Supplier(id, org, name, phone, is_approved, reliability_score, bank_account_verified)   # per-org, not shared
Sku(id, org, family, canonical_name, unit)            family ∈ {cement, steel, bricks, sand_aggregate}
SkuAlias(id, org, sku, alias_text, lang)

MaterialRequest(id, org, site, needed_by, status, source_message, created_by, origin)   # origin: live|seed
RequestLine(id, org, request, sku, quantity, unit)                                        # a request has many lines
RFQ(id, org, request, status, deadline_at, workflow_id)
RFQInvite(id, org, rfq, supplier, code, outbound_msg_id, sent_at)    # code = short ref in the RFQ template, e.g. "Q7K2"
Quote(id, org, rfq, supplier, delivery_charge, lead_time_days, source, raw_ref, is_complete)
                                                      # source: text|image_ocr|image_vlm (extraction source only)
QuoteLine(id, org, quote, request_line, unit_price)
PurchaseOrder(id, org, rfq, supplier, total_amount, expected_delivery_at, status, pdf_ref, approved_by)
POLine(id, org, po, request_line, sku, quantity, unit_price, line_total)
GRN(id, org, po, photo_ref, received_at)
GRNLine(id, org, grn, po_line, delivered_qty)
Invoice(id, org, po, amount, photo_ref, extracted_amount)          # total-level; matched vs Σ delivered value
QuoteBenchmark(id, org, sku, window_days, window_start, median_price)                     # window_days = 30

Document(id, org, kind, title, file_ref, uploaded_at)                                      # created empty in P1
DocumentChunk(id, org, document, page, chunk_index, content, embedding vector(1024))       # empty P1, filled P5

PolicyRule(id, org, rule_json, active)
Approval(id, org, subject_type, subject_id, workflow_id, requested_role, status, diff_json, reasons,
         expires_at, decided_by, decided_at)
AuditEvent(id, org, seq, actor_type, actor_id, action, before, after, policy_decision,
           model_version, prompt_version, trace_id, prev_hash, hash)
SystemAuditEvent(id, actor, action, detail, at)        # global, non-chained: login, failed login, etc.
```
Money fields are `Decimal(12,2)`. **AuditEvent columns all exist from Phase 1**; values populate
progressively: `model_version`/`prompt_version` from P2 (when an LLM is used), `trace_id` = app
correlation id (a UUID per inbound/workflow run) from P1 → OTel trace id from P6, `prev_hash`/`hash`
NULL until P5. `seq` is a per-org monotonic counter used for chain ordering.

---

## 3. Auth & RLS

### Auth
- **Cookie session + CSRF.** The browser talks only to **Next.js route handlers (same origin)**, which
  forward the session cookie to DRF — no cross-origin DRF calls, no CORS.
- Endpoints: `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`, CSRF via Django's cookie.
- Login reads the **global** `User` table by email (no org context), verifies the password, resolves
  `user.org_id`, then establishes org context for the rest of the request.

### Org resolution for context-less entrypoints (important)
- `POST /channels/whatsapp/webhook` (real WhatsApp): called **directly by Meta, not via Next**. CSRF-exempt;
  instead **verify `X-Hub-Signature-256`** (app secret) and reject unsigned/invalid requests. The payload
  carries the **receiving business number id** → global `InboundRoute` → `org_id` → set context → resolve
  the sender *within* that org (engineer by `User.phone`, supplier by `Supplier.phone`).
- `POST /channels/simulator`: **session-authenticated** (via Next route handler, CSRF on). `org_id` comes
  from the **logged-in user's session** — the body never carries an org. The body carries the simulated
  sender (`engineer` or a supplier id/phone), resolved within that org.
- `GET/POST /approvals/{token}`: the token is a **signed token (e.g. itsdangerous/JWT) embedding
  `org_id` + `approval_id` + expiry**. Verify the signature → extract `org_id` → set context → load the
  `Approval` under RLS. The DB lookup never happens before the org is known. **Deciding also requires a
  logged-in session** whose `org_id` matches the token and whose role is `requested_role` (see §6).

### RLS mechanics
- Enable RLS on every tenant table + `Document`/`DocumentChunk`:
  `USING (org_id = current_setting('app.org_id', true)::uuid)` (missing context → NULL → zero rows).
- Set context with **`SET LOCAL app.org_id` inside a transaction** — never a plain `SET` on a pooled
  connection. `ATOMIC_REQUESTS=True` wraps each request; middleware runs the `SET LOCAL` once.
- Workers (Celery/Temporal activities) use a `tenant_context(org_id)` context manager (opens a txn + SET LOCAL).
- **Two DB roles** on the app DB: app role `NOSUPERUSER NOBYPASSRLS`; a privileged role runs migrations.
- **Required test (P1):** tenant A cannot read/write tenant B via ORM **and** via a raw pgvector query on
  `DocumentChunk`; no context → zero rows.

```python
@contextmanager
def tenant_context(org_id):
    with transaction.atomic(), connection.cursor() as cur:
        cur.execute("SET LOCAL app.org_id = %s", [str(org_id)])
        yield
```

---

## 4. Tool registry (`apps/tools/`)

The only way anything acts on the system. Each tool is a typed fn with metadata — **no approval
threshold here** (policy owns thresholds).

```python
@tool(side_effect="external",            # none | internal | external | financial
      required_permission="procurement.send_rfq",
      idempotency="rfq:{rfq_id}", reversible=False)
def send_rfq(org_id: UUID, rfq_id: UUID, supplier_ids: list[UUID]) -> RfqResult: ...
```
Duties: generate the LLM tool schema (for the copilot); **execution guard** — any `external`/`financial`
tool requires a prior policy `allow`; enforce `required_permission` via `ROLE_PERMISSIONS`; apply
idempotency; emit an `AuditEvent` per execution. Ranking, matching, PO totals are **pure deterministic
functions — never LLM**.

**Canonical tool list (Phase 3 tests every one):**

| Tool | `side_effect` | `required_permission` | Copilot? |
|---|---|---|---|
| `search_approved_suppliers` | none | `procurement.view` | yes |
| `get_request` / `list_requests` | none | `procurement.view` | yes |
| `get_quotes` | none | `procurement.view` | yes |
| `spend_summary` | none | `reports.view` | yes |
| `answer_question` (RAG) | none | `procurement.view` | yes |
| `rank_quotes` (pure) | none | `procurement.view` | yes |
| `record_quote` | internal | `procurement.record_quote` | no |
| `request_approval` | internal | `approvals.request` | no |
| `record_grn` | internal | `procurement.record_grn` | no |
| `match_invoice` (pure + persist result) | internal | `procurement.match` | no |
| `send_rfq` | external | `procurement.send_rfq` | no |
| `send_message` (template only) | external | `channels.send` | no |
| `create_purchase_order` | financial | `procurement.create_po` | no |

**System actor:** the workflow calls tools as `actor_type="system"` with role `system`, which has its own
entry in `ROLE_PERMISSIONS` (everything above **except** approving). `system` can never satisfy a
`require_approval` — only a human session can.

---

## 5. Policy engine (`apps/policy/`)

Deterministic per-tenant rules, run **before** any side-effecting tool.
`Decision = {result: allow | require_approval | deny, role?: str, reasons: list[str]}`.

**Canonical seed rules (the single source — the plan references these, does not restate):**
```json
[
  {"if": {"action": "create_purchase_order", "amount_gt": 500000}, "then": {"require_approval": "owner", "reason": "amount_over_limit"}},
  {"if": {"action": "send_rfq", "supplier_not_approved": true}, "then": {"deny": "supplier_not_approved"}},
  {"if": {"action": "create_purchase_order", "price_over_benchmark_pct_gt": 5}, "then": {"require_approval": "owner", "reason": "price_over_benchmark"}},
  {"if": {"action": "add_supplier_bank_account"}, "then": {"deny": "requires_human_verification"}},
  {"if": {"action": "create_purchase_order", "needed_by_missed": true}, "then": {"require_approval": "owner", "reason": "needed_by_missed"}}
]
```
`price_over_benchmark_pct_gt` is evaluated **per line**: it matches if **any** PO line's `unit_price` is
more than N% above that SKU's 30-day benchmark; each offending line adds a reason
(e.g. `price_over_benchmark:DG-cement:+6.1%`). A SKU with no benchmark yet adds no flag.
`needed_by_missed` is set by `rank_quotes` when the winning quote can't meet `needed_by` (§8).

Rule 4 is defense-in-depth: no `add_supplier_bank_account` tool exists in MVP (the guardrail hard-stop
enforces it); the rule applies if/when such a tool is added. Each evaluation writes a decision-log row.

**Approval → allow:** approving does not bypass the guard. The `ProcurementWorkflow`, on resume, calls the
policy engine **again with the approval present**; the rule that returned `require_approval` now returns
`allow`, and the guarded tool executes.

---

## 6. Approval engine (`apps/approvals/`) — Phase 4

- First-class `Approval` (subject, diff, `reasons`, expiry, `workflow_id`). Delivery: a **signed-token link**
  (token embeds `org_id`+`approval_id`+expiry) + optional WhatsApp buttons.
- **Token identifies, session authorizes.** `GET /approvals/{token}` renders read-only for anyone holding a
  valid token; `POST /approvals/{token}/decide` additionally requires a **logged-in session** with
  `session.org_id == token.org_id` and `role == requested_role` (not logged in → login, then back to the
  link). It verifies the token, sets org context, loads under RLS, records `decided_by`/`decided_at`, and
  **signals the Temporal workflow by `Approval.workflow_id`**. `PurchaseOrder.approved_by` = `decided_by`.
- **Single-use:** deciding a non-`pending` approval returns 409. Tokens aren't stored; status is the guard.
- Expiry: the workflow's `wait_condition` has a timeout = `expires_at`; on timeout it marks the approval
  `expired` (= rejected) and notifies the owner.
- Prefer links over chat ping-pong (cuts WhatsApp cost).

---

## 7. Temporal workflow (`workflows/`) — Phase 4

`ProcurementWorkflow` (deterministic, durable):
1 validate → 2 `search_approved_suppliers` → 3 **fan out `send_rfq`** (one `RFQInvite` + code per supplier)
→ 4 `await quote_received signals OR deadline_at` → 5 `record_quote` → 6 **`rank_quotes`** (deterministic)
→ 7 **policy check** → 8 if `require_approval`: `await approval_signal` (timeout = `expires_at`), then
**re-check policy → allow** → 9 `create_purchase_order` → 10 `record_grn` → 11 **three-way match** → 12 close.

**How quotes reach the workflow:** supplier replies arrive at the channel entrypoint, not the workflow.
The inbound handler (photos/voice first go through the `celery-worker` media task) runs `ai.extract_quote`,
correlates the reply to an `RFQInvite` (§8), and **signals `quote_received(invite_id, QuoteExtract)`** to
`RFQ.workflow_id`. Invoices arrive the same way (`ai.extract_invoice`) and signal `invoice_received`.

Rules: activities idempotent + retry policies; activities set org via `tenant_context()`; workflow code
deterministic (no I/O, no wall-clock). **Crash test (DoD):** kill the worker mid-run → it resumes.
Temporal runs on its **own Postgres database** (separate DB/user from the app DB).

---

## 8. Business rules & defaults (named constants; per-tenant overridable later)

- **Ranking (single-supplier award):** only **complete** quotes (a `QuoteLine` for every `RequestLine`)
  are ranked; partial quotes are recorded and shown but excluded. Among complete quotes, prefer those that
  **meet `needed_by`** (`now + lead_time_days ≤ needed_by`); within that group choose the lowest **total
  delivered cost** (Σ line `unit_price`×`quantity` + `delivery_charge`); tie-break by `lead_time_days`, then
  `reliability_score`. A quote with `lead_time_days = None` is treated as **not** meeting `needed_by`.
  If **no** complete quote meets `needed_by`, pick the cheapest complete quote and set
  `needed_by_missed=true` (→ policy rule 5 → owner approval).
- **No complete quotes** (0 quotes, or only partial ones) → notify owner, request `status="stalled"`, no auto-PO.
- **`reliability_score`** = on-time GRNs ÷ total GRNs, where on-time = `GRN.received_at ≤ PO.expected_delivery_at`
  (`expected_delivery_at` = PO date + quoted `lead_time_days`). Default **0.80** for suppliers with no GRNs.
  Seeded from the synthetic PO→GRN history.
- **RFQ deadline** = **4 hours** (`deadline_at`); on timeout, proceed with the quotes received.
- **Quote ↔ RFQ correlation** (supplier may have several open RFQs), first match wins:
  (1) WhatsApp reply context (`context.id` == `RFQInvite.outbound_msg_id`); (2) an invite `code` found in the
  text/OCR output; (3) the supplier has exactly **one** open invite; else → **human queue** (owner picks).
- **Three-way match (per line):** for each `POLine`, `GRNLine.delivered_qty` within **±2%** (min ±1 unit) of
  ordered qty; invoice `amount` within **±1%** of Σ(delivered_qty × PO unit_price) + delivery charge.
  Outside tolerance → flag for human.
- **Benchmark:** median quoted `unit_price` per `sku` over a **30-day** window (`QuoteBenchmark`), recomputed
  nightly (Celery beat task on `celery-worker` from P4; also runnable as a management command).

---

## 9. Audit log (`apps/audit/`)

- `AuditEvent` **model + service-layer emission from Phase 1**, for every state change (tool execution,
  policy decision, approval). Includes `model_version`+`prompt_version`+`trace_id`.
- **Hash-chaining from Phase 5:** **one chain per org**, ordered by `seq`; `hash = H(prev_hash || canonical(event))`;
  a `verify` command runs per-org (sets context, walks that org's chain); a tamper test must fail verification.
- Pre-auth/system events (login, failed login) go to the global non-chained `SystemAuditEvent`.

---

## 10. Payments (mock only)
`MockPSPAdapter` with a Raast-style `request_to_pay(...)` + webhook that flips invoice status. MVP only
**records/reconciles** (match a Raast/IBFT reference to an invoice); never executes payments.

## 11. API surface (DRF; org-scoped by RLS + permissions)
**Browser-facing (only via Next route handlers, session + CSRF):** `POST /auth/login|logout`, `GET /auth/me`
· `POST /channels/simulator` · `GET /requests[/{id}]` · `GET/POST /approvals/{token}[/decide]` ·
`GET /suppliers|sites|skus` · `GET /reports/weekly` · `GET /metrics/cost` · `POST /assist` (P5).
**Machine-facing (direct, the one exception):** `POST /channels/whatsapp/webhook` (+ its `GET` verify
handshake) — CSRF-exempt, `X-Hub-Signature-256` verified, org via `InboundRoute`.

## 12. Conventions
`Decimal(12,2)` money; tz-aware datetimes; UUID PKs; logic in services (not views/models); pure functions
for ranking/benchmark/match (unit-tested); emit an `AuditEvent` for every state change.

## 13. Do-NOT list
- AI code must not create records, rank, or decide — accept only validated objects from `ai/`.
- No `external`/`financial` tool without a prior policy `allow`.
- No programmatic supplier bank account add/verify.
- No ORM-filter-only isolation — RLS on + tested.
- No plain `SET` for `app.org_id` — `SET LOCAL` in a txn (mw for requests, `tenant_context()` for workers).
- `Organization`/`User`/`InboundRoute` never under org-RLS.
- Never take `org_id` from a request body or query string — only from the session, `InboundRoute`, or a
  verified signed token.
- No OCR/transcription inside an HTTP request — enqueue on `celery-worker`.
- No I/O inside Temporal workflow code (activities only).
