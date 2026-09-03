# AgentForge — Controlled-Run Evidence Index

This directory is reserved for sanitized evidence artifacts that can be published without exposing the private production source, credentials, or sensitive operational data.

The source project documents the following high-value verification paths.

---

## 1. ReAct core execution

**Script:** `test_engine.py`  
**Purpose:** exercise the tool-calling loop and persistence behavior.  
**Classification:** TESTED in the documented project run.

Suggested public artifact when available:

- sanitized transcript of one tool-calling execution;
- tool name / arguments / result / duration;
- final outcome assertion.

---

## 2. Real search → read → grounded summary

**Script:** `test_real_engine.py`  
**Purpose:** verify a real multi-tool path rather than a mocked control flow.  
**Classification:** TESTED in the documented project run.

Suggested public artifact:

- search query;
- selected URL;
- source domain;
- final summary excerpt;
- redacted token/cost metadata if appropriate.

---

## 3. Deterministic Skill replay

**Script:** `test_recipe_skill.py`  
**Purpose:** prove trace capture → abstraction → replay while switching from the original topic to a new one.  
**Key property:** no LLM call in the deterministic replay path.  
**Classification:** TESTED in the documented project run.

The strongest public proof would show:

```text
original run topic: FastAPI
replay topic: Celery
replay result: actually about Celery
LLM reasoning calls during deterministic replay: 0
```

This is more meaningful than merely showing a green test status.

---

## 4. Guided chained replay

**Script:** `test_guided_skill.py`  
**Purpose:** verify a search → dynamic URL selection → read chain.  
**Documented result:** heuristic resolution selects the intended official documentation without rescue-token usage in that run.  
**Classification:** VERIFIED IN A DOCUMENTED TEST RUN.

Suggested public artifact:

- fresh search results;
- scored candidate URLs;
- selected URL;
- resolution strategy (`heuristic`);
- rescue-call count;
- final outcome assertion.

---

## 5. Marketplace + multi-turn flow

**Script:** `test_dual_flow.py`  
**Purpose:** exercise Skill packaging/customer execution plus conventional agent memory behavior.  
**Key contract:** customer execution must not expose the internal execution trace.  
**Classification:** TESTED in the documented project run.

---

## 6. MCP lifecycle

**Script:** `test_mcp_integration.py`  
**Purpose:** verify the lifecycle:

```text
connect
→ initialize
→ discover tools
→ register namespaced tool
→ call through normal ToolRegistry
→ unregister / disconnect
```

**Classification:** TESTED in the documented project run.

---

# Evidence publication rule

Until raw screenshots/transcripts are sanitized and copied into this directory, this public repository should describe these as **documented verification results**, not imply that every private artifact has already been published.

The intended standard is:

**Claim → Status → Evidence → Scope → Boundary.**
