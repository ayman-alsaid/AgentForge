# AgentForge Testing & Verification

This repository follows a strict claim boundary:

- a documented feature can be **IMPLEMENTED**;
- a documented verification script can support a **TESTED** claim for the covered path;
- a live deployment can support **DEPLOYED** only where the source project explicitly documents it;
- example token/latency numbers are not generalized into benchmarks unless a repeatable measurement study exists.

---

## Documented verification scripts

The source project lists six explicit scripts:

| Script | Scope | Evidence classification |
|---|---|---|
| `test_engine.py` | ReAct core, tool calling, persistence | **TESTED** |
| `test_real_engine.py` | real search → URL read → grounded summary | **TESTED** |
| `test_recipe_skill.py` | trace recording, abstraction, deterministic replay, changed target | **TESTED** |
| `test_guided_skill.py` | chained replay, heuristic URL resolution, official-source outcome | **TESTED** |
| `test_dual_flow.py` | marketplace packaging/run + multi-turn agent memory | **TESTED** |
| `test_mcp_integration.py` | MCP connect → discover → registry → call-as-native → unregister | **TESTED** |

These classifications refer to the documented project verification state. This public portfolio repository does not contain the private source or rerun the suite independently.

---

# Claim → Evidence matrix

## Claim: AgentForge runs a ReAct tool-calling loop

**Status:** IMPLEMENTED / TESTED  
**Evidence:** source architecture + `test_engine.py` / `test_real_engine.py` documentation.  
**Boundary:** this does not imply every external tool/provider failure mode is covered.

---

## Claim: successful traces can become parameterized Skills

**Status:** IMPLEMENTED / TESTED  
**Evidence:** abstractor, `execution_trace`, `variables_schema`, and documented recipe-skill verification.  
**Boundary:** abstraction quality depends on correctly separating future inputs from runtime-derived values.

---

## Claim: deterministic replay uses zero LLM calls

**Status:** IMPLEMENTED / TESTED for the documented replay path  
**Evidence:** deterministic executor design and `test_recipe_skill.py`.  
**Boundary:** “zero LLM calls” does not mean zero external API, database, compute, or network cost.

---

## Claim: Guided Replay can resolve a chained search → read workflow without rescue tokens

**Status:** VERIFIED IN A DOCUMENTED TEST RUN  
**Evidence:** `test_guided_skill.py` documents the heuristic path selecting official documentation with zero rescue tokens for that run.  
**Boundary:** this is not a guarantee that every chained workflow will avoid rescue.

---

## Claim: model rescue cannot silently invent a URL outside the candidate set

**Status:** IMPLEMENTED  
**Evidence:** documented candidate-validation boundary in the Guided Replay design.  
**Boundary:** a valid candidate can still be a poor choice if the ranking/rescue judgment is wrong.

---

## Claim: MCP tools use the same ToolRegistry as local tools

**Status:** IMPLEMENTED / TESTED  
**Evidence:** adapter/namespacing architecture and `test_mcp_integration.py`.  
**Boundary:** lifecycle recovery after restart is separately documented as a limitation.

---

## Claim: marketplace customers do not receive the internal execution trace

**Status:** IMPLEMENTED / TESTED in the documented marketplace flow  
**Evidence:** customer-facing contract + `test_dual_flow.py` description.  
**Boundary:** this concerns API response design; production security still depends on the complete deployed auth/configuration boundary.

---

## Claim: AgentForge is live

**Status:** DEPLOYED according to the latest supplied project README  
**Evidence:** documented live frontend/API URLs and project status table.  
**Boundary:** this portfolio repo does not independently probe current uptime on every review.

---

# Outcome correctness

One of the strongest quality lessons documented in the source project is that a green test is not useful if it proves only “nothing crashed.”

The verification strategy therefore checks behavioral outcomes where possible.

Examples:

- replay must switch to the new topic rather than accidentally reuse the original one;
- guided URL selection must reach the intended official documentation rather than merely return some valid URL;
- marketplace execution must preserve the no-trace-leak contract;
- MCP must complete the actual discovery/registration/call/unregister lifecycle.

This is materially stronger evidence than route-existence or no-exception tests alone.

---

# What is not claimed

This repository does **not** claim:

- that six scripts equal exhaustive automated coverage;
- that every Skill replay is zero-token;
- that every Guided Replay resolves heuristically;
- that the example latency/token figures in the private project record are universal benchmarks;
- that longer arbitrary dependency graphs are fully generalized;
- that deterministic replay removes third-party API costs.

Those boundaries are part of the evidence model, not disclaimers added after the fact.
