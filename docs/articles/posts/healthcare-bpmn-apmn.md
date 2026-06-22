---
date: 2026-06-22
categories: [Healthcare, APMN]
---
# Your Clinical Processes Already Know Where AI Belongs. BPMN Just Can't Say It.

*How healthcare organisations can use existing process assets to build a structured, safe, and paced path to agentic AI*

---

Most healthcare AI transformation programmes start in the wrong place.

The typical conversation goes: which AI platform should we adopt? Should we pilot a chatbot? What about ambient documentation? Let's run a proof of concept and see what sticks.

Meanwhile, sitting in a process repository somewhere, is a BPMN diagram for patient preadmission. Another for prior authorisation. Another for clinical triage. Each one represents years of institutional knowledge -- every edge case handled, every compliance requirement codified, every exception path tested against real patients in real situations.

That knowledge gets ignored. Organisations reinvent from scratch. Pilots fail or stay as pilots forever. The AI roadmap remains a whiteboard exercise.

There is a better way.

---
<!-- more --> 
## What BPMN Got Right

BPMN 2.0 is genuinely brilliant at describing how healthcare organisations work. It handles sequential care steps, parallel diagnostic workstreams, decision points based on clinical criteria, human tasks where a clinician must act, service tasks that call external systems, and exception handling when things go wrong.

Every major health system has BPMN or an equivalent. Appian, IBM BPM, and Pega are common in Australian and US health systems. The diagrams exist. They work. They are trusted.

The problem is not with BPMN. The problem is that BPMN was designed for a deterministic world, and AI agents are not deterministic.

---

## Three Things BPMN Cannot Express

**Probabilistic outputs.** A radiologist reviews a scan and returns a diagnosis. That is a human task in BPMN. Replace that radiologist with an AI model and the output is not a decision -- it is a probability distribution. The model might be 94% confident of a finding, or 61%, or 38%. BPMN has no way to express "proceed if confidence exceeds threshold X, escalate to human if below". Every implementation hacks this into service task variables and exclusive gateways. It works, but it obscures the clinical logic and makes governance nearly impossible.

**AI-specific failure modes.** BPMN handles system failures well -- timeouts, HTTP errors, database unavailability. It handles them badly when the failure mode is "the AI returned a plausible but clinically incorrect output". There is no boundary event for hallucination. There is no error handler for model drift. Clinical AI systems fail in ways that deterministic systems do not, and BPMN gives you no standard vocabulary to describe how to catch those failures.

**Paced adoption by risk appetite.** Perhaps the most important limitation for health. A hospital cannot flip a switch and hand clinical decisions to AI agents. Adoption must be paced -- starting with low-risk administrative tasks, building evidence, then moving to higher-acuity clinical tasks as confidence is established. BPMN has no concept of a transition state. A task is either human or automated. There is no notation for "run AI alongside human for six months, compare outcomes, then decide".

---

## Introducing APMN

APMN -- AI Process Model and Notation -- is an open extension of BPMN 2.0 that adds the constructs healthcare AI actually needs, while remaining fully backwards compatible with every existing BPMN tool.

It uses BPMN 2.0's official extension mechanism. Your existing diagrams remain valid. APMN adds the vocabulary that BPMN is missing.

The constructs most relevant to health:

**agentTask** -- an AI model performs clinical reasoning, document analysis, or administrative processing. Specifies the model, prompt context, and output schema.

**ragTask** -- retrieve relevant clinical context before reasoning. Patient history, previous imaging, medication records, clinical guidelines. The retrieval step is a first-class citizen of the process, not an afterthought.

**confidenceGate** -- route based on AI confidence score. High confidence proceeds automatically. Medium confidence flags for clinician review. Low confidence escalates or falls back to the deterministic track. The thresholds are configurable per process and per risk appetite.

**humanInLoopTask** -- structured human review of AI output before the process continues. Includes timeout and escalation logic. This is not the same as a standard BPMN human task -- it is specifically designed for AI oversight, with the AI output visible to the reviewer alongside the original clinical data.

**escapeGate** -- automatic safety net. If the AI fails, times out, produces an output outside expected parameters, or confidence drops below a minimum floor, the escapeGate catches it and routes to a defined fallback. The fallback is always the deterministic track -- the process that was running before AI was introduced.

**modelVersionGate** -- run two model versions in parallel on a percentage of cases and compare outcomes. This is how you validate a model upgrade before committing to it across the full patient population.

---

## The TwinTrack Approach: Pace Your AI Adoption

This is the architectural principle that makes APMN practical for health.

Think of your existing clinical process as the **reliable track** -- tested, audited, APRA/TGA/JCI compliant, trusted by clinicians. It does not change. It does not stop running.

AI runs on the **innovation track** -- in parallel, processing the same cases, comparing outputs, building evidence. The two tracks are joined by lightweight orchestration ramps: on-ramps that divert selected cases to the AI track, and off-ramps that return cases to the reliable track if AI confidence drops or an escapeGate fires.

The separation is foundational, not cosmetic. AI infrastructure and deterministic infrastructure are kept distinct. They communicate only through the orchestration ramps. This means:

- A failure in the AI track cannot propagate to the reliable track
- Governance of AI decisions is isolated from governance of deterministic decisions -- reducing compliance surface area, not increasing it
- Clinicians can trust the reliable track unconditionally, which is the psychological prerequisite for AI adoption in clinical settings

As evidence accumulates and confidence thresholds are validated against real outcomes, the proportion of cases routed to the AI track increases at a pace determined by the organisation's risk appetite. Not by the vendor's roadmap.

This is paced AI adoption with a reliable fallback -- built into the notation, not bolted on as an afterthought.

---

## A Real Example: Patient Preadmission

A simplified BPMN for hospital patient preadmission:

1. Receive referral
2. Verify insurance eligibility (human task)
3. Review medical history (human task)
4. Assess clinical priority (human task)
5. Schedule appointment
6. Send confirmation

Steps 2, 3, and 4 are human tasks. Clinicians and administrators spend time on each one. At high volume, these steps create queues, delays, and variability.

The same process in APMN, with TwinTrack:

**Step 2 -- Verify insurance eligibility:**
ragTask retrieves the patient's policy documents. agentTask verifies eligibility against policy terms. confidenceGate routes: above 90% confidence proceeds automatically; 70-90% flags for administrative review; below 70% routes to humanInLoopTask. escapeGate catches any AI failure and returns to the original human task on the reliable track.

**Step 3 -- Review medical history:**
ragTask retrieves EHR records, previous imaging, medication history, and relevant clinical guidelines. agentTask summarises and flags clinical risk factors. escapeGate validates output structure and clinical coherence before it reaches the clinician. humanInLoopTask presents the AI summary to the clinician for review and sign-off -- not replacing clinical judgment, accelerating the preparation for it.

**Step 4 -- Assess clinical priority:**
agentTask applies a validated urgency scoring model. confidenceGate: high urgency scores above threshold route directly to expedited scheduling; borderline scores go to consultant review; low confidence returns to the human task on the reliable track.

Steps 1, 5, and 6 are unchanged. The process logic is the same. The compliance requirements are the same. Three human bottlenecks now have AI handling the preparation work, with human oversight calibrated to confidence level and clinical risk.

The reliable track runs in parallel throughout. If any AI component fails or confidence drops systemically, every case routes back to the original process automatically. Patients are never exposed to AI failure.

---

## What This Means for Health AI Leaders

If your organisation has existing BPMN process diagrams, you have a structured AI roadmap already written. The human tasks in those diagrams are your AI agent candidates -- ranked by volume, complexity, and clinical risk.

The path forward:

1. Identify processes with the highest volume of human tasks
2. Map human tasks to APMN candidates using the confidence and risk framework
3. Convert to APMN using TwinTrack -- the tool identifies AI-ready nodes and scores conversion confidence automatically
4. Configure confidence thresholds and fallback behaviour to match your organisation's risk appetite
5. Run the TwinTrack parallel execution model -- reliable track never stops, AI track proves itself incrementally

APMN and TwinTrack do not require you to choose an AI platform before you start. The APMN model is platform-agnostic. Once designed, it compiles to Orkes Conductor for execution -- or to other orchestration platforms as the ecosystem grows.

Your existing process knowledge. Your existing compliance framework. A structured, paced, reversible path to clinical AI -- without reinventing what already works.

---

## Resources

APMN spec v0.1 (Apache 2.0, open source): apmn.kshetra.studio/spec/apmn-v0.1

APMN visual modeller (MIT, open source): apmn-modeler.kshetra.studio

TwinTrack -- BPMN to APMN compiler, free to try: bpmn2ai.kshetra.studio

Patient preadmission worked example: apmn.kshetra.studio/examples/patient_preadmission

Questions or contributions to the APMN standard: github.com/kshetra-studio/apmn

---

*Dinesh Singh Panwar is the founder of Kshetra Studio and the creator of APMN and TwinTrack. Former Head of Technology and Chief Engineer, Westpac Group.*
