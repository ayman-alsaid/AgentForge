# AgentForge — Limitations & Technical Boundaries

AgentForge is designed around explicit boundaries. The purpose of this document is not to weaken the project; it is to separate what the architecture already solves from what still requires engineering work or domain validation.

---

## 1. Guided Replay is strongest on short dependency chains

The documented implementation is particularly strong on patterns such as:

```text
search → choose result → read
```

Longer chains create more dependency-resolution states and increase the chance that an intermediate value genuinely needs fresh reasoning.

**Current boundary:** the system should not be represented as a general-purpose workflow compiler for arbitrary dependency graphs.

---

## 2. MCP live connection recovery requires lifecycle hardening

MCP connection metadata is persisted, while live client objects are runtime state.

The source project documents restart recovery / auto-reconnect as a known lifecycle gap.

**Implication:** a backend restart can require re-establishing live remote-tool connections even though metadata exists.

---

## 3. A recorded Skill can become stale

Deterministic execution is only correct while its assumptions remain correct.

Potential sources of staleness include:

- changed website structure;
- changed API contracts;
- new authentication requirements;
- altered organizational policy;
- deprecated MCP tools;
- modified external search behavior.

A production replay system therefore needs operational signals that indicate a Skill should be reviewed or re-recorded.

---

## 4. Heuristics are domain-specific

The documented smart URL scorer is appropriate to documentation/search workflows.

It does **not** automatically generalize to:

- choosing the correct SRE log query;
- resolving a CRM entity;
- interpreting a finance workflow decision;
- selecting an RPA UI target.

The architecture generalizes; the heuristic vocabulary does not.

---

## 5. Zero-token replay is not zero-cost execution

A deterministic replay can still consume:

- search API quota;
- HTTP requests;
- database operations;
- network bandwidth;
- remote MCP service resources;
- third-party paid API calls.

The valid claim is: **no LLM reasoning call occurs on the deterministic replay path.**

---

## 6. Not every workflow should be replayed

Replay is valuable when repetition is desirable.

For tasks where fresh judgment, novelty, or creativity is the core value, deterministic reuse may be actively harmful.

A content-generation workflow is a useful example: replaying the same successful structure too aggressively can produce stale, repetitive output.

---

## 7. Model rescue remains probabilistic

Candidate validation prevents a rescue model from inventing a value outside the allowed set, but it cannot guarantee that the selected allowed value is the best one.

Constrained probabilistic choice is safer than unconstrained generation; it is not deterministic correctness.

---

## 8. Marketplace secrecy is scoped to the contract boundary

Customer endpoints are designed not to expose internal traces and arguments.

That does not mean a workflow is cryptographically impossible to infer from its behavior. Intellectual-property protection still depends on deployment design, access controls, and what information the product output itself reveals.

---

## 9. Public repository ≠ production repository

This repository intentionally contains engineering evidence, not the private production implementation.

The absence of production source here should not be interpreted as absence of implementation; equally, readers should not treat architecture documentation as a substitute for code-level security review.

---

## 10. Example token/latency values are not generalized benchmarks

The source project contains example cost/latency figures for specific executions.

Unless a repeated benchmark methodology is published, those numbers should remain run-specific observations rather than platform-wide performance guarantees.

---

## Why these limitations matter

AgentForge's strongest claim is not “AI reasoning is no longer needed.”

It is narrower and more defensible:

> **When successful reasoning has become repeatable structure, the system should know how to stop paying for the reasoning part unnecessarily.**

The limitations above identify where that transition from fresh reasoning to reusable structure is safe, where it is probabilistic, and where it still requires additional engineering.
