# Sales Document Ingestion and PII Redaction Pipeline: Why the Order of Operations Is a Compliance Decision

## The Business Problem

Your sales team generates documents constantly: proposals, contracts, email threads, Salesforce attachments, shared drive files. Some of those documents contain customer PII — names, email addresses, phone numbers, financial account numbers, government IDs. Your organisation wants to vectorise that content for AI-powered knowledge retrieval. The business case is straightforward: sales reps should be able to search the entire document history and get relevant answers instantly.

The compliance question is equally straightforward, and it has to be answered before any vectorisation happens: how do you guarantee that PII never reaches the vector store unredacted?

This is not a question about the redaction algorithm. That is an engineering concern. The business concern is the ordering: redact first, vectorise second, always, with no path through the workflow that bypasses that sequence. If that ordering is enforced in code but invisible in the process diagram, the compliance team has no way to verify it without reading Python. If it is visible in the diagram but not actually enforced in the compiled output, the diagram is a fiction.

The sales document ingestion pipeline in APMN makes the ordering visible at the diagram level and enforces it in the compiled output from the same source.

---

## The Shape of the Process

The workflow begins when a new document is detected in a sales channel. An MCP tool call (`task_crawl_source`) pulls the raw document from wherever it lives — CRM attachment, email, shared drive. The raw document is never indexed. It goes directly to PII detection.

The PII detection step (`task_detect_pii`) runs `gemini-2.5-flash` with a system prompt that scans for PII spans and returns two things: a list of detected spans and a confidence score for detection completeness. That confidence score drives the next gate.

If detection confidence is high (above 0.9), the workflow proceeds directly to redaction. If it is medium (0.6 to 0.9), a human data governance reviewer is asked to confirm or correct the flagged spans before redaction proceeds. If it is low (below 0.6), the workflow routes to the escape gate immediately — a document with unreliable PII detection is not redacted and not filed; it raises a ServiceNow exception ticket instead.

After redaction, the workflow splits into two parallel streams: the redacted document is embedded into the `sales_knowledge_base` vector store, and simultaneously filed in the enterprise content management system with a retention policy applied. Only the redacted copy is ever embedded. The original document with PII intact is never passed to either the vector store or the ECM.

After the parallel join, a grader agent (`task_grade_quality`) verifies that the embedded chunks are coherent and the ECM filing metadata is complete. A final observability event logs the trace to Langfuse. If the grade passes, the workflow ends with a successful ingestion.

An escape gate (`gw_escape`) watches five nodes throughout: `task_detect_pii`, `task_redact_pii`, `task_vectorise`, `task_file_ecm`, and `task_grade_quality`. Any failure, timeout, or result below a 0.6 confidence floor raises a ServiceNow exception ticket via `task_raise_snow_ticket` rather than failing silently.

---

## One Level Down

The flow sequence in the APMN source is the enforcement mechanism.

The flow definitions make the ordering explicit: `f2` connects `task_crawl_source` to `task_detect_pii`. `f3` connects `task_detect_pii` to `gw_pii_confidence`. `f4`, `f5`, and `f6` connect the confidence gate to `task_redact_pii`, `task_human_review_pii`, and `gw_escape` respectively. `f8` connects `task_redact_pii` to `gw_split`. `f9` and `f10` connect the split to `task_vectorise` and `task_file_ecm`.

There is no flow definition that connects `task_crawl_source` or `task_detect_pii` directly to `task_vectorise`. In the compiled LangGraph graph, that means there is no edge from the raw document to the vector store. The graph topology itself enforces the compliance requirement. A developer cannot accidentally create a shortcut path without modifying the APMN source, which would be visible in version control.

The `task_redact_pii` node documentation states: "Produces a PII-clean copy; original is never indexed or vectorised." The `task_vectorise` node documentation states: "Retriever equivalent — only the PII-clean copy is ever embedded." Both notes are in the APMN source. Both compile to the same graph. The architect wrote a compliance requirement; the developer got a compiled graph that enforces it.

The human review step (`task_human_review_pii`) has a four-hour timeout with `escalate_to: gw_escape`. If a data governance reviewer does not respond within four hours, the document does not proceed to redaction. It raises a ServiceNow ticket. The business requirement "never vectorise a document with unreliable PII detection" is enforced by the timeout escalation path, not by relying on a human to remember to escalate.

The grader agent (`task_grade_quality`) runs at the end with a 0.85 confidence threshold and a system prompt that checks two things: chunk coherence and ECM filing metadata completeness (title, source, retention class). This is the equivalent of LangChain's grader node in the Agentic RAG pattern — a quality gate that confirms the retrieval corpus is actually usable before the workflow declares success.

---

## The Design Decision That Requires Both BA and Developer

The placement of `gw_pii_confidence` before `gw_split` is the joint decision.

A BA reading the diagram can verify the compliance requirement visually: the confidence gate is upstream of the parallel split. There is no path from document ingestion to vectorisation that bypasses it. The BA does not need to read the flow definitions to confirm this; the diagram makes it visible.

A developer implementing the compiled graph needs to understand why the confidence gate routes to `gw_escape` on low confidence rather than to a retry loop. The reason is that a document with a PII detection confidence below 0.6 is not a transient failure — it is a document the model cannot reliably assess. Retrying with the same model against the same document will not improve the outcome. The correct response is to stop, raise a ticket, and let a human decide. That is a business decision about risk tolerance, not an engineering preference.

The medium-confidence path (`task_human_review_pii` with a four-hour timeout and `escalate_to: gw_escape`) encodes a second business decision: human review is required when the model is uncertain, but human review is itself time-bounded. A reviewer who does not respond within four hours is treated the same as a model failure. The document does not proceed. This timeout value (PT4H) is not an engineering default. It is a compliance SLA. A BA needs to set it; a developer needs to implement it. The APMN source is where both parties write it down once.

---

## Why This Matters

The compliance requirement in this workflow is simple to state: redact before vectorise, always. It is surprisingly easy to violate in practice, especially when an engineer is under pressure to ship quickly and the compliance requirement lives in a Google Doc that was last updated six months ago.

In this pipeline, the compliance requirement is not documented separately from the implementation. It is the implementation. The flow topology in the APMN source file is the enforcement mechanism. The diagram that the compliance team reviews and the graph that runs in production are compiled from the same source.

When a data governance auditor asks "how do we know PII never reaches the vector store," the answer is: show them the APMN diagram. The redaction node is upstream of the vectorisation node. There is no edge connecting them directly. The escape gate catches every failure mode. The ServiceNow ticket ensures no exception is silent.

That is a verifiable answer. It does not require reading Python.
