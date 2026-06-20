# AI Process Model and Notation — Specification v0.1 (Draft)

**Status:** Draft  
**Date:** 2026-06-20  
**Authors:** Kshetra Studio  
**Namespace:** `http://apmn.kshetra.studio/ns/1.0`  
**License:** Apache 2.0

---

## 1. Introduction

AI Process Model and Notation (APN) is an open specification for representing business processes that involve AI agents, large language models (LLMs), retrieval systems, MCP tools, and human participants.

APMN is designed as a **superset of BPMN 2.0**:
- Every APMN document is a valid BPMN 2.0 document
- APMN extensions use the `apn:` XML namespace inside `<bpmn:extensionElements>`
- APMN also defines a canonical YAML serialisation as the primary authoring format
- Existing BPMN tools open APMN files without modification (extensions are skipped gracefully)

### 1.1 Design Principles

1. **Business-readable** — a business analyst should be able to read and author APN diagrams without writing code
2. **Probabilistic-first** — routing can be based on confidence scores and LLM reasoning, not only hard-coded conditions
3. **AI-native** — agents, RAG, MCP tools, vector stores, and memory are first-class node types, not afterthoughts
4. **Vendor-neutral** — APN compiles to any target runtime (Orkes, Google ADK, LangGraph, AWS Bedrock, etc.)
5. **Observable by default** — observability events are part of the notation, not an external concern
6. **Backward compatible** — all BPMN 2.0 node types are valid APN

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

| APN Node Type | BPMN 2.0 Equivalent |
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

### 3.2 New APN Task Types

#### 3.2.1 agentTask

An LLM-driven task. The agent receives a system prompt, optional context, and a list of available tools, and produces a result.

```yaml
- id: string                    # required
  type: agentTask                # required
  name: string                  # required
  model: string                  # required — e.g. "gemini-2.0-flash", "claude-sonnet-4-6"
  system_prompt: string          # optional — overrides default instruction
  tools: [string]                # optional — MCP tool URIs or named tool references
  temperature: float             # optional, default 0.0
  max_tokens: int                # optional
  confidence_threshold: float    # optional, 0.0–1.0 — output routed to confidenceGate
  documentation: string          # optional
```

**Compiler targets:**
- Orkes → `SIMPLE` worker task
- Google ADK → `@tool` function + `LlmAgent`
- LangGraph → node function with LLM call

#### 3.2.2 ragTask

A retrieval-augmented generation task. Queries a vector store and augments a prompt with retrieved context.

```yaml
- id: string
  type: ragTask
  name: string
  vector_store: string           # required — store ID or URI
  query_from: string             # required — input field to use as query
  top_k: int                     # optional, default 5
  similarity_threshold: float    # optional, default 0.7
  model: string                  # optional — LLM for synthesis step
  documentation: string
```

#### 3.2.3 mcpToolTask

Invokes a specific tool on an MCP (Model Context Protocol) server.

```yaml
- id: string
  type: mcpToolTask
  name: string
  server: string                 # required — MCP server URI (e.g. mcp://fs, mcp://github)
  tool: string                   # required — tool name on the server
  input_schema: object           # optional — JSON Schema for tool input
  documentation: string
```

#### 3.2.4 humanInLoopTask

A task where an AI agent produces a proposal that a human must approve, reject, or modify before the flow continues.

```yaml
- id: string
  type: humanInLoopTask
  name: string
  confidence_threshold: float    # optional — if agent confidence < threshold, auto-route to human
  timeout: string                # optional — ISO 8601 duration (e.g. PT4H)
  escalate_to: string            # optional — node ID if timeout exceeded
  documentation: string
```

#### 3.2.5 agentHandoff

Delegates control to a specialist agent. Used in multi-agent orchestration.

```yaml
- id: string
  type: agentHandoff
  name: string
  target_agent: string           # required — agent ID or URI
  context_fields: [string]       # optional — which workflow fields to pass
  documentation: string
```

#### 3.2.6 memoryTask

Reads from or writes to an agent memory store (session-scoped or long-term).

```yaml
- id: string
  type: memoryTask
  name: string
  store: string                  # required — "session" | "long_term" | store URI
  operation: string              # required — "read" | "write" | "append"
  key: string                    # required — memory key
  value_from: string             # required for write — input field
  documentation: string
```

#### 3.2.7 vectorTask

Embeds content into, or queries, a vector store directly (as a standalone task, distinct from ragTask which combines retrieval + synthesis).

```yaml
- id: string
  type: vectorTask
  name: string
  store: string                  # required — vector store ID or URI
  operation: string              # required — "embed" | "query" | "delete"
  embedding_model: string        # optional — embedding model to use
  documentation: string
```

#### 3.2.8 observeEvent

Emits a trace, span, or log event to an observability platform. Does not affect flow — it is a side-effect event.

```yaml
- id: string
  type: observeEvent
  name: string
  platform: string               # required — "langfuse" | "phoenix" | "otel" | "datadog"
  trace_name: string             # optional
  span_name: string              # optional
  attributes: object             # optional — key-value pairs to attach
```

---

## 4. Gate Types

### 4.1 confidenceGate

Routes flow based on a numeric confidence score produced by an upstream `agentTask`. Replaces the binary `exclusiveGateway` for AI-native routing.

```yaml
- id: string
  type: confidenceGate
  name: string
  source: string                 # required — "taskId.confidence"
  routes:
    high:   string               # condition expression → target node ID
    medium: string
    low:    string
    default: string              # fallback if no route matches
```

### 4.2 reasoningGate

Routes by parsing the LLM's chain-of-thought output from an upstream `agentTask`. No hard-coded condition required.

```yaml
- id: string
  type: reasoningGate
  name: string
  source: string                 # required — "taskId.reasoning"
  routes:
    - match: string              # natural-language match expression
      target: string             # target node ID
  default: string
```

### 4.3 modelVersionGate

Routes a percentage of traffic to different model versions. Enables canary rollout and A/B testing of model upgrades.

```yaml
- id: string
  type: modelVersionGate
  name: string
  routes:
    - model: string
      weight: float              # 0.0–1.0, sum must equal 1.0
      target: string
```

### 4.4 semanticGate

Routes by computing semantic similarity between the input and a set of intent cluster descriptions using vector embeddings.

```yaml
- id: string
  type: semanticGate
  name: string
  embedding_model: string        # optional
  intents:
    - label: string
      description: string        # natural-language description of this route
      target: string
  default: string
```

### 4.5 mcpGate

Calls an MCP tool and routes based on the tool's return value.

```yaml
- id: string
  type: mcpGate
  name: string
  server: string
  tool: string
  routes:
    - result_match: string       # match expression on tool result
      target: string
  default: string
```

### 4.6 escapeGate

An automatic safety-net gate. If any upstream node fails, exceeds a timeout, or falls below a confidence floor, this gate activates and routes to a human task or fallback agent.

```yaml
- id: string
  type: escapeGate
  name: string
  watches: [string]              # node IDs to monitor
  triggers:
    on_failure: string           # target node ID
    on_timeout: string
    on_low_confidence: string
  confidence_floor: float        # optional, default 0.5
```

---

## 5. Sequence Flows

Sequence flows in APN follow BPMN 2.0 conventions with an optional `condition` field.

```yaml
flows:
  - id: string
    source: string               # source node ID
    target: string               # target node ID
    name: string                 # optional — label shown on diagram
    condition: string            # optional — condition expression
```

---

## 6. Swimlanes and Pools

APN uses BPMN 2.0 pools and lanes with two additional lane types:

```yaml
pools:
  - id: string
    name: string
    type: string                 # "human" | "agent" | "tool" | "system"
    lanes: [Lane]
```

New lane types:
- `agent` — contains agentTask, ragTask, mcpToolTask nodes
- `tool` — contains mcpToolTask, vectorTask nodes
- `system` — automated serviceTask, scriptTask nodes

---

## 7. Compiler Contract

Any tool claiming APMN compliance must:

1. Accept APMN YAML as defined in this spec
2. Accept BPMN 2.0 XML with `apn:` extension elements
3. Translate all BPMN 2.0 node types correctly
4. Translate all APMN v0.1 node types to at least one target runtime
5. Produce a `POST_TRANSFORM.md` listing all touchpoints requiring manual completion
6. Reject documents that reference undefined node types

---

## 8. Versioning

APN follows semantic versioning. This document describes APMN v0.1.

Breaking changes increment the minor version until v1.0, after which semver applies strictly.

The `apmn_version` field in YAML documents must match a supported spec version.

---

## Appendix A — Compiler Target Mapping (informative)

| APN Node | Orkes Conductor | Google ADK | LangGraph |
|---|---|---|---|
| agentTask | SIMPLE | LlmAgent + @tool | node fn + LLM call |
| ragTask | SIMPLE | @tool with vector retrieval | node fn |
| mcpToolTask | HTTP / SIMPLE | @tool (MCP client) | node fn |
| humanInLoopTask | HUMAN | callback tool | interrupt node |
| agentHandoff | SUB_WORKFLOW | sub-Agent | send_message |
| memoryTask | SIMPLE | @tool | node fn |
| vectorTask | SIMPLE | @tool | node fn |
| observeEvent | SIMPLE (side effect) | @tool | passthrough node |
| confidenceGate | SWITCH | Gemini routing | conditional edge |
| reasoningGate | SWITCH | Gemini routing | conditional edge |
| modelVersionGate | SWITCH | Gemini routing | conditional edge |
| semanticGate | SWITCH | Gemini routing | conditional edge |
| mcpGate | SWITCH | Gemini routing | conditional edge |
| escapeGate | SWITCH + error handler | EscapeAgent | error edge |
