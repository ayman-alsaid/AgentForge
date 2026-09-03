# AgentForge — Security & Trust Boundaries

AgentForge's security model is shaped by a simple principle:

> **A tool-using agent should have explicit boundaries around identity, execution authority, remote tools, and commercial information exposure.**

---

## Authentication and ownership

The source project documents:

- JWT authentication for users;
- bcrypt password hashing;
- ownership isolation for user-scoped resources;
- service-to-service execution through `X-API-Key`;
- constant-time API-key comparison.

Foreign resources are intentionally hidden rather than exposed through descriptive authorization errors.

---

## Tool execution boundary

Local and MCP tools share the same `BaseTool` contract, but that does not imply they share the same trust level.

The ToolRegistry provides a unified execution interface while transport-specific risk is handled at the integration layer.

---

## Why MCP uses HTTP/SSE only

Stdio MCP was deliberately excluded from the shared-host deployment model.

A stdio integration can require spawning a local subprocess. That changes the trust boundary from “call a remote service” to “execute a process on the host.”

AgentForge chooses HTTP/SSE JSON-RPC instead.

This reduces protocol coverage, but it also reduces host execution authority.

---

## MCP namespacing

Remote tools are registered as:

```text
mcp__{server}__{tool}
```

This prevents name collisions between independent servers exposing tools such as `search`, `read`, or `create`.

Namespacing also makes the remote origin visible to operators and logs.

---

## Marketplace information boundary

The owner-side Skill representation may contain:

- execution trace;
- internal tool arguments;
- variable schema;
- replay configuration;
- commercial metadata.

Customer-facing routes intentionally expose only the product contract and result fields needed to use the Skill.

The internal trace is not merely hidden in the UI; it is excluded from the customer response contract.

---

## Bounded execution

The ReAct loop is capped at a documented maximum number of iterations.

Other bounded behaviors documented by the project include:

- limited conversation-memory window;
- bounded page content extraction;
- bounded search result counts;
- narrowly-scoped model rescue.

These limits reduce runaway execution, cost, and oversized-context behavior.

---

## Anti-hallucination validation as a safety boundary

When Guided Replay uses an LLM to resolve one ambiguous runtime value, the model does not get authority to invent the candidate domain.

For URL resolution, the selected value must correspond to an actual value returned by the real search step.

This keeps “reasoning about which candidate is best” separate from “asserting that a candidate exists.”

---

## Secrets and configuration

The public portfolio repository contains no credentials or private production configuration.

The source project documents environment-based secret management and private production implementation.

---

## What this document does not claim

This portfolio evidence does not claim a formal penetration test, external certification, or exhaustive security audit unless such evidence is separately published.

It documents implemented trust boundaries and the security decisions present in the supplied project record.
