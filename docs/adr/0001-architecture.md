# 0001 — Architecture: modular monolith + Temporal + Postgres RLS; the LLM is not the source of truth

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
KaamOS Lite is a solo-built, ~6-week, near-zero-cost demonstration of a policy-governed procurement agent
for Pakistani construction contractors. It must show senior-level engineering (multi-tenancy, durable
workflows, bounded AI, auditability) while staying small enough for one person to run on one VM.

## Decision
- **Django + DRF modular monolith.** Apps (`tenancy`, `catalog`, `procurement`, `policy`, `approvals`,
  `tools`, `audit`, `channels`) plus an in-process `ai/` module. No service split.
- **Temporal** orchestrates the one core workflow (`ProcurementWorkflow`) — durable timers, signals for
  approvals, retries, replay.
- **Postgres 16** is the single datastore: relational data, **Row-Level Security** for tenant isolation,
  **pgvector** for embeddings, `pg_trgm` + full-text for fuzzy/BM25 search.
- **The LLM is never the source of truth.** It turns unstructured input into typed, validated objects and
  answers with citations. Deterministic code does math, ranking, policy and record creation; a human
  approves anything over a policy threshold; every action is audited.
- **Single-node Docker Compose** deployment.

## Consequences
- One codebase, one deploy, one DB to back up; RLS and pgvector share the same isolation story.
- Tenant isolation is enforced by the database, not just ORM filters (see 0003).
- AI quality problems degrade to a human queue, never to a wrong PO.
- Scaling past one node is explicitly out of scope; acceptable for a portfolio/MVP.

## Alternatives considered
- **Microservices** — operational overhead with no benefit at this size.
- **Free-running agent loop (LLM picks the next step)** — impressive demo, but not auditable or safe for
  money; rejected (see 0002).
- **Separate vector DB (Qdrant/Pinecone)** — a second isolation boundary to secure; pgvector under RLS is enough.
- **Celery-only orchestration** — no durable waits on human approval or replay (see 0008).
