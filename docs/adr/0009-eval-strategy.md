# 0009 — Eval strategy: golden set, phased metrics, recorded cassettes in CI

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
Extraction quality must gate merges, but live LLM calls in CI are non-deterministic, cost money, and need a
secret. Some metrics (tool-call correctness, policy violations) only make sense once tools and policy exist.
Real labeled data won't exist on day one.

## Decision
- **Golden set (~200 items):** Roman Urdu, Urdu script, English, typos, voice transcripts, quote photos.
  v1 is **synthetic, seeded from the real Day-0 contractor thread** and corrected by the founder; if no
  thread is available, it is **fully synthetic** from the SKU list + persona patterns. It grows with
  human-corrected production data. Versioned in the repo.
- **Classifier training data is held out** from the golden eval set (no overlap).
- **Metrics ramp by phase:**
  - P2: extraction field-F1, SKU-match accuracy, intent accuracy, % resolved without an LLM, cost/msg.
  - P3 adds: tool-call correctness (copilot), **policy-violation rate — must be 0**.
- **CI gate replays recorded LLM cassettes** (keyed by prompt hash) — no live key, deterministic results.
  A **cassette miss fails CI** with a clear message; re-record locally via `make evals-record` (dev key,
  synthetic data only) and commit the cassettes **in the same PR** as the prompt/schema change.
- A **manually triggered live-eval job** runs against the real model periodically to catch provider drift.
- A PR that regresses below threshold, or produces any policy violation, fails.

## Consequences
- CI is fast, free, and reproducible; prompt changes show up as reviewable cassette diffs.
- Cassettes can go stale relative to the live model — the live job is the safety net.
- Cassettes contain only synthetic data, consistent with the free-tier rule.

## Alternatives considered
- **Live LLM in CI** — flaky, costly, needs secrets in CI.
- **Skip on cassette miss** — silently weakens the gate.
- **No gate until real data exists** — quality regressions go unnoticed during the build.
