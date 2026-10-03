# 0005 — Org resolution for context-less entrypoints

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
Under RLS, nothing can be read until `app.org_id` is set. Three entrypoints arrive without an org: real
WhatsApp messages, simulator messages, and approval links opened from a phone. Looking up a supplier or
approval to *find* the org would need a cross-tenant read, which RLS forbids.

## Decision
- **Real WhatsApp** — `POST /channels/whatsapp/webhook`: verify `X-Hub-Signature-256`; read the
  **receiving business number id** from the payload → global `InboundRoute(channel, business_number_id → org)`
  → set context → resolve the sender *within* that org (engineer by `User.phone`, supplier by `Supplier.phone`).
- **Simulator** — `POST /channels/simulator`: **session-authenticated**; `org_id` comes from the logged-in
  session. The body carries only the simulated sender + content. **An org id in the body is ignored** (tested).
- **Approval links** — a **signed token** (itsdangerous/JWT) embedding `org_id` + `approval_id` + expiry.
  Verify signature → extract `org_id` → set context → load the `Approval` under RLS. (Who may *decide* is
  covered in 0006.)
- **Suppliers are per-org**, not shared across tenants; the same real-world supplier serving two orgs is
  two `Supplier` rows.
- **Rule:** `org_id` only ever comes from a session, an `InboundRoute`, or a verified signed token — never
  from a request body or query string.

## Consequences
- No DB lookup ever happens before the org is known; RLS stays strict.
- Each org needs its own WhatsApp business number in production (fine for MVP; a shared-number router is a
  `docs/NEXT.md` item if ever needed).
- Seed creates one simulator `InboundRoute` per tenant for parity with the real path.

## Alternatives considered
- **Simulator passes the tenant in the payload** — contradicts "never trust client-supplied org ids".
- **Global supplier directory** — cross-tenant data and a much harder RLS story.
- **Opaque random approval tokens stored in the DB** — requires a pre-context lookup on a tenant table.
