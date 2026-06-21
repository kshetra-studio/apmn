# AI Process Model and Notation — Specification v0.1 (Draft)

**Status:** Draft  
**Date:** 2026-06-21  
**Authors:** Kshetra Studio  
**Namespace:** `http://apmn.kshetra.studio/ns/1.0`  
**License:** Apache 2.0  
**Canonical URL:** `https://apmn.kshetra.studio/spec/v0.1` *(GitHub source: this file)*

---

## 1. Introduction

AI Process Model and Notation (APMN) is an open specification for representing business processes that involve AI agents, large language models (LLMs), retrieval systems, MCP tools, and human participants.

APMN is designed as a **superset of BPMN 2.0**:
- Every APMN document is a valid BPMN 2.0 document
- APMN extensions use the `apmn:` XML namespace inside `<bpmn:extensionElements>`
- APMN also defines a canonical YAML serialisation as the primary authoring format
- Existing BPMN tools open APMN files without modification (extensions are skipped gracefully)

### 1.1 Design Principles

1. **Business-readable** — a business analyst should be able to read and author APMN diagrams without writing code
2. **Probabilistic-first** — routing can be based on confidence scores and LLM reasoning, not only hard-coded conditions
3. **AI-native** — agents, RAG, MCP tools, vector stores, and memory are first-class node types, not afterthoughts
4. **Vendor-neutral** — APMN compiles to any target runtime (Orkes Conductor, Google Agent Development Kit, LangGraph, AWS Bedrock, Temporal, etc.)
5. **Observable by default** — observability events are part of the notation, not an external concern
6. **Backward compatible** — all BPMN 2.0 node types are valid APMN

### 1.2 Resources

| Resource | URL |
|---|---|
| Spec (this document) | `https://apmn.kshetra.studio/spec/v0.1` |
| JSON Schema | `https://apmn.kshetra.studio/schema/v0.1/apmn-schema.yaml` |
| GitHub | `https://github.com/kshetra-studio/apmn` |
| Reference compiler | [TwinTrack](https://bpmn2ai.kshetra.studio) — BPMN → APMN → target |
| APMN Modeller | [apmn-modeler.kshetra.studio](https://apmn-modeler.kshetra.studio) |

---

## 2. Serialisation Formats

### 2.1 YAML (primary)

APMN YAML is the recommended authoring format. It is human-readable, diff-friendly, and version-control friendly.

```yaml
# APMN YAML document
apmn_version: "0.1"
xmlns_apmn: "http://apmn.kshetra.studio/ns/1.0"

process:
  id: string           # required, unique process identifier
  name: string         # required, human-readable name
  description: string  # optional
  targets: [string]    # optional — compiler targets (orkes, google_adk, langraph, bedrock)

nodes: [Node]          # ordered list of all nodes
flows: [Flow]          # sequence flows connecting nodes
```

Machine-readable schema: [`schema/apmn-schema.yaml`](../schema/apmn-schema.yaml) (JSON Schema 2020-12 expressed in YAML).

### 2.2 BPMN 2.0 XML with APMN extensions (secondary)

```xml
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
                  xmlns:apmn="http://apmn.kshetra.studio/ns/1.0">
  <bpmn:process id="my_process">
    <bpmn:serviceTask id="task_1" name="Verify Insurance">
      <bpmn:extensionElements>
        <apmn:agentTask model="gemini-2.0-flash"
                       confidence_threshold="0.95"/>
      </bpmn:extensionElements>
    </bpmn:serviceTask>
  </bpmn:process>
</bpmn:definitions>
```

---

## 3. Node Types

### 3.1 Inherited from BPMN 2.0 (unchanged semantics)

| APMN Node Type | BPMN 2.0 Equivalent |
|---|---|
| `serviceTask` | `bpmn:serviceTask` |
| `scriptTask` | `bpmn:scriptTask` |
| `userTask` | `bpmn:userTask` |
| `manualTask` | `bpmn:manualTask` |
| `businessRuleTask` | `bpmn:businessRuleTask` |
| `sendTask` | `bpmn:sendTask` |
| `receiveTask` | `bpmn:receiveTask` |
| `exclusiveGateway` | `bpmn:exclusiveGateway` |
| `parallelGateway` | `bpmn:parallelGateway` |
| `startEvent` | `bpmn:startEvent` |
| `endEvent` | `bpmn:endEvent` |
| `timerEvent` | `bpmn:intermediateCatchEvent` + timerEventDefinition |

### 3.2 New APMN Task Types

> **Implementation notes** in each section describe what a compiler **must generate** and what **post-transform work** the implementer must complete. Compiler support status for TwinTrack v0.1 is noted where applicable.

---

#### 3.2.1 `agentTask`

An LLM-driven task. The agent receives a system prompt, optional context, and a list of available tools, and produces a result.

```yaml
- id: string                    # required
  type: agentTask               # required
  name: string                  # required
  model: string                 # required — e.g. "gemini-2.0-flash", "claude-sonnet-4-6"
  system_prompt: string         # optional — overrides default instruction
  tools: [string]               # optional — MCP tool URIs or named tool references
  temperature: float            # optional, default 0.0
  max_tokens: int               # optional
  confidence_threshold: float   # optional, 0.0–1.0 — output routed to confidenceGate
  documentation: string         # optional
```

**Compiler targets — implementation notes:**

| Target | What the compiler generates | What you must complete |
|---|---|---|
| **Orkes Conductor** | A `SIMPLE` task definition JSON with `taskType: SIMPLE` and `inputParameters`. | Deploy a **Conductor Worker** — an external process (any language) that polls `GET /api/v1/tasks/{taskName}` on your Orkes instance, calls your LLM, and POSTs the result to `POST /api/v1/tasks`. TwinTrack generates the task definition only; the worker is your code. |
| **Google ADK** | An `LlmAgent` instance configured with the specified model and a `Runner`. | Supply `GOOGLE_API_KEY` (Gemini API) or set `GOOGLE_CLOUD_PROJECT` + `GOOGLE_CLOUD_LOCATION` for Vertex AI. The generated agent is a standalone Python module you run locally or deploy to Agent Engine. |
| **LangGraph** | A node function with an LLM call stub. | Wire the node into your `StateGraph` and supply the model client. |

---

#### 3.2.2 `ragTask`

A retrieval-augmented generation task. Queries a vector store and augments a prompt with retrieved context before an LLM call.

```yaml
- id: string
  type: ragTask
  name: string
  vector_store: string          # required — store ID or URI
  query_from: string            # required — input field to use as query
  top_k: int                    # optional, default 5
  similarity_threshold: float   # optional, default 0.7
  model: string                 # optional — LLM for synthesis step
  documentation: string
```

**Compiler targets — implementation notes:**

| Target | What the compiler generates | What you must complete |
|---|---|---|
| **Orkes Conductor** | A `SIMPLE` task stub. | Implement the worker: query your vector store (Pinecone, Weaviate, pgvector, etc.) using `query_from` as the query, inject retrieved context into the LLM prompt, POST result back to Conductor. |
| **Google ADK** | An `LlmAgent` with a retrieval tool stub. | Wire your vector store client into the retrieval tool. Google provides [Vertex AI Search](https://cloud.google.com/generative-ai-app-builder) as a managed option; for external stores implement a `FunctionTool` wrapper. |

---

#### 3.2.3 `mcpToolTask`

Invokes a specific tool on an MCP (Model Context Protocol) server.

```yaml
- id: string
  type: mcpToolTask
  name: string
  server: string                # required — MCP server URI (e.g. "https://mcp.example.com")
  tool: string                  # required — tool name on the server
  input_schema: object          # optional — JSON Schema for tool input
  documentation: string
```

**Compiler targets — implementation notes:**

| Target | What the compiler generates | What you must complete |
|---|---|---|
| **Orkes Conductor** | An `HTTP` task pointing to the MCP server's tool endpoint (`POST <server>/tools/<tool>`). | Ensure the MCP server is reachable from your Orkes workers. Supply any auth headers via Orkes task configuration. |
| **Google ADK** | An `MCPToolset` integration stub referencing the server URL. | The MCP server must be running and accessible. Provide credentials if the server requires auth. |

---

#### 3.2.4 `humanInLoopTask`

A task where an AI agent produces a proposal that a human must approve, reject, or modify before the flow continues.

```yaml
- id: string
  type: humanInLoopTask
  name: string
  confidence_threshold: float   # optional — auto-route to human if agent confidence < threshold
  timeout: string               # optional — ISO 8601 duration (e.g. "PT48H")
  escalate_to: string           # optional — node ID if timeout exceeded
  documentation: string
```

**Compiler targets — implementation notes:**

| Target | What the compiler generates | What you must complete |
|---|---|---|
| **Orkes Conductor** | A `HUMAN` task definition. Orkes pauses the workflow at this task and waits. | Build the human-facing UI (web app, email link, Slack bot, etc.) that calls `POST /api/v1/tasks` with the task outcome when the human responds. Without this, the workflow will pause indefinitely. If `timeout` is set, pair with an `escapeGate` to handle the timed-out path. |
| **Google ADK** | A `before_tool_callback` stub that yields control. | Implement the callback to surface the agent's proposal to a human and await their input. The mechanism is application-specific (HTTP callback, queue, etc.). |

> ⚠️ **Best practice:** Always set `timeout` and connect an `escapeGate` to handle the timeout path. An unattended `humanInLoopTask` with no timeout is a workflow deadlock risk.

---

#### 3.2.5 `agentHandoff`

Delegates control to a specialist sub-agent. Used in multi-agent orchestration.

```yaml
- id: string
  type: agentHandoff
  name: string
  target_agent: string          # required — agent ID or URI
  context_fields: [string]      # optional — which workflow fields to pass to the sub-agent
  documentation: string
```

**Compiler targets — implementation notes:**

| Target | What the compiler generates | What you must complete |
|---|---|---|
| **Orkes Conductor** | A `SUB_WORKFLOW` task referencing `target_agent` as the sub-workflow name. | The sub-workflow must be registered on your Orkes instance separately. Supply input variable mappings. |
| **Google ADK** | A sub-`LlmAgent` call stub inside a `SequentialAgent`. | Define the sub-agent and wire it into the pipeline. |

---

#### 3.2.6 `memoryTask`

Reads from or writes to an agent memory store (session-scoped or long-term).

```yaml
- id: string
  type: memoryTask
  name: string
  store: string                 # required — "session" | "long_term" | store URI
  operation: string             # required — "read" | "write" | "append"
  key: string                   # required — memory key
  value_from: string            # required for write — input field
  documentation: string
```

**Compiler targets — implementation notes:**

| Target | What the compiler generates | What you must complete |
|---|---|---|
| **Orkes Conductor** | A `SIMPLE` task stub. | Implement the worker with your memory backend (Redis, DynamoDB, a vector store, etc.). |
| **Google ADK** | A `FunctionTool` stub with read/write scaffolding. | Wire your persistence layer (Firestore, Memorystore, etc.) into the generated function. |

> **Note:** TwinTrack v0.1 generates stubs for `memoryTask`. Full implementation is the implementer's responsibility.

---

#### 3.2.7 `vectorTask`

Embeds content into, or queries, a vector store directly — as a standalone task, distinct from `ragTask` which combines retrieval with LLM synthesis.

```yaml
- id: string
  type: vectorTask
  name: string
  store: string                 # required — vector store ID or URI
  operation: string             # required — "embed" | "query" | "delete"
  embedding_model: string       # optional — embedding model to use
  documentation: string
```

**Compiler targets — implementation notes:**

| Target | What the compiler generates | What you must complete |
|---|---|---|
| **Orkes Conductor** | A `SIMPLE` task stub. | Implement the worker: call your embedding API (OpenAI, Vertex AI, etc.) and upsert/query your vector store. |
| **Google ADK** | A `FunctionTool` stub. | Wire the Vertex AI Embeddings API or your chosen embedding service. |

> **Note:** TwinTrack v0.1 generates stubs for `vectorTask`. Full implementation is the implementer's responsibility.

---

#### 3.2.8 `observeEvent`

Emits a trace, span, or log event to an observability platform. Does not block the flow — it is a fire-and-forget side-effect.

```yaml
- id: string
  type: observeEvent
  name: string
  platform: string              # required — "langfuse" | "phoenix" | "otel" | "datadog"
  trace_name: string            # optional
  span_name: string             # optional
  attributes: object            # optional — key-value pairs to attach
```

**Compiler targets — implementation notes:**

| Target | What the compiler generates | What you must complete |
|---|---|---|
| **Orkes Conductor** | A `SIMPLE` task stub (async, fire-and-forget recommended). | Implement the worker to POST to your observability platform. For LangFuse: `POST /api/v1/trace`. For OTEL: use the OTLP exporter. |
| **Google ADK** | A `FunctionTool` stub that logs to Cloud Logging by default. | Replace with your preferred platform's SDK call if not using Cloud Logging. |

---

## 4. Gate Types

> Gates are routing constructs. Unlike BPMN gateways, APMN gates carry AI-specific routing semantics (confidence, reasoning, semantic similarity). All gates compile to conditional branching constructs in target runtimes.

---

### 4.1 `confidenceGate`

Routes flow based on a numeric confidence score (0.0–1.0) produced by an upstream `agentTask`. Replaces the binary `exclusiveGateway` for AI-native routing.

```yaml
- id: string
  type: confidenceGate
  name: string
  source: string                # required — "taskId.confidence"
  routes:
    high:    string             # target node ID for confidence ≥ high_threshold
    medium:  string             # target node ID for confidence in [low_threshold, high_threshold)
    low:     string             # target node ID for confidence < low_threshold
    default: string             # fallback if source field is absent
  high_threshold: float         # optional, default 0.85
  low_threshold:  float         # optional, default 0.60
```

**Compiler targets:**

| Target | Generated construct | Notes |
|---|---|---|
| **Orkes Conductor** | `SWITCH` task evaluating `${taskId.output.confidence}` | TwinTrack v0.1 generates a basic SWITCH; confidence expression wiring is a post-transform step. |
| **Google ADK** | Conditional branch on agent output field | Wire `confidence` output from upstream `LlmAgent`. |

> ⚠️ **Best practice:** Always route the `low` path to a `humanInLoopTask`, never directly to an `endEvent`.

---

### 4.2 `reasoningGate`

Routes by parsing the LLM's chain-of-thought or structured output from an upstream `agentTask`. No hard-coded numeric condition required.

```yaml
- id: string
  type: reasoningGate
  name: string
  source: string                # required — "taskId.reasoning" or "taskId.category"
  routes:
    - match: string             # value to match against source field
      target: string            # target node ID
  default: string
```

**Compiler targets:**

| Target | Generated construct | Notes |
|---|---|---|
| **Orkes Conductor** | `SWITCH` task on parsed output field | Requires `agentTask` to produce structured output (JSON with `category` or `intent` field). TwinTrack v0.1 generates a basic SWITCH stub. |
| **Google ADK** | Conditional on `LlmAgent` output schema field | Use `output_schema` on the upstream `agentTask` to enforce a structured response. |

---

### 4.3 `modelVersionGate`

Routes a percentage of traffic to different model versions. Enables canary rollout and A/B testing of model upgrades without redeploying the workflow.

```yaml
- id: string
  type: modelVersionGate
  name: string
  routes:
    - model: string
      weight: float             # 0.0–1.0, weights must sum to 1.0
      target: string
```

**Compiler targets:**

| Target | Generated construct | Notes |
|---|---|---|
| **Orkes Conductor** | `SWITCH` on a random-seed variable injected at workflow start | TwinTrack v0.1 generates a basic SWITCH stub. Full weighted routing requires a pre-task that sets a `model_version` variable. |
| **Google ADK** | Conditional on a version selector input | Inject `model_version` as a workflow input parameter. |

---

### 4.4 `semanticGate`

Routes by computing semantic similarity between the process input and a set of intent cluster descriptions using vector embeddings. No keyword rules — intent is inferred.

```yaml
- id: string
  type: semanticGate
  name: string
  embedding_model: string       # optional, default "text-embedding-004"
  intents:
    - label: string
      description: string       # natural-language description of this route
      target: string
  default: string
```

**Compiler targets:**

| Target | Generated construct | Notes |
|---|---|---|
| **Orkes Conductor** | `SWITCH` on a cluster label from a preceding `vectorTask` or `ragTask` | TwinTrack v0.1 generates a basic SWITCH stub. Wire the cluster label from your embedding step. |
| **Google ADK** | Conditional on a semantic classifier tool result | Implement a `FunctionTool` that embeds the input and returns the closest intent label. |

> **Note:** TwinTrack v0.1 generates stubs for `semanticGate`. The embedding call is the implementer's responsibility.

---

### 4.5 `mcpGate`

Calls an MCP tool and routes based on the tool's return value. Commonly used for compliance checks and external validation steps.

```yaml
- id: string
  type: mcpGate
  name: string
  server: string
  tool: string
  routes:
    - result_match: string      # value to match against tool result
      target: string
  default: string
```

**Compiler targets:**

| Target | Generated construct | Notes |
|---|---|---|
| **Orkes Conductor** | `HTTP` task + `SWITCH` on the tool result field | Wire `result_match` against the HTTP response body field. TwinTrack v0.1 generates a basic SWITCH stub. |
| **Google ADK** | `MCPToolset` call + conditional on result | Ensure the MCP server is accessible from your ADK runtime. |

---

### 4.6 `escapeGate`

An automatic safety-net gate. Activates if any upstream node fails, exceeds a timeout, or falls below a confidence floor. Routes to a human task or fallback agent.

```yaml
- id: string
  type: escapeGate
  name: string
  watches: [string]             # node IDs to monitor
  triggers:
    on_failure:        string   # target node ID
    on_timeout:        string
    on_low_confidence: string
  confidence_floor: float       # optional, default 0.5
```

**Compiler targets:**

| Target | Generated construct | Notes |
|---|---|---|
| **Orkes Conductor** | `DO_WHILE` with error handler + `SWITCH` on condition | TwinTrack v0.1 generates a `SWITCH` stub. Full `DO_WHILE` retry + error handler pattern is a post-transform step. Pair with Orkes task-level `retryCount` and `timeoutSeconds`. |
| **Google ADK** | Error callback handler on the watched agent | Implement `on_error` / `after_agent_callback` in the generated `LlmAgent`. |

> ⚠️ **Spec constraint:** An unconnected `escapeGate` (no outgoing flows) is a spec violation. The `on_failure` path must always route to a `humanInLoopTask` or another agent — never directly to an `endEvent`.

---

## 5. Sequence Flows

Sequence flows follow BPMN 2.0 conventions with an optional `condition` field.

```yaml
flows:
  - id: string
    source: string              # source node ID
    target: string              # target node ID
    name: string                # optional — label shown on diagram
    condition: string           # optional — condition expression
```

---

## 6. Swimlanes and Pools

APMN uses BPMN 2.0 pools and lanes with two additional lane types:

```yaml
pools:
  - id: string
    name: string
    type: string                # "human" | "agent" | "tool" | "system"
    lanes: [Lane]
```

New lane types:
- `agent` — contains `agentTask`, `ragTask`, `mcpToolTask` nodes
- `tool` — contains `mcpToolTask`, `vectorTask` nodes
- `system` — automated `serviceTask`, `scriptTask` nodes

---

## 7. Compiler Contract

Any tool claiming APMN compliance must:

1. Accept APMN YAML as defined in this spec
2. Accept BPMN 2.0 XML with `apmn:` extension elements
3. Translate all BPMN 2.0 node types correctly
4. Translate all APMN v0.1 node types to at least one target runtime
5. Produce a `POST_TRANSFORM.md` listing all touchpoints requiring manual completion
6. Reject documents that reference undefined node types

---

## 8. Versioning

APMN follows semantic versioning. This document describes APMN v0.1.

Breaking changes increment the minor version until v1.0, after which semver applies strictly.

The `apmn_version` field in YAML documents must match a supported spec version.

---

## Appendix A — Compiler Target Mapping

### A.1 TwinTrack v0.1 Support Status

> ✅ Full — generates ready-to-deploy output (worker implementation still required per Section 3)  
> ⚠️ Stub — generates scaffolding with TODO markers; logic must be completed post-transform  
> ❌ Not yet — not implemented in TwinTrack v0.1

| APMN Node | Orkes Conductor | Google ADK | Notes |
|---|---|---|---|
| `agentTask` | ✅ `SIMPLE` task | ✅ `LlmAgent` | Worker/agent implementation required |
| `ragTask` | ⚠️ `SIMPLE` stub | ⚠️ `LlmAgent` + retrieval stub | Vector store wiring required |
| `mcpToolTask` | ✅ `HTTP` task | ⚠️ `MCPToolset` stub | Server URL + auth required |
| `humanInLoopTask` | ✅ `HUMAN` task | ⚠️ Callback stub | Human UI / callback handler required |
| `agentHandoff` | ⚠️ `SUB_WORKFLOW` stub | ⚠️ Sub-agent stub | Sub-agent/sub-workflow must be registered separately |
| `memoryTask` | ⚠️ `SIMPLE` stub | ⚠️ `FunctionTool` stub | Persistence backend required |
| `vectorTask` | ⚠️ `SIMPLE` stub | ⚠️ `FunctionTool` stub | Embedding service required |
| `observeEvent` | ⚠️ `SIMPLE` stub | ⚠️ `FunctionTool` stub | Observability platform client required |
| `confidenceGate` | ⚠️ `SWITCH` (basic) | ✅ Conditional | Confidence expression wiring is post-transform for Orkes |
| `reasoningGate` | ⚠️ `SWITCH` (basic) | ✅ Conditional | Structured output from upstream agent required |
| `modelVersionGate` | ⚠️ `SWITCH` (basic) | ⚠️ Conditional stub | Version selector injection required |
| `semanticGate` | ⚠️ `SWITCH` (basic) | ⚠️ Conditional stub | Embedding step required |
| `mcpGate` | ⚠️ `HTTP` + `SWITCH` stub | ⚠️ `MCPToolset` + conditional stub | MCP server required |
| `escapeGate` | ⚠️ `SWITCH` stub | ⚠️ Error callback stub | `DO_WHILE` + error handler wiring is post-transform for Orkes |
| `parallelGateway` | ✅ `FORK_JOIN` | ✅ `ParallelAgent` | Fully supported |
| `exclusiveGateway` | ✅ `SWITCH` | ✅ Conditional | Standard BPMN gateway |
| `timerEvent` | ✅ `WAIT` task | ⚠️ Stub | Duration wiring required |

### A.2 Runtime Mapping Reference

| APMN Node | Orkes Conductor | Google ADK (v2.x) | LangGraph |
|---|---|---|---|
| `agentTask` | `SIMPLE` worker task | `LlmAgent` | node fn + LLM call |
| `ragTask` | `SIMPLE` worker task | `LlmAgent` + retrieval `FunctionTool` | node fn |
| `mcpToolTask` | `HTTP` task | `MCPToolset` | node fn |
| `humanInLoopTask` | `HUMAN` task | `before_tool_callback` / interrupt | interrupt node |
| `agentHandoff` | `SUB_WORKFLOW` | Sub-`LlmAgent` in `SequentialAgent` | `send_message` |
| `memoryTask` | `SIMPLE` worker task | `FunctionTool` (read/write) | node fn |
| `vectorTask` | `SIMPLE` worker task | `FunctionTool` (embed/query) | node fn |
| `observeEvent` | `SIMPLE` (async) | `FunctionTool` / Cloud Logging | passthrough node |
| `confidenceGate` | `SWITCH` on confidence field | Conditional on `LlmAgent` output | conditional edge |
| `reasoningGate` | `SWITCH` on structured output field | Conditional on output schema field | conditional edge |
| `modelVersionGate` | `SWITCH` on version variable | Conditional on version input | conditional edge |
| `semanticGate` | `SWITCH` on cluster label | Conditional on classifier result | conditional edge |
| `mcpGate` | `HTTP` + `SWITCH` on result | `MCPToolset` call + conditional | conditional edge |
| `escapeGate` | `DO_WHILE` + `SWITCH` on error/timeout | `on_error` / `after_agent_callback` | error edge |
| `parallelGateway` | `FORK_JOIN` | `ParallelAgent` | parallel edges |

> **Google ADK note:** The `@tool` decorator was removed in google-adk ≥ 2.0. All tool definitions use `FunctionTool` or are passed as plain Python callables to `LlmAgent(tools=[...])`. `MCPToolset` is the standard integration for MCP servers. See [Google ADK documentation](https://google.github.io/adk-docs/) for current API.
