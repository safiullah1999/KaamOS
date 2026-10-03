# 0003 — RLS mechanics: `SET LOCAL app.org_id` per transaction, `tenant_context()` for workers, two DB roles

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
Tenant isolation is a headline feature. ORM `filter(org=…)` alone fails silently when a filter is missed.
Postgres RLS is the backstop, but setting the tenant context is easy to get subtly wrong with pooled or
persistent connections.

## Decision
- **Tenant key:** `org_id` everywhere (FK field `org` → column `org_id`; session var `app.org_id`). Never
  `tenant_id` in code.
- RLS enabled on **every tenant table** plus `Document`/`DocumentChunk`, with policy
  `USING (org_id = current_setting('app.org_id', true)::uuid)` (and matching `WITH CHECK`).
  `missing_ok = true` → **no context means NULL → zero rows**, not an error.
- Context is set with **`SET LOCAL` inside a transaction**, never plain `SET`:
  - **Requests:** `ATOMIC_REQUESTS = True`; middleware runs `SET LOCAL app.org_id = %s` once per request.
  - **Workers** (Celery tasks, Temporal activities): `tenant_context(org_id)` opens `transaction.atomic()`
    and runs the same `SET LOCAL`.
- **Two DB roles** on the app database: the app role is `NOSUPERUSER NOBYPASSRLS`; a separate privileged
  role owns tables and runs migrations.
- **Global tables (no org-RLS):** `Organization`, `User`, `InboundRoute`, `SystemAuditEvent`.
- **Required test (Phase 1 DoD):** tenant A cannot read or write tenant B rows via the ORM **and** via a
  raw pgvector similarity query on `DocumentChunk`; a query with no context returns zero rows.

## Consequences
- A missed ORM filter returns nothing rather than another tenant's data.
- Context cannot leak between requests on a reused connection (it resets at commit/rollback).
- Long-running work must happen inside `tenant_context()`; anything outside it sees zero rows.
- The test suite needs a connection as the app role (not the migration role) to prove isolation.

## Alternatives considered
- **ORM-only scoping** — one missed filter is a cross-tenant leak.
- **Schema-per-tenant / DB-per-tenant** — heavier migrations and ops; weaker CV story for RLS.
- **Plain `SET` per connection** — leaks across requests with pooling.
