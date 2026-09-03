# AgentForge Engineering Decisions

This document records the decisions that best explain *why* AgentForge is architected the way it is.

The recurring theme is simple:

> **Do not spend probabilistic reasoning where deterministic structure already exists.**

---

## ADR-01 — Replay execution traces, not cached answers

**Decision:** store and abstract successful tool-use traces rather than caching final text responses.

**Reason:** a cached answer is tied to one request. A trace captures the reusable process that produced the answer.

**Trade-off:** trace replay requires variable semantics, dependency handling, and tool-level persistence that simple response caching does not.

---

## ADR-02 — Deterministic replay is the default Skill mode

**Decision:** if all required values are known from caller input or the recipe, execute tools directly without an LLM call.

**Reason:** injecting a model into a deterministic workflow would add cost, latency, and variance without adding useful intelligence.

**Boundary:** “zero tokens” refers to LLM reasoning in the replay path, not external API/infrastructure cost.

---

## ADR-03 — Cheapest-first resolution for dynamic chains

**Decision:** resolve runtime dependencies in this order:

1. user input;
2. deterministic/heuristic extraction;
3. narrowly-scoped model rescue;
4. recorded default.

**Rejected alternative:** invoke the complete agent whenever a dependency is dynamic.

**Reason:** full-agent fallback would destroy the cost advantage of Skills for common chained workflows.

**Trade-off:** more execution states and a larger verification surface.

---

## ADR-04 — Conservative abstraction with `skip_urls`

**Decision:** avoid turning values such as URLs into user parameters by default when they are likely to be outputs of previous steps.

**Reason:** an intermediate output is not automatically a meaningful future input.

**Rejected alternative:** template every concrete argument.

**Trade-off:** fewer customization controls, but a lower chance of callers breaking a valid dependency chain.

---

## ADR-05 — Score URL candidates before model rescue

**Decision:** use deterministic ranking signals to select likely official/relevant URLs before spending an LLM call.

**Reason:** most ambiguity is not genuinely semantic enough to justify a model.

**Trade-off:** heuristic scoring needs domain-aware tuning and cannot guarantee correctness for every source set.

---

## ADR-06 — Model rescue may select but not invent

**Decision:** validate model-resolved values against the actual candidate set.

**Reason:** the source of truth for possible URLs/values is the real preceding tool result, not the model's imagination.

**Trade-off:** invalid rescue output may fall through to a default instead of being accepted as a plausible answer.

---

## ADR-07 — Unified ToolRegistry

**Decision:** local tools and remote MCP tools implement the same execution contract.

**Reason:** the executor should depend on capabilities, not transport origin.

**Result:** MCP support can be added through adapters/namespacing without modifying the core ReAct loop.

---

## ADR-08 — MCP over HTTP/SSE, not stdio

**Decision:** deliberately omit stdio MCP in the shared-host deployment model.

**Reason:** stdio requires local subprocess execution and materially enlarges the trust boundary.

**Trade-off:** stdio-only MCP servers require a bridge or cannot connect directly.

---

## ADR-09 — Marketplace hides execution internals server-side

**Decision:** customer-facing responses omit execution traces and internal tool arguments at the API boundary.

**Reason:** frontend-only hiding is not a security or intellectual-property boundary.

**Trade-off:** customer execution has intentionally less debugging visibility than owner execution.

---

## ADR-10 — Resolution provenance is part of product UX

**Decision:** expose how each guided value was resolved instead of showing only a final “optimized” label.

**Reason:** cost-aware agent systems should be auditable at the point where cost and reliability trade off.

**Result:** the UI can distinguish free direct matches, heuristics, rescue calls, and defaults.

---

## ADR-11 — Outcome-correctness tests over vacuous success

**Decision:** tests must assert that the result is semantically the intended result, not merely that execution completed.

**Reason:** a project review identified the danger of tests that pass while proving almost nothing.

**Example:** a replay test should establish that the result actually switched from FastAPI-related content to Celery-related content, not just that the replay returned a string.

---

## ADR-12 — Persist known limitations

**Decision:** keep lifecycle and generalization gaps visible rather than describing the engine as universally solved.

Documented examples include MCP restart recovery and longer guided-chain generalization.

**Reason:** technical credibility increases when architectural boundaries are explicit.
