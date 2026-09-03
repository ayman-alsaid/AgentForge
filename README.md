# AgentForge — Reusable Intelligence for Tool-Using AI Agents

<p align="center">
  <img src="https://img.shields.io/badge/AgentForge-Agent%20Execution%20Engine-5E7FA3?style=flat-square" alt="AgentForge" />
  <img src="https://img.shields.io/badge/Skills-Record%20Once%20%C2%B7%20Replay%20Cheap-6E8F88?style=flat-square" alt="Skills" />
  <img src="https://img.shields.io/badge/Replay-Deterministic%20%2B%20Guided-756F9C?style=flat-square" alt="Replay" />
  <img src="https://img.shields.io/badge/MCP-HTTP%20%2F%20SSE-6E8097?style=flat-square" alt="MCP" />
  <img src="https://img.shields.io/badge/Evidence-Outcome--Correctness%20Tests-7F7792?style=flat-square" alt="Evidence" />
</p>

> **What if an AI agent only paid full reasoning cost the first time it solved a repeatable problem?**

AgentForge is a tool-calling AI agent execution engine built around one economic idea:

**successful reasoning should become reusable infrastructure.**

A normal agent can search, read, call tools, reason, and persist a conversation. AgentForge adds a second layer on top of that behavior: it records successful tool-use traces, abstracts them into parameterized **Skills**, and replays those Skills without re-running full LLM reasoning whenever the workflow is genuinely repeatable.

The result is not response caching. It is a **record → abstract → resolve → replay → package** architecture for reusable agent behavior.

**Live:** https://agentforge.agentcraft.info  
**API:** https://api.agentforge.agentcraft.info  
**Portfolio:** https://agentcraft.info

---

## Why this project exists

Most agent cost discussions focus on model choice: use a smaller model, quantize, route to a cheaper provider.

AgentForge attacks a different layer of the problem:

> **Why are we paying a model to rediscover a workflow it has already successfully discovered?**

Consider a recurring task:

```text
search for the latest release of X
→ choose the official documentation result
→ read the page
→ summarize the important changes
```

A conventional agent reasons through that sequence again on run 2, run 10, and run 10,000.

AgentForge lets the first successful run become a reusable execution asset.

```text
FULL REASONING ONCE
        │
        ▼
 successful tool trace
        │
        ▼
 conservative abstraction
        │
        ▼
 parameterized Skill
        │
   ┌────┴───────────────┐
   │                    │
independent         chained
steps               ambiguity
   │                    │
   ▼                    ▼
deterministic       cheapest-first
replay              resolution
0 LLM calls         free → free → tiny rescue
   │                    │
   └─────────┬──────────┘
             ▼
      reusable product
```

That is the engineering thesis of AgentForge:

**Use AI where reasoning is genuinely needed. Replace it with deterministic execution everywhere else.**

---

# The core problem is not caching

Caching an LLM's text response is easy, but brittle. It works only when the future request is sufficiently close to the cached request.

AgentForge operates one level deeper: on the **execution trace**.

A successful run may contain:

- a search query;
- tool arguments;
- returned URLs;
- intermediate results;
- dependent tool calls;
- the final response.

The difficult question is not “can this trace be stored?” It is:

> **Which parts of this trace are true future inputs, which parts are deterministic, and which parts cannot be known until a fresh run produces new intermediate data?**

That boundary is where most of AgentForge's engineering lives.

---

# System architecture

```text
                       ┌──────────────────────────────┐
                       │      Next.js 15 Frontend      │
                       │ chat · agents · skills · MCP  │
                       │ marketplace · EN/AR + RTL     │
                       └──────────────┬───────────────┘
                                      │ REST / JWT / SSE
                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                         FastAPI Backend                          │
│ auth · agents · chat · execution · skills · marketplace · MCP   │
└────────┬──────────────────────────┬─────────────────────────────┘
         │                          │
         ▼                          ▼
┌─────────────────┐        ┌─────────────────────────────┐
│  ReAct Executor │ trace  │       Skill Pipeline        │
│  5-iteration    │───────▶│ recorder → abstractor →     │
│  safety cap     │        │ deterministic / guided      │
└────────┬────────┘        └─────────────┬───────────────┘
         │ tools                         │ tools
         ▼                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         ToolRegistry                              │
│ local tools + namespaced MCP tools through one interface         │
│ web_search · read_url · mcp__{server}__{tool}                    │
└─────────────┬───────────────────────────────┬───────────────────┘
              │                               │
              ▼                               ▼
      OpenRouter / Tavily             MCP servers over HTTP/SSE
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│ PostgreSQL + pgvector · Redis · Celery · persisted conversations │
└─────────────────────────────────────────────────────────────────┘
```

All LLM calls are routed through raw `httpx` requests to OpenRouter rather than an SDK-specific execution path. That keeps model selection replaceable by model slug and prevents the executor from being coupled to one vendor client library.

---

# 1. ReAct execution engine

At the base is a standard tool-using ReAct loop:

1. send the user message, system prompt, history, and tool schemas to the model;
2. execute returned tool calls;
3. append each tool result using the correct `tool_call_id` wire format;
4. feed results back to the model;
5. repeat until a final answer or the five-iteration safety cap.

The important implementation decision is that tools are not hard-coded into the executor.

Everything resolves through a central **ToolRegistry**.

A local Python tool and a remote MCP tool therefore become equivalent at the execution boundary. The executor does not need a special branch for “MCP execution.” Once adapted, the tool is just another `BaseTool`.

That makes adding tools a registry problem, not an executor rewrite.

---

# 2. Trace capture — record what actually happened

Every successful execution can be captured with:

- tool name;
- input arguments;
- result;
- error state;
- duration;
- model/token metadata where relevant.

The system records the **actual execution**, not a human-authored description of the intended workflow.

That distinction matters. A replayable workflow should be based on evidence of what succeeded, not on what someone believes should have happened.

---

# 3. Conservative abstraction — turning a trace into a Skill

The abstractor walks the captured trace and replaces selected concrete arguments with variables such as:

```text
"latest FastAPI version"
        ↓
{{var_1}}
```

The Skill stores both:

- an `execution_trace` containing placeholders;
- a `variables_schema` describing future inputs.

The abstraction is deliberately conservative.

URLs and already-templated values are skipped by default via `skip_urls` because a URL inside a trace is commonly an **intermediate result from a previous tool**, not something a future customer should override manually.

This is an intentional trade-off:

> fewer variables, but variables that actually mean something.

A maximally aggressive abstractor would make a Skill look more configurable while simultaneously making it easier to break.

The same `variables_schema` later drives the frontend input form and marketplace customer form, avoiding a second translation layer between execution and product packaging.

---

# 4. Deterministic replay — the zero-reasoning path

If every required value can be known before execution, the Skill takes the cheapest possible path:

```text
inputs
  ↓
fill placeholders
  ↓
execute recorded tools in order
  ↓
collect results
```

**No LLM reasoning call occurs in this path.**

That is the strongest economic property of the system: a workflow that is genuinely deterministic does not receive an AI call simply because the product is branded as an AI agent platform.

The project test suite includes a record → abstract → replay path specifically designed to verify that replay can change the target topic while remaining at zero LLM tokens for the deterministic replay itself.

---

# 5. Guided replay — where the real engineering starts

Deterministic replay is straightforward until steps depend on fresh intermediate results.

Example:

```text
Step 1: search for "latest Celery release"
Step 2: read the official URL returned by Step 1
```

The URL for Step 2 does not exist until Step 1 runs.

The easy solution would be:

> “This chain is dynamic, so ask the full agent to reason again.”

AgentForge explicitly avoids that fallback because it would erase most of the value of Skills for exactly the workflows most worth optimizing.

Instead, **Guided Replay** resolves ambiguity using the cheapest strategy that can plausibly work.

```text
value required at runtime
        │
        ▼
1. user supplied it? ──────────────► use it
        │ no                         FREE
        ▼
2. heuristic extraction from
   previous result? ───────────────► use it
        │ no                         FREE
        ▼
3. tiny LLM rescue for ONLY
   this ambiguous value ───────────► validate answer
        │                            LOW COST
        ▼
4. recorded default
```

The model rescue is not allowed to re-plan the task. It resolves one missing value with a small token budget.

Every step records **how** its value was resolved:

- `user_input`
- `heuristic`
- `llm_rescue`
- `text_passthrough`
- `default`

The frontend exposes those strategies directly as cost/reliability indicators rather than hiding them under an “AI optimized” label.

This is a key design principle:

> **Cost optimization should be inspectable.**

---

# 6. Smart URL selection before spending a token

Search → read chains create a recurring ambiguity: which returned link should the next step consume?

Before asking any model, AgentForge scores URL candidates using deterministic signals.

Examples documented in the project include:

```text
trusted keywords such as docs / documentation / official / github  → positive weight
trusted TLDs such as .org / .dev / .io                             → positive weight
query-token matches                                                 → positive weight
known low-quality / spam-like patterns                              → penalty
```

When one candidate clearly leads the others, Guided Replay selects it without an LLM call.

Only genuinely ambiguous rankings escalate to the rescue model.

The rescue answer is then checked against the **actual candidate set**. A model cannot invent a plausible-looking URL and have it silently accepted as reality.

This is a small architectural choice with a large reliability effect:

**probabilistic selection is allowed; probabilistic existence is not.**

---

# 7. Anti-hallucination validation

LLMs are useful when an ambiguity cannot be resolved deterministically, but they should not be trusted to manufacture the domain of possible answers.

For guided URL resolution, the candidate list is produced by the real tool call. The model may choose among candidates, but the returned answer must correspond to something that actually exists in that candidate set.

This changes the role of the model from:

```text
"Tell me the URL I should use"
```

into:

```text
"Choose the most appropriate value from these real values"
```

That boundary is representative of the broader AgentCraft engineering philosophy:

**give AI capabilities, but put deterministic constraints around the places where invention would be dangerous.**

---

# 8. Multilingual search handling

For non-English search queries, the project can translate the query before sending it to the search provider while preserving both the original and translated form in structured output.

This exists to improve retrieval quality for documentation-heavy queries while keeping the transformation observable rather than silently replacing what the user asked.

---

# 9. MCP without a second execution engine

AgentForge integrates Model Context Protocol tools over **HTTP/SSE using JSON-RPC 2.0**.

The MCP lifecycle includes:

- `initialize`;
- `notifications/initialized`;
- `tools/list`;
- `tools/call`;
- `ping`;
- SSE response parsing;
- connect / list / refresh / disconnect operations.

Remote tools are wrapped as normal `BaseTool` instances and namespaced as:

```text
mcp__{server}__{tool}
```

That avoids collisions when two MCP servers expose tools with the same name.

### Deliberate non-feature: no stdio MCP

Stdio support was intentionally not implemented.

On a shared or multi-tenant server, stdio MCP implies spawning arbitrary subprocesses. AgentForge chooses the smaller attack surface of HTTP/SSE rather than treating protocol completeness as more important than deployment safety.

This is one of the project's strongest examples of a **deliberate non-feature** being meaningful engineering evidence.

---

# 10. Skill packaging — execution becomes a product

A Skill is not only an optimization primitive.

It can also be packaged as a commercial unit with fields such as:

- product name;
- description;
- pricing;
- currency;
- published state;
- customer input schema.

The customer form is generated from the same variable schema produced by the abstractor.

The commercial path therefore becomes:

```text
successful execution
→ captured trace
→ reusable Skill
→ verified replay
→ packaged product
```

No separate workflow-export format is required.

### Information hiding is enforced at the API boundary

Marketplace customers do **not** receive the internal execution trace or private tool arguments.

This is not merely hidden in the UI. Customer-facing endpoints are designed not to expose those internals.

A customer buys the **result contract**, not the implementation recipe.

---

# 11. Persistent agents and conversations

AgentForge also supports the conventional agent product layer:

- JWT authentication;
- user-owned agents;
- conversation persistence;
- bounded conversation memory;
- stored user / assistant / tool messages;
- model and token metadata;
- service-to-service execution through `X-API-Key` with constant-time comparison.

This matters because Skills are built from real agent activity rather than existing as a disconnected automation feature.

---

# 12. Frontend as an observability surface

The Next.js interface is not simply a chat window.

It exposes the properties the engine cares about:

- live ToolCallCards;
- arguments and result previews;
- duration/status;
- Save-as-Skill flow;
- generated variable inputs;
- deterministic vs guided replay;
- real token-cost banners;
- per-step resolution strategy chips;
- MCP connection/tool discovery;
- marketplace packaging and customer forms;
- English / Arabic with full RTL support;
- dark / light themes.

The source project documents 10 routes and 53 components for the application interface.

The important design point is not the component count. It is that **cost and reasoning decisions are visible to the operator** rather than buried in backend logs.

---

# Evidence: what has actually been verified

The original project record documents six explicit verification scripts.

| Verification path | What it demonstrates | Evidence classification |
|---|---|---|
| `test_engine.py` | ReAct loop, tool calling, persistence | **TESTED** in documented project run |
| `test_real_engine.py` | search → read → grounded summary | **TESTED** in documented project run |
| `test_recipe_skill.py` | record → abstract → correct topic switch with zero-token deterministic replay | **TESTED** in documented project run |
| `test_guided_skill.py` | chained replay chooses the correct official documentation via the heuristic path without rescue tokens | **TESTED** in documented project run |
| `test_dual_flow.py` | marketplace packaging/run + multi-turn memory | **TESTED** in documented project run |
| `test_mcp_integration.py` | MCP connect → discover → registry → call-as-native → unregister | **TESTED** in documented project run |

The project specifically emphasizes **outcome correctness**, not merely “the script did not throw an exception.”

That distinction came from an actual review lesson: a vacuously green test can be more dangerous than a visibly failing one because it creates confidence without proving behavior.

The public evidence repository does not claim that these six scripts constitute exhaustive coverage of every production path.

---

# A verified 5-minute story

The project's intended end-to-end demonstration is straightforward:

1. create an agent with `web_search` and `read_url`;
2. ask it to find and summarize documentation;
3. watch the real tool calls execute;
4. save the successful execution as a Skill;
5. replay the Skill with a different topic;
6. observe deterministic/guided resolution and real cost metadata;
7. package the Skill as a marketplace product;
8. run it through the customer-facing contract without exposing its internal trace.

The important part is that this is one continuous lifecycle:

**reasoning → evidence → abstraction → reusable execution → productization.**

---

# Engineering decisions that matter most

## Decision 1 — Tiered resolution instead of full reasoning fallback

**Alternative:** if any chained value is ambiguous, invoke the full agent again.

**Why rejected:** the workflows most worth optimizing are often chained workflows. A full reasoning fallback would preserve product functionality while destroying the economic thesis.

**Trade-off accepted:** Guided Replay becomes more complex because it must maintain several resolution paths and make their behavior inspectable.

---

## Decision 2 — Conservative abstraction instead of maximal parameterization

**Alternative:** convert every concrete value in a trace into a variable.

**Why rejected:** intermediate outputs are not necessarily meaningful customer inputs. Turning them into form fields increases fragility.

**Trade-off accepted:** some customization requires a new recording instead of another exposed knob.

---

## Decision 3 — Validate rescue answers against reality

**Alternative:** accept a plausible LLM-produced URL/value.

**Why rejected:** selection and invention are different permissions. The model is allowed to select a candidate, not create a candidate that did not exist.

**Trade-off accepted:** invalid rescue output can fall back rather than always producing a graceful perfect choice.

---

## Decision 4 — One ToolRegistry for local and MCP tools

**Alternative:** build separate execution code paths for local tools and MCP.

**Why rejected:** execution semantics should depend on the tool contract, not its network origin.

**Trade-off accepted:** protocol translation complexity is concentrated in the MCP adapter.

---

## Decision 5 — HTTP/SSE MCP only

**Alternative:** support stdio for completeness.

**Why rejected:** arbitrary subprocess execution is an undesirable boundary on a shared host.

**Trade-off accepted:** some stdio-only MCP servers cannot connect without an HTTP bridge.

---

## Decision 6 — Hide marketplace internals at the API layer

**Alternative:** return complete Skill objects and hide sensitive fields only in the frontend.

**Why rejected:** UI concealment is not an information-security boundary.

**Trade-off accepted:** customer-side debugging has intentionally less visibility than owner-side execution inspection.

---

# The economics of reusable reasoning

AgentForge's central economic distinction is qualitative rather than a universal benchmark:

| Execution type | Reasoning requirement |
|---|---|
| Fresh ReAct run | full model reasoning |
| Deterministic Skill replay | **no LLM call in replay path** |
| Guided replay, heuristics succeed | **no rescue LLM call** |
| Guided replay, ambiguity remains | narrowly scoped rescue call only |

The source project contains example token/latency figures for particular flows. This portfolio repository does **not** generalize those individual figures into platform-wide performance guarantees.

What is architectural — and therefore the stronger claim — is that deterministic replay removes the reasoning call by construction, while Guided Replay attempts cheaper resolution before model rescue.

---

# Generalizable architecture

The deeper pattern is:

```text
RECORD ONCE
   ↓
ABSTRACT CONSERVATIVELY
   ↓
RESOLVE AMBIGUITY CHEAPEST-FIRST
   ↓
REPLAY DETERMINISTICALLY WHERE POSSIBLE
   ↓
PACKAGE WITHOUT LEAKING THE MECHANISM
```

That pattern is not specific to web search.

The AgentForge content system explores credible adaptations to:

- robotic process automation;
- DevOps / SRE incident runbooks;
- customer-support runbooks;
- sales research and enrichment;
- competitive-intelligence monitoring.

The generalization has a boundary: each domain still needs domain-specific tools and domain-specific heuristics. A URL scorer is not a log-query resolver. Reusable architecture does not mean interchangeable domain logic.

A second important boundary is **fresh judgment**. Creative-generation workflows, for example, are a poor target for deterministic replay when novelty is the point of the work. Optimization is only valuable when repetition is actually desirable.

---

# Known limitations

The source project explicitly documents limits rather than hiding them.

### MCP reconnect lifecycle

MCP live connections are tracked in memory. The project documents auto-reconnect after backend restart as an area requiring lifecycle hardening.

### Longer guided chains

The documented Guided Replay design is strongest on short chains such as search → select URL → read. Longer dependency graphs increase ambiguity and require more generalized dependency resolution.

### Skill staleness

A recorded workflow assumes enough of its environment remains stable to replay successfully. External APIs, site structures, policies, or tool contracts can change. A replay engine therefore needs monitoring and re-recording discipline; deterministic execution does not magically make the surrounding world deterministic.

### Zero-token means replay reasoning, not zero infrastructure cost

A deterministic Skill may call external tools, APIs, databases, or services that carry their own cost. “0 tokens” describes the absence of LLM calls in the replay path, not a claim that the entire end-to-end execution is economically free.

---

# What I built vs. what I used

AgentForge uses external model and search providers. Those are not the project's engineering contribution.

The original work documented here is the surrounding execution architecture:

- ReAct executor and persistence model;
- central ToolRegistry;
- trace recorder;
- conservative trace abstraction;
- deterministic recipe executor;
- Guided Replay resolution engine;
- smart URL selection;
- anti-hallucination candidate validation;
- MCP adapter/registry lifecycle;
- Skill packaging and customer information boundaries;
- frontend observability around tool calls, replay modes, costs, and resolution strategy.

In other words:

> **The interesting question is not whether an LLM can call a tool. It is how much of yesterday's successful reasoning we can safely avoid paying for tomorrow.**

---

# Technology stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 · TypeScript · Tailwind CSS · shadcn-style primitives · EN/AR RTL |
| Backend | FastAPI · async SQLAlchemy · Pydantic |
| Data | PostgreSQL 16 · pgvector · Alembic |
| Async / cache | Redis · Celery |
| LLM | OpenRouter through raw `httpx` |
| Search / read | Tavily · HTTP page parsing |
| Tool interoperability | MCP JSON-RPC 2.0 over HTTP/SSE |
| Auth | JWT · bcrypt · service `X-API-Key` |
| Infrastructure | Docker Compose · reverse proxy · TLS |

---

# Evidence map

- [`docs/CASE_STUDY.md`](docs/CASE_STUDY.md) — full engineering narrative, alternatives, trade-offs, and lessons.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — execution topology, Skill lifecycle, MCP boundary, data flow.
- [`docs/ENGINEERING_DECISIONS.md`](docs/ENGINEERING_DECISIONS.md) — decision records for the architecture's most important boundaries.
- [`docs/TESTING_AND_VERIFICATION.md`](docs/TESTING_AND_VERIFICATION.md) — documented test paths and claim classification.
- [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md) — current constraints and operational risks.
- [`evidence/controlled-runs/README.md`](evidence/controlled-runs/README.md) — public evidence index and artifact plan.
- [`PORTFOLIO_NOTICE.md`](PORTFOLIO_NOTICE.md) — production-source and review-scope boundary.

---

# Source code & review scope

The production source code for AgentForge is maintained privately and is not distributed through this portfolio repository.

This public repository exists as **Engineering Evidence / Technical Case Study material**. Its purpose is to expose enough architecture, behavioral evidence, engineering decisions, trade-offs, verification methodology, and limitations for a technical reviewer to evaluate the work without publishing proprietary implementation details, credentials, or production secrets.

---

<p align="center">
  <strong>Record once. Resolve cheap. Replay deterministic.</strong><br/>
  Build agents that remember successful work as executable structure, not just conversation history.
</p>

---

**Ayman Alsaid** · Senior AI / Product Engineer  
**AgentCraft** · https://agentcraft.info · contact@agentcraft.info
