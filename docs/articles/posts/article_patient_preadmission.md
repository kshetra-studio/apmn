# Patient Preadmission: Why Compliance Has to Come Before Confidence

## The Business Problem

A private hospital's preadmission process determines whether a patient can be admitted, collects their details, verifies their insurance, generates their paperwork, and notifies the care team — all before the patient arrives. Done manually, this process involves phone calls, faxes, and a clinician reviewing a paper form before making an admission decision. It is slow, variable, and difficult to audit.

The case for automating parts of it is straightforward. The conversational intake, insurance verification, and document generation are high-volume, pattern-driven tasks that AI handles well. The clinical judgment call — whether this patient meets admission criteria — is more nuanced, but a well-prompted model with access to the right policy documents can produce a reliable first pass.

What leadership and legal need to know is not how the clinical criteria model works. It is this: does HIPAA compliance and patient consent get checked before any AI judgment happens? Not after. Not in parallel. Before.

That ordering is a legal requirement. It is also, in most implementations, the thing most likely to get quietly violated when an engineer is optimising for throughput and the compliance gate feels like a bottleneck.

The patient preadmission workflow in APMN makes that ordering visible and enforces it structurally.

---

## The Shape of the Process

The workflow begins when a patient submits a preadmission request. A conversational intake agent (`task_collect_demographics`) collects name, date of birth, Medicare number, private health fund details, contact information, and next of kin. The documentation note describes it plainly: "Conversational intake agent — replaces paper form."

From there, two steps run in sequence before any clinical judgment occurs. First, a RAG retrieval step (`task_check_policy`) pulls relevant admission policy sections from the `hospital_policy_kb` vector store based on the patient profile. Second, an MCP tool call (`task_verify_insurance`) confirms coverage and benefit limits by calling the insurance API directly.

Only after both of those steps complete does the workflow reach the compliance gate (`gw_compliance`). This is an `mcpGate` — not an AI judgment, not a confidence-weighted decision — that calls a compliance checker tool and evaluates the result deterministically. If the result is `pass`, the workflow proceeds to clinical criteria assessment. If the result is `review`, a human clinician must obtain explicit consent before anything proceeds. If the compliance check produces any other result, the workflow routes directly to `end_rejected`.

The `task_manual_consent` node has a four-hour timeout with `escalate_to: end_rejected`. A patient who cannot provide consent within four hours is not admitted. That is a policy decision, not an engineering default.

After compliance passes, the clinical criteria agent (`task_apply_criteria`) runs `gemini-2.5-flash` with a system prompt that applies the retrieved policy sections to the patient profile. It returns whether the patient is approved, a confidence score between zero and one, a reasoning string, and any risk flags.

That confidence score drives `gw_confidence`. Above 0.85, the workflow proceeds to document generation and care team notification. Between 0.6 and 0.85, a clinician must perform a clinical risk assessment before the workflow continues. Below 0.6, the admission is rejected.

After the high or medium confidence path resolves, the workflow splits into two parallel streams: generating admission documents and notifying the care team. These run concurrently. After the join, a staff member applies the patient wristband — a `manualTask`, because applying a wristband is a physical action that cannot be automated. The patient's admission context is saved to long-term memory keyed to `patient.${patient_id}.admission`. A two-hour timer waits for the patient to arrive. An observability event logs the trace to Langfuse. The workflow ends with admission complete.

An escape gate (`gw_escape`) watches three nodes: `task_apply_criteria`, `task_verify_insurance`, and `task_check_policy`. If any of them fail, time out, or return results below a 0.5 confidence floor, the workflow routes to `task_clinical_risk` — a human clinician — rather than failing or proceeding automatically.

---

## One Level Down

The compliance gate's position in the flow sequence is not a diagram convention. It is structural enforcement.

The flow definitions in the APMN source show the exact sequence: `f3` connects `task_check_policy` to `task_verify_insurance`. `f4` connects `task_verify_insurance` to `gw_compliance`. `f5` connects `gw_compliance` to `task_apply_criteria` on a `pass` result. `f8` connects `task_apply_criteria` to `gw_confidence`.

There is no flow definition connecting `task_collect_demographics`, `task_check_policy`, or `task_verify_insurance` directly to `task_apply_criteria`. In the compiled graph — whether targeting Orkes Conductor or Google ADK — there is no edge from the intake or verification steps to the clinical criteria agent that bypasses `gw_compliance`. The compliance check is not a recommended step. It is a graph topology constraint.

The `gw_compliance` node is typed as `mcpGate`, not `confidenceGate` or `exclusiveGateway`. That distinction matters. A `confidenceGate` evaluates a probability. An `exclusiveGateway` evaluates a deterministic condition. An `mcpGate` calls an external tool and routes on the result. The compliance checker at `mcp://compliance-checker.kshetra.studio` returns `pass` or `review`. The routing is deterministic on that string. No model output, no probability, no threshold to tune. The compliance decision is binary and externally authoritative.

The `task_apply_criteria` agent returns structured JSON with four fields: `approved` (bool), `confidence` (float 0-1), `reasoning` (string), and `risk_flags`. The confidence gate reads `task_apply_criteria.confidence` directly. The 0.85 threshold in the `confidenceGate` and the `confidence_threshold: 0.85` in the `agentTask` node are the same value, set once in the APMN source and enforced consistently in the compiled output. There is no version of this system where the diagram says 0.85 and the code implements 0.75.

The `task_save_memory` node writes to long-term memory with `key: "patient.${patient_id}.admission"` and `value_from: task_apply_criteria.output`. This means the clinical reasoning — including the confidence score and risk flags — is persisted and available to any future agent call for this patient. The memory task is positioned after the wristband step and before the arrival timer, ensuring that context is available before the patient's first clinical interaction after admission.

---

## The Design Decision That Requires Both BA and Developer

The placement of `gw_compliance` before `gw_confidence` is the joint decision, and it is one that could easily have been made differently by either party working alone.

A developer optimising the workflow for throughput might move the compliance check to run in parallel with insurance verification, or even run it after the clinical criteria assessment to reduce the critical path. The outcome in most cases would be identical — a compliant patient who meets clinical criteria gets admitted either way. But the ordering would no longer reflect the legal requirement: consent must be obtained before AI judgment is applied to the patient.

A BA writing requirements might state "HIPAA compliance must be verified" without specifying where in the sequence. A developer could interpret that as a checkbox at the end, or a logging requirement, rather than a structural gate that blocks all downstream AI processing.

The APMN source resolves this by making the ordering explicit and executable. The `gw_compliance` node's documentation says: "Compliance flag — clinician must obtain explicit consent before proceeding." The flow connections enforce that by making `gw_compliance` the only path from insurance verification to clinical assessment. That is a joint BA and developer decision, written once in the same artifact, enforced in the compiled graph.

The `gw_escape` watching `task_apply_criteria`, `task_verify_insurance`, and `task_check_policy` with a 0.5 confidence floor is the second joint decision. The escape routes to `task_clinical_risk` — a human clinician — rather than to `end_rejected`. The reasoning is that a failure in any of these three nodes is not a definitive rejection. It is an unknown. A clinician should decide what to do with an unknown, not an automated end state. That is a clinical governance decision that required both a BA who understood the hospital's risk policy and a developer who understood what a runtime failure looks like.

---

## Why This Matters

Healthcare AI implementations fail compliance reviews for one of two reasons: the compliance step is missing, or it is present but not verifiably ordered correctly relative to the AI judgment step.

In this workflow, the ordering is not documented in a separate spec that might drift from the implementation. It is the implementation. The flow topology in the APMN source is the structural guarantee. The diagram that a compliance officer reviews and the graph that runs in production are compiled from the same file.

When a regulator asks "how do you guarantee that patient consent is obtained before any AI assessment is applied," the answer is: the compliance gate is a required node in the graph, positioned upstream of the clinical criteria agent, with no bypass path. A patient cannot reach `task_apply_criteria` without passing through `gw_compliance`. That is verifiable from the APMN source without reading the compiled output.

The `task_apply_wristband` manualTask and the two-hour `event_wait_arrival` timer are small details, but they represent the same principle. Physical actions and real-world time are represented explicitly in the diagram, not hidden in implementation notes. A clinical coordinator reading the diagram knows exactly what the system does and does not automate. A developer implementing the compiled graph knows exactly which nodes are AI, which are human, and which are waiting for the physical world to catch up.

That shared understanding — across legal, clinical, and engineering — is what APMN is built for.
