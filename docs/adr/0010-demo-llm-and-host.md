# 0010 — Demo LLM and host: local Ollama on a 24 GB ARM VM

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
The public demo has a guest login, so strangers can type arbitrary — possibly real or sensitive — text.
Free-tier cloud LLMs may train on inputs. The stack (two Postgres instances, Temporal, Redis, Django, Next,
Ollama, bge-m3, bge-reranker, PaddleOCR, whisper.cpp) must fit on one near-free VM.

## Decision
- **Public demo LLM = local Ollama (`qwen2.5:7b`)** behind the same gateway (`demo_mode`).
- **Free-tier cloud APIs = dev/eval on synthetic data only** — never the public demo, never real data.
- **Production = the cheapest paid model that passes the eval gate**, chosen by gateway task config.
- **Host = Oracle Always Free ARM (4 OCPU / 24 GB)**; fallback **Hetzner CX43 (16 GB)**. The 8 GB CX33 is
  too small once Ollama (~5 GB) is loaded alongside everything else.
- **Headline cost figures** (e.g. `$0.002/workflow`) are computed from **prod-model prices × recorded token
  counts**, since the Ollama demo itself costs $0. The cost dashboard labels this clearly.

## Consequences
- Guest input never leaves the VM; no data-use exposure from free tiers.
- Demo extraction quality is a 7B model's — the cascade (rules → classifier first) keeps most traffic off
  the LLM anyway, and cassettes keep CI independent of it.
- ARM images are required for every service (all chosen components publish them).
- Oracle free-tier capacity can be hard to obtain; the Hetzner fallback costs a few euros/month.

## Alternatives considered
- **Free-tier cloud API for the demo** — guests' input could be trained on.
- **Paid API for the demo** — unbounded cost from anonymous traffic.
- **Hetzner CX33 (8 GB)** — insufficient memory with a local LLM.
