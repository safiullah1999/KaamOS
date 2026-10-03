# 0006 — Policy is the single source of truth for approvals

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
Approval thresholds appeared in two places (tool metadata and `PolicyRule`). A "flag" outcome existed in a
seed rule but not in the `Decision` type. The execution guard required a prior `allow`, but nothing said how
an approved action became allowed, or who may approve.

## Decision
- **Tools declare** `side_effect`, `required_permission`, `idempotency`, `reversible` — **no thresholds**.
  All thresholds live in per-org `PolicyRule`s (canonical seed list: `docs/BACKEND.md` §5).
- **`Decision = {result: allow | require_approval | deny, role?, reasons: list[str]}`.** "Flag" is a reason
  attached to `require_approval`, not a separate result. Every evaluation writes a decision-log row.
- **Execution guard:** any `external`/`financial` tool requires a prior `allow` for that action.
- **Approval → allow:** approving does not bypass the guard. On resume, the workflow **re-evaluates policy
  with the approval present**; the rule that required approval now yields `allow`, then the tool runs.
- **Token identifies, session authorizes.** The signed link identifies the approval; deciding requires a
  logged-in session with `org_id == token.org_id` and `role == requested_role`. `decided_by` records the
  user; deciding a non-pending approval returns 409. Expiry = workflow wait timeout → auto-reject.
- **`system` actor** (the workflow) has its own `ROLE_PERMISSIONS` entry and **can never satisfy
  `require_approval`**.

## Consequences
- One place to change a limit; tool code never encodes business thresholds.
- A forwarded approval link alone cannot approve a PO.
- The audit trail shows both the original `require_approval` and the later `allow` with the approval id.

## Alternatives considered
- **Thresholds on tools** — two sources of truth that drift.
- **Approval directly flips the guard** — skips re-checking state that may have changed while waiting.
- **Token-only approval** — anyone holding the link could approve.
- **OPA/Rego or Cedar now** — deferred; JSON rules + tests are enough for MVP.
