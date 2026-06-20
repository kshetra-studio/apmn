# APMN — AI Process Model and Notation

**BPMN for the AI era.**

> BPMN gave the world a shared language for business processes.  
> APMN gives the world a shared language for *agentic* processes.

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-v0.1-green.svg)](spec/apmn-v0.1.md)
[![Status](https://img.shields.io/badge/status-draft-orange.svg)](spec/apmn-v0.1.md)
[![Creator](https://img.shields.io/badge/creator-Kshetra%20Studio-6a0dad.svg)](https://kshetra.studio)

---

## What is APMN?

**AI Process Model and Notation (APMN)** is an open, vendor-neutral, visual notation standard for designing workflows where humans, AI agents, and automated systems collaborate.

It is a **superset of BPMN 2.0** — every APMN file is a valid BPMN 2.0 XML document. APMN-specific constructs live in the `apmn:` extension namespace, meaning existing BPMN tools open APMN files without modification.

**APMN adds what BPMN was never designed for:**

| BPMN 2.0 | APN |
|---|---|
| Deterministic flow only | Probabilistic (confidence-based) routing |
| Human tasks and service calls | Agents, RAG, MCP tools, vector stores, memory |
| Binary gateways (yes/no) | Confidence gates, reasoning gates, semantic gates |
| No observability concept | Observability events baked into the notation |
| No multi-agent concept | Agent handoff as a first-class flow element |

---

## New Node Types

| Node | Type | Description |
|---|---|---|
| 🤖 `agentTask` | Task | LLM-driven task. Attrs: `model`, `system_prompt`, `tools[]`, `temperature` |
| 📚 `ragTask` | Task | Retrieval-augmented generation. Attrs: `vector_store`, `top_k`, `similarity_threshold` |
| 🔌 `mcpToolTask` | Task | Invoke an MCP server tool. Attrs: `server`, `tool_name`, `input_schema` |
| 👤 `humanInLoopTask` | Task | Agent proposes, human approves. Attrs: `confidence_threshold`, `timeout` |
| ↗ `agentHandoff` | Task | Delegate to a specialist agent. Multi-agent orchestration. |
| 🧠 `memoryTask` | Task | Read/write agent memory (session or long-term). |
| ⬡ `vectorTask` | Task | Embed content into or query a vector store. |
| 📡 `observeEvent` | Event | Emit a trace/span to an observability platform (LangFuse, Phoenix, OTEL). |

## New Gate Types

| Gate | Description |
|---|---|
| `confidenceGate` | Routes on agent confidence score (0.0–1.0). Replaces binary exclusiveGateway. |
| `reasoningGate` | Routes by parsing LLM chain-of-thought output. No hard-coded condition. |
| `modelVersionGate` | A/B or canary route between model versions. Safe model upgrade pattern. |
| `semanticGate` | Routes by vector similarity — input matched against intent clusters. |
| `mcpGate` | Calls an MCP tool and routes on result (pass/fail/review). |
| `escapeGate` | Auto-escalation. If agent fails, times out, or hits confidence floor — route to human. |

---

## Quick Example

```yaml
# APMN v0.1 — Patient Preadmission
# xmlns:apmn="http://apmn.kshetra.studio/ns/1.0"

process:
  id: patient_preadmission
  name: Patient Preadmission
  targets: [orkes, google_adk]

nodes:
  - id: verify_insurance
    type: agentTask
    model: gemini-2.0-flash
    tools: [mcp://insurance-api, mcp://medicare-lookup]
    confidence_threshold: 0.95

  - id: check_policy_docs
    type: ragTask
    vector_store: hospital_policy_kb
    top_k: 5

  - id: gw_confidence
    type: confidenceGate
    source: verify_insurance.confidence
    routes:
      high:   ">= 0.95 → proceed"
      medium: "0.7–0.95 → human_review"
      low:    "< 0.7 → escalate"

  - id: compliance_check
    type: mcpToolTask
    server: mcp://compliance-checker.kshetra.studio
    tool: check_hipaa_consent

  - id: log_trace
    type: observeEvent
    platform: langfuse
    trace_name: insurance_verification
```

---

## Reference Implementation

**[TwinTrack](https://bpmn2ai.kshetra.studio)** is the reference compiler for APMN.

```
APMN YAML / BPMN 2.0 + apmn: XML
             ↓
         TwinTrack
        /         \
Orkes JSON   Google ADK Python   (LangGraph, AWS Bedrock — coming)
```

---

## Relationship to BPMN 2.0

APMN is designed as a **superset**, not a replacement:

- APMN YAML files are the primary authoring format (human-readable, diff-friendly)
- APMN also supports BPMN 2.0 XML with `apmn:` extension elements as an alternative input
- All existing BPMN 2.0 node types (`serviceTask`, `userTask`, `parallelGateway`, etc.) are valid APMN and pass through unchanged
- APMN extension elements are ignored by BPMN tools that don't support them — full backward compatibility

---

## Spec

- [APMN Specification v0.1](spec/apmn-v0.1.md)
- [YAML Schema](schema/apmn-schema.yaml)
- [BPMN Extension Schema (XSD)](schema/apmn-extension.xsd)
- [Examples](examples/)

---

## Roadmap

| Version | Theme |
|---|---|
| v0.1 (now) | Core node types, gates, YAML schema, XSD extension |
| v0.2 | Visual shape library, BPMN.io plugin |
| v0.3 | VS Code extension, APMN Playground |
| v1.0 | Stable spec, multiple reference implementations |
| v2.0 | Submit to OMG BPMN 3.0 working group |

---

## Contributing

APMN is governed as an open community standard. See [CONTRIBUTING.md](CONTRIBUTING.md).

All contributions are welcome — spec clarifications, new node type proposals, example workflows, tooling.

---

## License

Apache License 2.0 — see [LICENSE](LICENSE).

The APN specification and all artefacts in this repository are free to use, implement, and extend without restriction.

---

*APMN is an open standards initiative by [Kshetra Studio](https://kshetra.studio).*  
*Website: [apmn.kshetra.studio](https://apmn.kshetra.studio) · GitHub: [kshetra-studio/apmn](https://github.com/kshetra-studio/apmn)*
