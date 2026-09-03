# AgentForge Architecture

## System topology

```text
Next.js 15 frontend
   │
   │ REST / JWT / SSE
   ▼
FastAPI backend
   ├── auth / users / agents
   ├── conversations / chat
   ├── execution
   ├── skills / replay
   ├── marketplace
   └── MCP lifecycle
        │
        ├───────────────► PostgreSQL + pgvector
        │                  users · agents · conversations · messages
        │                  skills · mcp_connections
        │
        ├───────────────► Redis / Celery
        │
        ▼
ReAct Executor
        │
        ▼
ToolRegistry
   ├── local tools
   └── MCP adapters
        │
        ├── Tavily-backed web_search
        ├── read_url
        └── mcp__{server}__{tool}

LLM access: OpenRouter through raw httpx
MCP transport: JSON-RPC 2.0 over HTTP/SSE
```

---

## ReAct execution flow

```text
user input
   ↓
load bounded conversation history
   ↓
model call with tool schemas
   ↓
tool calls?
   ├── yes → registry execution → persist tool messages/results → model again
   └── no  → persist final answer
   ↓
stop at final answer or iteration cap
```

The executor is capped at five reasoning iterations to prevent runaway loops.

Tool calls and results are persisted using the OpenAI-compatible `tool_call_id` relationship so multi-turn tool history remains reconstructable.

---

## Skill lifecycle

```text
successful agent execution
        │
        ▼
Recorder
captures tool + args + result + duration/error
        │
        ▼
Abstractor
identifies future inputs
        │
        ├── true input → {{var_N}}
        └── derived/runtime value → protected / skipped where appropriate
        │
        ▼
Skill
execution_trace + variables_schema + replay_mode
        │
   ┌────┴────────┐
   ▼             ▼
Deterministic   Guided
Replay          Replay
```

---

## Deterministic replay data flow

```text
caller variables
   ↓
placeholder substitution
   ↓
recorded step 1 → ToolRegistry
   ↓
recorded step 2 → ToolRegistry
   ↓
...
   ↓
results
```

There is no LLM reasoning call in this replay path.

---

## Guided replay data flow

Guided Replay is required when later steps depend on fresh results produced by earlier steps.

```text
step requires unresolved value
       │
       ▼
caller supplied value?
       ├── yes → use
       └── no
            ↓
heuristic extraction from prior results?
       ├── yes → use
       └── no
            ↓
narrow LLM rescue
            ↓
validate against allowed candidates
       ├── valid → use
       └── invalid / unavailable → recorded default
```

Each step stores resolution provenance so the system can explain whether the value came from `user_input`, a heuristic, `llm_rescue`, passthrough behavior, or a default.

---

## Smart URL resolution

For search → read chains, candidate links are scored with deterministic signals before model rescue.

The architecture deliberately separates:

1. **candidate discovery** — performed by the real search tool;
2. **candidate ranking** — deterministic when possible;
3. **ambiguous selection** — model-assisted only if required;
4. **candidate validation** — deterministic enforcement that the chosen value really existed.

This keeps the model out of the role of inventing source URLs.

---

## ToolRegistry boundary

All executable tools conform to one interface.

```text
                 BaseTool contract
                       │
        ┌──────────────┴───────────────┐
        ▼                              ▼
local Python tool                 MCP adapter
                                       │
                                       ▼
                              remote JSON-RPC tool
```

The ReAct executor, deterministic executor, and guided executor therefore share the same tool boundary.

This makes remote-tool support compositional rather than a second agent engine.

---

## MCP lifecycle

```text
connection metadata persisted
          │
          ▼
HTTP/SSE initialize
          │
          ▼
tools/list
          │
          ▼
wrap each remote tool as BaseTool
          │
          ▼
register mcp__{server}__{tool}
          │
          ▼
normal ToolRegistry execution
```

Supported lifecycle operations documented by the source project include connect, list, inspect tools, refresh, ping, and disconnect.

### Trust boundary

Stdio MCP is deliberately unsupported in this deployment model because spawning arbitrary local subprocesses materially changes the host security boundary.

---

## Marketplace boundary

```text
owner Skill object
   ├── execution_trace
   ├── internal arguments
   ├── variable schema
   ├── replay mode
   └── commercial metadata
        │
        ▼
customer-facing contract
   ├── product metadata
   ├── price
   ├── customer input fields
   └── result

execution_trace / internal args do not cross this boundary
```

Information hiding is an API design decision, not merely a frontend presentation choice.

---

## Persistence model

The source project documents six main tables:

| Table | Responsibility |
|---|---|
| `users` | accounts and auth state |
| `agents` | system prompts, tool configuration, execution configuration |
| `conversations` | multi-turn execution containers |
| `messages` | user/assistant/tool history, tool call/result metadata, token/model metadata |
| `skills` | trace, variables schema, replay mode, packaging fields |
| `mcp_connections` | connection metadata, discovered tools, lifecycle state |

PostgreSQL + pgvector is used as the primary persistence layer. Redis/Celery is wired for background execution responsibilities.

---

## Security boundaries

- JWT user authentication.
- Ownership isolation for user-scoped agents, conversations, and Skills.
- Separate `X-API-Key` service execution path with constant-time comparison.
- Marketplace customer responses exclude internal traces.
- No stdio MCP subprocess execution.
- Tool result sizes and conversation memory are bounded.
- Secrets are configuration, not committed artifacts.

---

## Observability boundary

The frontend exposes execution semantics that are easy to hide in other agent systems:

- tool-call state and duration;
- deterministic vs guided mode;
- guidance-token usage;
- resolution source per value;
- MCP registration state;
- replay outcome.

This is intentional. Cost optimization that cannot be inspected is difficult to trust or tune.
