# Architecture Decision Records

Short records of decisions that shape the system. Format: Status · Date · Context · Decision ·
Consequences · Alternatives considered. To change a decision, add a new ADR that supersedes the old one
(and mark the old one `Superseded by NNNN`) — don't rewrite history.

| # | Decision | Status |
|---|---|---|
| [0001](0001-architecture.md) | Modular monolith + Temporal + Postgres RLS; LLM is not the source of truth | Accepted |
| [0002](0002-llm-autonomy-boundary.md) | Deterministic workflow; read-only copilot; no `plan()` | Accepted |
| [0003](0003-rls-mechanics.md) | `SET LOCAL app.org_id` per txn; `tenant_context()`; two DB roles | Accepted |
| [0004](0004-auth.md) | Cookie session + CSRF via Next route handlers; webhook exception | Accepted |
| [0005](0005-org-resolution.md) | `InboundRoute`, session-scoped simulator, signed approval tokens | Accepted |
| [0006](0006-policy-single-source.md) | Policy owns thresholds; re-eval → allow; token identifies, session authorizes | Accepted |
| [0007](0007-data-shape.md) | `Decimal(12,2)`; line items end to end; documents; role enum | Accepted |
| [0008](0008-temporal-vs-celery.md) | Temporal for the workflow (own DB); Celery for media/batch jobs | Accepted |
| [0009](0009-eval-strategy.md) | Golden set; phased metrics; recorded cassettes in CI | Accepted |
| [0010](0010-demo-llm-and-host.md) | Local Ollama demo on Oracle ARM 24 GB; cost from prod prices | Accepted |
