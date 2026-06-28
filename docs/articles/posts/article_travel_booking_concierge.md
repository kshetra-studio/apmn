# Travel Booking Concierge: How a Multi-Agent Workflow Handles Real Money Without Losing Human Control

## The Business Problem

Someone in your organisation needs to book business-class flights and hotel rooms for five employees attending a conference. In a manual process, a travel coordinator receives the request, searches multiple platforms, assembles options, gets manager sign-off on the cost, and confirms the booking. The whole thing takes days and involves a chain of emails that nobody can audit cleanly afterwards.

The question leadership asks when you propose automating this is not "how does the search agent work." It is: when does a human have to approve the spend, and what happens if the AI books something wrong? Those two questions contain everything that matters about risk, policy, and accountability.

The travel booking concierge workflow in APMN answers both questions visibly, before a line of code is written.

---

## The Shape of the Process

The workflow begins when a booking request arrives. A supervisor agent parses the request into structured sub-tasks: traveler count (five), cabin class (business), conference dates and location, and a budget ceiling. The supervisor never touches a booking tool directly. Its job is decomposition and delegation.

From there, the workflow splits into two parallel searches running concurrently: one agent searches business-class flights, another searches hotels near the conference venue. They run at the same time and report results back independently. Neither agent talks to the other.

When both searches complete, the supervisor re-engages. It assembles the best-fit package from the combined results and scores the match against the original request. That score drives the next decision.

If the package match confidence is high (above 0.8), the workflow moves to a spend check. If confidence is medium (0.5 to 0.8), a human is asked to clarify the constraints before the search runs again. If confidence is low (below 0.5), the workflow loops back to the supervisor to re-plan from scratch.

The spend check is a straightforward policy gate: if the total cost exceeds ten thousand dollars, a human approver must sign off before any transaction is committed. If it is within the threshold, the booking proceeds automatically.

Only one node in the entire workflow commits a real transaction: `task_book_package`. Everything before it is search, assembly, assessment, and approval. After booking, a trace is logged to Langfuse and the workflow ends.

An escape gate watches three nodes throughout: `task_search_flights`, `task_search_hotels`, and `task_book_package`. If any of them fail, time out, or return results below a 0.5 confidence floor, the workflow routes to human refinement rather than failing silently.

---

## One Level Down

The APMN diagram is not a specification that gets handed to a developer who then re-derives intent. The diagram is the code, expressed at two altitudes.

The supervisor agent (`task_supervisor_plan`) runs `gemini-2.5-flash` with a system prompt that extracts structured output: traveler count, cabin class, conference dates, conference location, and budget ceiling. Those output fields are referenced directly by downstream nodes. The flight search node reads `task_supervisor_plan.output.conference_dates`. The hotel search node reads `task_supervisor_plan.output.conference_location`. There is no translation step. The architect drew an arrow from the supervisor to the parallel split, and that arrow is the data dependency in the compiled graph.

The parallel gateway (`gw_split_search`) compiles to a LangGraph fork: both `task_search_flights` and `task_search_hotels` are dispatched simultaneously. Each is an `mcpToolTask` calling a specific MCP server and tool. The flight search calls `mcp://flight-search.kshetra.studio` with `tool: search_flights`. The hotel search calls `mcp://hotel-search.kshetra.studio` with `tool: search_hotels`. The parallel join (`gw_join_search`) compiles to a synchronisation point that waits for both results before proceeding. The architect drew two parallel lanes; the compiled graph has two concurrent nodes with a barrier.

The assembly agent (`task_assemble_package`) runs the same model and returns three fields: package summary, total cost, and a confidence score for fit-to-request. That confidence score is read by `gw_confidence` via `source: task_assemble_package.confidence`. The gate evaluates the score at runtime and routes accordingly. The architect drew three outgoing arrows from the confidence gate labelled high, medium, and low. In the compiled graph, those are three conditional edges evaluated against the same confidence float.

---

## The Design Decision That Requires Both BA and Developer

The most important design choice in this diagram is the distinction between `gw_confidence` and `gw_spend_check`, and why they are two separate gates rather than one.

`gw_confidence` is an AI judgment call. It evaluates how well the assembled package matches the original request. There is no rule that defines what a good match looks like. That is exactly the kind of fuzzy, context-dependent assessment that an AI model handles well. A human could not write a deterministic rule for it.

`gw_spend_check` is a deterministic policy check. The documentation note in the APMN source is explicit: "Standard exclusiveGateway, not confidenceGate — this is a deterministic policy check, not an AI judgment call." The condition is `total_cost > 10000`. Either the cost exceeds the threshold or it does not. An AI model adds no value here and should not be involved.

This distinction matters because these two gates have different failure modes and different accountability chains. If `gw_confidence` routes incorrectly, the consequence is a suboptimal travel package. If `gw_spend_check` routes incorrectly, the company commits to an unapproved expense. The second failure is a compliance problem, not a quality problem.

A BA reading the diagram can see that the spend threshold check is an `exclusiveGateway`, not a `confidenceGate`, and understand that this node will never produce a probabilistic output. A developer implementing the compiled graph can see that this node evaluates a hard condition against a numeric field, not a model output. Both are reading the same artifact. Neither is guessing about the intent of the other.

The `gw_escape` watching `task_search_flights`, `task_search_hotels`, and `task_book_package` with a confidence floor of 0.5 is the final safety net. If `task_book_package` fails after a transaction has been initiated, the escape gate routes to human refinement rather than leaving the booking in an unknown state. That is the developer's concern: what does partial failure look like at runtime? But the BA needs to know it exists: the system will never silently fail on a real transaction.

---

## Why This Matters

The problem with most AI workflow implementations is not the code. It is the gap between what the architect intended and what the developer built, which widens every time a Slack message substitutes for a spec update.

In this workflow, the architect's diagram and the developer's compiled graph are the same artifact expressed at different altitudes. When the policy team asks "at what point does a human approve the spend," the answer is visible in the diagram without reading Python. When the developer asks "what does the confidence gate evaluate," the answer is in the APMN source without reading a requirements document.

The `task_book_package` note says it plainly: "Only node that commits a real transaction — gated behind confidence and spend checks." Both the BA who drew it and the developer who compiled it are working from the same statement. That is what APMN is for.
