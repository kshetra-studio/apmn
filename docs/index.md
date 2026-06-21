# APMN — AI Process Model and Notation

**APMN** is an open specification for modelling AI-native business workflows. It extends BPMN 2.0 with first-class constructs for agents, RAG pipelines, confidence routing, MCP tool calls, and human-in-the-loop review — everything you need to describe a modern AI workflow in a vendor-neutral, executable way.

## Why APMN?

| Problem | APMN solution |
|---|---|
| BPMN has no concept of an AI agent | `agentTask`, `ragTask`, `memoryTask` node types |
| No standard way to express confidence-based routing | `confidenceGate` gateway |
| Human review patterns are ad-hoc | `humanInLoopTask` with timeout and escalation |
| MCP tool calls are invisible in diagrams | `mcpToolTask` with server + tool attributes |
| Process diagrams can't be compiled to real infrastructure | TwinTrack compiles APMN → Orkes Conductor, Google ADK |

## Quick Start

1. Open the [APMN Modeller](https://apmn-modeler.kshetra.studio) to design your workflow visually.
2. Export as APMN YAML.
3. Use [TwinTrack](https://bpmn2ai.kshetra.studio) to compile to Orkes Conductor JSON or Google ADK Python.

## Specification

→ [APMN v0.1 Specification](spec/apmn-v0.1.md)

## Schema

- [YAML Schema](schema/apmn-schema.md) — JSON Schema for APMN YAML files
- [BPMN Extension XSD](schema/apmn-extension.md) — XML extension for BPMN 2.0 tools

## Examples

- [Patient Preadmission](examples/patient_preadmission.md) — multi-agent clinical intake workflow

## Resources

| Resource | Link |
|---|---|
| Specification | [apmn.kshetra.studio/spec/v0.1](https://apmn.kshetra.studio/spec/v0.1) |
| GitHub | [kshetra-studio/apmn](https://github.com/kshetra-studio/apmn) |
| APMN Modeller | [apmn-modeler.kshetra.studio](https://apmn-modeler.kshetra.studio) |
| TwinTrack (compiler) | [bpmn2ai.kshetra.studio](https://bpmn2ai.kshetra.studio) |
| License | Apache 2.0 |
