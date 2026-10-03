# 0002 — LLM autonomy boundary: deterministic workflow, read-only copilot, no `plan()`

- **Status:** Accepted
- **Date:** 2026-10-04

## Context
Early docs promised "agentic tool-calling" and a tool registry that generates an LLM tool schema, while
also requiring that the LLM never decides anything and that the workflow runs in a fixed order. A
`plan()` function appeared in one doc and not another. We need one clear boundary.

## Decision
- **`ProcurementWorkflow` is deterministic**, orchestrated by Temporal in a fixed order. The LLM does not
  choose its steps.
- The LLM's roles are exactly:
  1. **Extract** — `extract_material_request`, `extract_quote`, `extract_invoice` → Pydantic-validated objects.
  2. **Answer** — `answer_grounded` → cited answer or refusal (RAG).
  3. **Assist** — `assist`, an owner-facing copilot that **may propose tool calls only for tools with
     `side_effect="none"`** (the "Copilot" column in `docs/BACKEND.md` §4).
- **There is no `plan()`.**
- Tool calls never originate from a turn whose content came from a supplier/external party.

## Consequences
- "Agentic tool-calling" is real but bounded to read-only tools; write paths are reachable only from
  deterministic code behind the policy engine.
- "Tool-call correctness" is measured on the copilot; injection tests cover both the copilot and the
  extraction path (`docs/AI.md` §8).
- The demo story stays honest: "AI is the fallback, not the decider."

## Alternatives considered
- **LLM planner chooses workflow steps** — non-deterministic, hard to replay/audit, unsafe near money.
- **No tool calling at all** — loses a legitimate showcase of allow-listed, guarded tool use.
