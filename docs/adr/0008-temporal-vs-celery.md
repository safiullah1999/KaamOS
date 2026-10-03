# 0008 — Temporal for the workflow, Celery for background jobs

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
The procurement flow waits hours for supplier quotes and for a human approval, must survive worker
crashes, and must be replayable. Separately, OCR, voice transcription, benchmark recompute, and embeddings
are slow, stateless jobs that must not run inside an HTTP request.

## Decision
- **Temporal** (self-hosted OSS) runs `ProcurementWorkflow`: durable timers (RFQ deadline, approval
  expiry), **signals** (`quote_received`, `invoice_received`, approval), retries, replay. Workflow code is
  deterministic (no I/O, no wall-clock); activities are idempotent and use `tenant_context()`.
- Temporal uses **its own Postgres database and user** (`temporal-postgres`), separate from the app DB and
  its RLS roles.
- **Celery + Redis** (`celery-worker`) for non-durable jobs:
  - **Phase 2+:** OCR (PaddleOCR/Tesseract/VLM fallback) and voice transcription (whisper.cpp).
  - **Phase 4+:** nightly benchmark + `reliability_score` recompute (Celery beat; also a management command).
  - **Phase 5+:** RAG ingestion and embeddings.
- Temporal sits behind a `WorkflowEngine` interface so a fallback is possible, but Temporal is the default.

## Consequences
- Crash-resume and replay come for free and become a demo moment.
- Two worker processes to run and monitor (`temporal-worker`, `celery-worker`).
- Media results reach the workflow via the inbound handler signalling it, not by Celery calling Temporal
  internals.

## Alternatives considered
- **Celery-only with a DB state machine** — durable waits and replay become hand-rolled.
- **Temporal for everything** — overkill for fire-and-forget media jobs.
- **Sharing the app Postgres with Temporal** — tangles Temporal's schema with RLS roles and migrations.
