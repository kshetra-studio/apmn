# Appendix A — BPMN 2.0 to APMN Mapping Reference

**Status:** Draft · Non-normative appendix to [APMN v0.1](apmn-v0.1.md)
**Date:** 2026-06-22
**Canonical URL:** `https://apmn.kshetra.studio/spec/bpmn-mapping/`

---

## 1. Purpose

This appendix is the reference table for migrating an existing BPMN 2.0 process to APMN. It answers the question every architect asks first: *"I have this BPMN element — what does it become in an AI-native workflow?"*

It is **non-normative** — APMN itself doesn't require any particular BPMN element to map to any particular APMN type. This document describes the default, deterministic mapping used by [TwinTrack](https://bpmn2ai.kshetra.studio), the reference compiler for APMN, and is intended as general migration guidance for anyone hand-converting a process or building their own tooling against the spec.

Two confidence tiers appear throughout:

- **High** — the mapping is structurally unambiguous; safe to apply automatically.
- **Medium / Review** — the BPMN element is automatable but the *better-fit* APMN type depends on what the task actually does (a `serviceTask` named "Send welcome email" and one named "Calculate risk score" are structurally identical in BPMN but should become different APMN node types). These are flagged for human review during migration, never silently auto-applied.

---

## 2. Structural Mapping — BPMN Element → APMN Type

| BPMN 2.0 Element | Default APMN Type | Confidence | Rationale |
|---|---|---|---|
| `startEvent` | `startEvent` | High | Pass-through, unchanged |
| `endEvent` | `endEvent` | High | Pass-through, unchanged |
| `userTask` | `humanInLoopTask` | High | Explicit human action in BPMN — maps directly to APMN's human-in-the-loop primitive |
| `manualTask` | `humanInLoopTask` | High | Physical/manual action — same human-in-the-loop primitive, `automatable: false` |
| `serviceTask` | `agentTask` | High | An automated service call is the closest BPMN analog to an LLM-executable task — but see §3, the task *name* often suggests a more specific type (`ragTask`, `mcpToolTask`) |
| `scriptTask` | `mcpToolTask` | Medium | Custom code is best represented as a tool call via an MCP server wrapping that script, rather than re-implemented as an LLM prompt |
| `businessRuleTask` | `agentTask` or `mcpToolTask` | Medium | An LLM can evaluate simple rules directly (`agentTask`); a DMN/rules-engine-backed task is better wrapped as an MCP tool call (`mcpToolTask`) — depends on rule complexity |
| `sendTask` | `mcpToolTask` | Medium | External message dispatch (email, SMS, webhook) — modeled as an MCP tool call to the relevant channel |
| `receiveTask` | `agentTask` | Medium | Poll/await pattern — usually becomes an agent task that waits on or processes an inbound payload |
| `exclusiveGateway` (XOR) | `exclusiveGateway` (kept) or upgraded gate | High (kept) / Medium (upgrade) | Structurally valid as-is in APMN; see §4 for when to upgrade to a probabilistic gate |
| `parallelGateway` (AND) | `parallelGateway` | High | Unchanged — APMN preserves deterministic parallel split/join |
| `inclusiveGateway` (OR) | `exclusiveGateway` | Medium | APMN v0.1 has no inclusive-gate equivalent yet; falls back to exclusive with a review flag |
| `complexGateway` | `exclusiveGateway` | Review | Always flagged — complex conditions need a human to confirm intended routing logic before any automated suggestion is safe |
| `timerEvent` | `timerEvent` | High | Unchanged |
| `intermediateCatchEvent` | `timerEvent` | Medium | Common case (catching a timer/message); other catch-event subtypes need case-by-case review |
| `intermediateThrowEvent` | `observeEvent` | Medium | Throw events are usually a side-effect/signal — modeled as an observability emission unless evidence suggests otherwise |

---

## 3. Task-Naming Heuristics

The single biggest improvement over a pure structural mapping is reading the BPMN task's **name**, not just its element type. Two `serviceTask` elements with different names should usually become different APMN types:

| Task name suggests… | Recommended APMN type | Example BPMN task names |
|---|---|---|
| Retrieval from a knowledge base or external bureau | `ragTask` | "Pull credit bureau report", "Policy lookup" |
| A call to an external system, API, or compliance check | `mcpToolTask` | "AML screening", "Update CRM", "Send notification" |
| An explicit human decision or sign-off | `humanInLoopTask` | "Underwriter review", "Customer signs offer" |
| Delegation to a specialist process/agent | `agentHandoff` | "Transfer to fraud specialist" |
| Logging, auditing, or monitoring | `observeEvent` | "Log decision", "Emit audit trail" |
| Embedding or vector-store operations | `vectorTask` | "Index document", "Generate embedding" |
| Reading or writing session/long-term context | `memoryTask` | "Recall customer preferences" |
| Scoring, calculation, document generation, financial processing | `agentTask` | "Calculate risk score", "Generate offer letter", "Disburse funds" |

This is necessarily **judgment-based, not exhaustive** — natural-language task names vary too much across organizations to fully enumerate. Treat this table as a starting heuristic, not a complete decision procedure. Tooling that automates this step (including TwinTrack) should always surface its reasoning and confidence rather than silently rewriting the diagram.

---

## 4. Gateway Upgrade Guidance

BPMN's `exclusiveGateway` is binary by construction — every outgoing path needs a hard-coded condition. APMN adds six probabilistic/semantic gate types specifically so that AI-driven routing doesn't have to be forced into binary conditions. An XOR gateway is a good candidate for upgrade when:

| Signal | Suggested upgrade | Gate type |
|---|---|---|
| Immediately downstream of an `agentTask` or other automated node | Route on the upstream task's confidence score | `confidenceGate` |
| Outgoing conditions reference classification, intent, or category | Route on LLM reasoning output rather than a hard-coded value | `reasoningGate` |
| Outgoing conditions reference model/version rollout | A/B or canary routing between model versions | `modelVersionGate` |
| Outgoing conditions reference similarity or intent clusters | Route by vector similarity | `semanticGate` |
| Outgoing conditions check the result of a tool/API call | Route on MCP tool result (pass/fail/review) | `mcpGate` |
| Gateway sits on a failure, timeout, or error path | Auto-escalate to a human safety net | `escapeGate` |

A gateway that matches none of these signals is left as a standard `exclusiveGateway` — APMN does not require every gate to be upgraded, and most processes will still have plenty of genuinely deterministic branches (e.g. "amount > $10,000 → secondary approval" is a perfectly good `exclusiveGateway` and shouldn't be forced into a `confidenceGate`).

---

## 5. What This Appendix Does Not Cover

This is a **mapping reference**, not a migration tool. It deliberately stops short of:

- The exact heuristics, keyword sets, or scoring weights used by any specific implementation (these vary by tool and are where reference compilers like TwinTrack differentiate)
- AI-assisted (Layer 2) enhancement — using an LLM to re-evaluate ambiguous mappings, accept/reject suggestions, or learn organization-specific naming conventions
- Code generation — compiling the resulting APMN document to a target runtime (Orkes Conductor, Google ADK, LangGraph, etc.)

For an automated, end-to-end implementation of this mapping — including the AI-enhancement layer and compilation to Orkes/Google ADK — see [TwinTrack](https://bpmn2ai.kshetra.studio).

---

*This appendix is part of the [APMN specification](apmn-v0.1.md), maintained by [Kshetra Studio](https://kshetra.studio). Contributions and corrections welcome via [GitHub](https://github.com/kshetra-studio/apmn).*
