# 0007 — Data shape: money, line items, documents, roles

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
Several schema questions had to be settled before Phase 1: how to store money, whether a request can hold
several items, how RAG chunks cite pages, and whether roles/permissions are tables.

## Decision
- **Money:** `Decimal(12,2)` PKR everywhere; never float.
- **Line items end to end:** `RequestLine → QuoteLine → POLine → GRNLine`. `Invoice` stays total-level and
  is matched against Σ(delivered qty × PO unit price) + delivery charge.
- **Single-supplier award:** one supplier wins a whole request. A quote missing any line is stored
  (`is_complete = false`) and shown, but **excluded from ranking**.
- **RFQ correlation:** `RFQInvite(rfq, supplier, code, outbound_msg_id)` — one per supplier per RFQ, with a
  short code included in the RFQ template.
- **Delivery timeliness:** `PurchaseOrder.expected_delivery_at` (PO date + quoted lead time) drives
  `reliability_score` = on-time GRNs ÷ total GRNs (default 0.80).
- **Seed vs live:** `MaterialRequest.origin ∈ {live, seed}`; `Quote.source` only records the extraction
  source (`text | image_ocr | image_vlm`).
- **Documents:** `Document(org, kind, title, file_ref)` + `DocumentChunk(org, document, page, chunk_index,
  content, embedding vector(1024))`, both created empty with RLS in Phase 1.
- **Roles:** enum on `User` + static `ROLE_PERMISSIONS` in code; no `Role`/`Permission` tables.
- **Audit:** all `AuditEvent` columns exist from Phase 1 (`seq`, `model_version`, `prompt_version`,
  `trace_id`, `prev_hash`, `hash`); values populate progressively; one hash chain **per org** from Phase 5.
- **Keys/time:** UUID PKs; timezone-aware datetimes.

## Consequences
- Realistic multi-item messages ("cement aur sariya") work without schema changes later.
- Three-way match and benchmark checks are per line, so flags point at the exact offending SKU.
- Split awards across suppliers are not supported (a `docs/NEXT.md` candidate).
- Adding a role needs a code change — acceptable for four fixed roles.

## Alternatives considered
- **Integer paisa** — safe but noisier everywhere money is displayed; `Decimal` with Django is sufficient.
- **One SKU per request** — breaks on the most common real message shape.
- **Role/Permission tables** — admin UI and migrations for no MVP benefit.
