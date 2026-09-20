# Claude Certified Architect – Foundations (CCAR-F)
## Domain 5: Context Management & Reliability — Study Guide

**Domain weight:** 15% of scored items (~9 of 60)
**Appears in scenarios:** Customer Support Resolution Agent, Code Generation with Claude Code, Multi-Agent Research System, Structured Data Extraction (4 of 6 scenarios)

**Two principles run through every correct answer in this domain:**

1. Critical information must be made **structurally persistent**, not entrusted to the model's memory, summarization, or confidence.
2. **Degrade gracefully with annotated partial results.** Silent suppression and total termination are both anti-patterns.

---

# 5.1 — Preserving context across long interactions

## Core Concept

Long sessions lose fidelity in three distinct ways, and the exam tests whether you can name the right one:

- **Progressive summarization loss.** Condensing history turns precise values (amounts, percentages, dates, order numbers, customer-stated expectations) into vague prose. Exact figures are exactly what a refund or extraction decision depends on.
- **Lost-in-the-middle.** Models process the beginning and end of long inputs reliably and may omit findings from middle sections.
- **Tool-result bloat.** Raw tool payloads consume tokens disproportionately to relevance (40+ fields returned when 5 matter).

The remedy in each case is structural, not exhortative.

## Canonical Skeleton — "Case Facts Layer"

```
Tool returns raw payload
  → TRIM (PostToolUse hook / wrapper): keep only task-relevant fields
  → EXTRACT: transactional facts into a structured block
       { order_id, amount, date, status, customer_stated_expectation }
  → PERSIST: re-inject the facts block verbatim in every prompt,
       OUTSIDE the summarized conversation history
  → SUMMARIZE: narrative history may compact freely; facts are safe
  → ORDER: key findings summary FIRST, detailed results under explicit
       section headers, the actual request LAST
```

For multi-agent handoffs, the same idea applies upstream: when a downstream agent has a limited context budget, modify the **upstream** agent to return structured key facts, citations, and relevance scores instead of verbose content and reasoning chains.

## Exam Traps

- "Move to a model with a larger context window." Larger windows do not fix attention quality or summarization loss. This distractor appears in several forms across the exam.
- "Instruct the model to preserve all numerical details / pay attention to the middle." Probabilistic fix for a structural problem.
- "Summarize more aggressively." Worsens the stated symptom.
- Sending only a summary in the next API request. You pass **complete conversation history** plus the facts block.
- Assuming subagents inherit parent context. They do not; context must be explicitly placed in the prompt.
- Trimming tool output *after* it has accumulated. Trim at ingest.

## Key Facts to Memorize

- Case facts block lives **outside** summarized history and is included in **each** prompt.
- Position remedy: key findings at the beginning, explicit section headers for detail.
- Subagent structured outputs must carry metadata: dates, source locations, methodological context.
- Customer-stated expectations exist only in conversation. No tool call can recover them once summarized away.

---

# 5.2 — Escalation and ambiguity resolution

## Core Concept

Escalation is a policy decision with explicit triggers, not a sentiment reading. The three legitimate triggers:

1. The customer **explicitly requests** a human.
2. A **policy exception or policy gap** (the policy is silent or ambiguous on this request).
3. **Inability to make meaningful progress.**

Note what is *not* a trigger: mere case complexity. And note the branch distinction the exam loves:

- Customer **explicitly demands** a human → escalate immediately, no investigation first.
- Customer expresses **frustration** but the issue is within capability → acknowledge the frustration, offer resolution, escalate only if they reiterate their preference.

When escalation calibration is off, the proportionate first fix is explicit escalation criteria in the system prompt plus few-shot examples showing escalate-vs-resolve, before any new infrastructure.

## Canonical Skeleton — "Escalation Decision Gate"

```
1. Explicit request for a human?        → escalate NOW
2. Policy silent/ambiguous on request?  → escalate (policy gap)
3. Tool returned multiple matches?      → ask for an additional identifier
4. No meaningful progress possible?     → escalate
5. Otherwise                            → resolve autonomously
6. On any escalation → structured handoff:
     customer ID · root cause · amount · recommended action
     (the human agent has no access to the conversation transcript)
```

## Exam Traps

- **Sentiment-based escalation.** Sentiment does not correlate with case complexity. It solves a different problem.
- **Self-reported confidence scores (1–10) as an escalation trigger.** LLM self-confidence is poorly calibrated, and an agent mishandling hard cases is already wrongly confident on them.
- **Training a classifier on historical tickets.** Over-engineered when prompt optimization has not been attempted.
- **Investigating first when the customer demanded a human**, justified by the first-contact resolution target.
- **Heuristic selection among multiple customer matches** (most recent, best name match, highest match confidence). Ask for another identifier.
- Escalating every ambiguous tool result to a human. Multiple matches is a clarification case, not an escalation case.

## Key Facts to Memorize

- Three triggers: explicit request, policy exception/gap, no meaningful progress.
- Handoff summary contents: customer ID, root cause, refund amount, recommended action.
- Confidence used for **escalation** = unreliable. Confidence used for **review routing after calibration** = correct (see 5.5). Know which side a question is on.

---

# 5.3 — Error propagation across multi-agent systems

## Core Concept

A coordinator can only recover as intelligently as the error report allows. Structured error context contains: **failure type, what was attempted, partial results, and potential alternative approaches.**

The hard distinction: an **access failure** (timeout, service unavailable, needing a retry decision) is not the same as a **valid empty result** (the query succeeded and matched nothing). Collapsing them causes either pointless retries or silently abandoned topics.

Recovery layering: subagents handle **local recovery for transient failures** and propagate only what they cannot resolve, including what was attempted and any partial results.

## Canonical Skeleton — "Local Recover → Structured Propagate → Annotate Coverage"

```
SUBAGENT
  classify error → transient? retry locally
  unresolved → return {
     failureType, attemptedQuery, partialResults,
     alternatives, errorCategory, isRetryable
  }

COORDINATOR
  decide: retry with modified query | reroute to alternative source
        | proceed with partial results

SYNTHESIS
  coverage annotations: which findings are well-supported,
  which topic areas have gaps due to unavailable sources
```

Error categories carried over from Domain 2: **transient / validation / business / permission**, with an `isRetryable` boolean and a human-readable description. Business rule violations get `retriable: false` plus a customer-friendly explanation.

## Exam Traps

- **"Retry with exponential backoff, then return a generic 'search unavailable' status."** The retry is fine; the generic status is the defect. This is the most common partially-correct distractor in this task statement.
- **Return an empty result set marked successful.** Suppresses the error and risks incomplete research presented as complete.
- **Propagate the exception to a top-level handler that terminates the workflow.** Kills recoverable work.
- **Uniform "Operation failed" responses.** Prevents appropriate recovery decisions.
- Retrying a business or validation error, which will never succeed.

## Key Facts to Memorize

- Four error categories; `isRetryable` prevents wasted retry attempts.
- Access failure ≠ valid empty result.
- Partial results travel **with** the error, never discarded.
- Final output carries coverage-gap annotations.

---

# 5.4 — Context management in large codebase exploration

## Core Concept

The diagnostic tell in a scenario stem: after an extended session, the agent gives **inconsistent answers and references "typical patterns"** rather than the specific classes and files it discovered earlier. That phrasing means context degradation, and the answer is persistence plus isolation, never a bigger model.

Four mechanisms:

- **Subagent delegation** to isolate verbose exploration while the main agent keeps high-level coordination.
- **Scratchpad files** that persist key findings across context boundaries and are referenced for later questions.
- **Phase summarization**: summarize phase N's key findings, inject as initial context for phase N+1's subagents.
- **`/compact`** to reduce context usage mid-session when discovery output fills the window.

Crash recovery is separate: each agent exports **structured state to a known location**, and the coordinator loads a **manifest** on resume and injects it into agent prompts.

## Canonical Skeleton — "Explore → Persist → Summarize → Delegate"

```
Phase 1  Main agent maps structure (no bulk file contents in main context)
Phase 2  Spawn a subagent per specific question
         ("find all test files", "trace refund flow dependencies")
         → subagent returns a summary, not raw output
Phase 3  Write key findings to a scratchpad file; reference it later
Phase 4  Summarize the phase; inject the summary into the next phase's context
Phase 5  /compact when verbose discovery fills the window
Phase 6  Export agent state manifest for crash recovery
```

Session choice (shared with 1.7): use `--resume <session-name>` when prior context is **mostly valid**; start a **new session with an injected structured summary** when prior tool results are **stale**. On resume after code changes, tell the session which specific files changed for targeted re-analysis rather than forcing full re-exploration.

## Exam Traps

- Reading all files upfront. Build understanding incrementally: Grep for entry points, then Read to follow imports and trace flows.
- Larger context window as the remedy for degradation.
- "Summarize and continue in the same session." The degraded context is still in place; the pattern is summarize, then inject into a fresh subagent or session.
- Resuming a session whose tool results are stale.
- Restarting from scratch when a scratchpad or manifest would preserve valid findings.
- Treating a scratchpad as an instruction to the model to remember. It is a file on disk.

## Key Facts to Memorize

- Degradation symptom wording: inconsistent answers + generic "typical patterns."
- `/compact`, scratchpad files, Explore subagent, state manifests.
- Resume vs fresh-with-summary depends on staleness of prior tool results.

---

# 5.5 — Human review workflows and confidence calibration

## Core Concept

**Aggregate accuracy masks segment failure.** A 97% overall extraction accuracy can conceal a document type or a specific field performing far worse. Before reducing human review, validate accuracy **by document type and by field**.

Confidence is usable here, under three conditions that distinguish it from the escalation case in 5.2: it must be **field-level**, **calibrated against a labeled validation set**, and used to **route review attention** rather than to make the final business decision.

Ongoing assurance comes from **stratified random sampling of high-confidence extractions**, which measures the real error rate in auto-approved output and detects novel error patterns as document mixes drift.

## Canonical Skeleton — "Segment → Calibrate → Route → Sample"

```
1. SEGMENT   accuracy by document type × field on labeled data
2. EMIT      field-level confidence scores in the extraction schema
3. CALIBRATE thresholds using a labeled validation set
4. ROUTE     low-confidence + ambiguous/contradictory sources → human,
             prioritizing limited reviewer capacity
5. SAMPLE    stratified random sample of HIGH-confidence auto-approved
             output, continuously
6. FEED BACK detected_pattern fields on dismissed findings to analyze
             false-positive patterns
```

## Exam Traps

- "97% overall, so automate the high-confidence tier." Segment first.
- **Unstratified** random sampling. Rare document types, where errors concentrate, get under-sampled.
- Raw, uncalibrated confidence thresholds treated as probabilities.
- Sampling only the low-confidence items. You would never learn the error rate of what you auto-approve.
- Stopping sampling once targets are met, removing novel-pattern detection.
- Routing purely on confidence while ignoring **ambiguous or contradictory source documents**, which are an independent routing signal.

## Key Facts to Memorize

- Stratified random sampling targets **high-confidence** extractions.
- Calibration requires a **labeled validation set**.
- Segment by **document type AND field** before reducing review.

---

# 5.6 — Provenance and uncertainty in multi-source synthesis

## Core Concept

**Source attribution is lost at the summarization step.** If a subagent returns prose, the claim-to-source mapping no longer exists in synthesis's input, and any citation synthesis produces is a guess. The fix is structural: subagents emit **structured claim-source mappings** that downstream agents preserve and merge.

Conflicting statistics from credible sources are **annotated with attribution, not arbitrarily resolved**. Publication or data-collection dates are required in structured outputs so that temporal differences are not misread as contradictions. Report structure separates **well-established** from **contested** findings, preserving original source characterizations and methodological context.

Content type should drive rendering: financial data as tables, news as prose, technical findings as structured lists, rather than forcing one uniform format.

## Canonical Skeleton — "Claim-Source Record"

```
Each subagent finding:
{
  claim,
  evidence_excerpt,
  source_name / source_url,
  publication_or_collection_date,
  methodology_note
}

Coordinator: merges records, detects conflicts, decides reconciliation
Synthesis:   preserves mappings; report split into
             ESTABLISHED  |  CONTESTED (both values + attribution)  |  GAPS
```

The document analysis agent's job on conflicting values is to **complete the analysis with both values included and explicitly annotated**, letting the coordinator decide reconciliation before synthesis.

## Exam Traps

- "Prefer the more recent / more authoritative source." Arbitrary selection discards information, and the apparent conflict may be a temporal difference.
- "Average the values." Produces a figure present in neither source.
- "Instruct the synthesis agent to add citations." It cannot restore mappings absent from its input; this induces hallucinated attribution.
- Appending a bibliography of all source URLs. No claim-level mapping.
- Having synthesis re-fetch sources to resolve conflicts. Adds latency and may surface a third figure.
- Converting all findings to a uniform output format.

## Key Facts to Memorize

- Structured claim-source mappings must survive **every** intermediate step.
- Dates are mandatory metadata for temporal disambiguation.
- Conflicts are annotated with attribution; the coordinator, not the analysis agent, decides reconciliation.

---

# Exam-Style Questions

### Q1 — Support agent

After 25 turns on a billing dispute, the agent quotes a refund amount inconsistent with the figure the customer stated earlier. Conversation history is progressively summarized to stay within budget. Most effective fix?

A. Extract transactional facts into a persistent case facts block included in each prompt, outside the summarized history
B. Switch to a model with a larger context window so summarization is unnecessary
C. Instruct the summarizer to preserve all numerical values verbatim
D. Re-run `get_customer` and `lookup_order` at the start of each turn to refresh the data

**Answer: A.** Summarization discards precision; a persistent structured fact layer is immune to it. B does not address summarization loss or attention quality. C is a probabilistic instruction applied to a structural failure. D is actively harmful, re-bloating context, and cannot recover a customer-stated expectation, which exists only in the conversation and in no backend system.

---

### Q2 — Multi-agent research

The coordinator aggregates nine subagent reports into one synthesis prompt. Final reports consistently omit findings from the middle reports while covering the first and last thoroughly. Best mitigation?

A. Instruct the synthesis agent to read every section carefully before writing
B. Place a key findings summary at the beginning of the aggregated input and organize detailed results under explicit section headers
C. Reduce the number of subagents to four so there is less input to process
D. Have the synthesis agent process one report at a time with no cross-report view

**Answer: B.** This is the lost-in-the-middle effect, and the documented remedy is position-aware ordering plus explicit structure. A is exhortation against a positional processing characteristic. C narrows coverage and reintroduces the decomposition problem from Domain 1. D eliminates cross-source synthesis and conflict detection, which is the synthesis agent's purpose.

---

### Q3 — Support agent

A customer opens with "I want to speak to a human right now" regarding a damaged item. The agent runs `get_customer` and `lookup_order`, then proposes a replacement. What should it have done?

A. Continue the investigation, since the issue is within capability and the target is 80% first-contact resolution
B. Run sentiment analysis and escalate only if frustration exceeds a threshold
C. Escalate immediately with a structured handoff summary
D. Ask the customer to confirm they do not want automated help, then escalate if they reiterate

**Answer: C.** An explicit request for a human is honored immediately, without first attempting investigation. A subordinates a stated customer preference to a metric. B is an unreliable proxy and irrelevant when the request is explicit. D confuses the two branches: the acknowledge-and-offer-then-escalate-if-reiterated pattern applies to expressed frustration without an explicit demand.

---

### Q4 — Support agent

`get_customer` returns three matches for "J. Martinez." The agent selects the account with the most recent order and processes a refund against the wrong account. Best fix?

A. Rank matches by recency and order volume, selecting the highest-scoring
B. Instruct the agent to request an additional identifier (order number, email, postal code) when multiple matches are returned
C. Escalate to a human whenever a lookup returns multiple matches
D. Have the tool return a match-confidence score and select any match above 0.8

**Answer: B.** Multiple matches require clarification rather than heuristic selection. A restates the failing heuristic. C consumes scarce human capacity on a case one question resolves, and ambiguous lookups are not a listed escalation trigger. D is the same heuristic selection wearing a score, and match similarity does not establish identity.

---

### Q5 — Multi-agent research

The document analysis subagent is given five PDFs. One request times out; another returns a permission error behind a paywall. Best error propagation design?

A. Return a uniform "document unavailable" status for both failures
B. Return an empty result marked successful so the pipeline is not disrupted
C. Retry the transient timeout locally, and for the unresolved permission failure return structured error context with failure type, attempted access, partial results from the three successful documents, and alternative sources, allowing the coordinator to proceed and annotate coverage gaps
D. Propagate the exception to a top-level handler that fails the research task

**Answer: C.** Local recovery for transient failures, structured propagation of the unresolvable one, partial results preserved, coverage gap annotated. A collapses the transient/permission distinction the coordinator needs and hides partial results. B suppresses failure as success. D terminates a workflow where 60% of the evidence was successfully gathered.

---

### Q6 — Multi-agent research

The web search subagent returns "no results" both when the search API times out and when a query legitimately matches nothing. The coordinator repeatedly re-runs a genuinely empty query and silently abandons a topic that failed on a timeout. Most effective change?

A. Always retry any zero-result search once before treating it as empty
B. Increase the subagent's timeout so timeouts stop occurring
C. Distinguish access failures from valid empty results in the subagent's response, including failure type and retryability
D. Have the synthesis agent infer from the final report which gaps were caused by failures

**Answer: C.** The two conditions are semantically different and the coordinator's recovery decision depends on which occurred. A wastes calls on genuine empties and still gives the coordinator no basis for recovery. B reduces frequency without fixing the reporting semantics, and network failures persist. D asks an agent with no visibility into the tool layer to reconstruct information that was discarded upstream.

---

### Q7 — Developer productivity

After two hours exploring a legacy payments codebase, the agent begins giving inconsistent answers and describes "typical patterns in payment processing systems" rather than the specific classes it identified earlier. Best response?

A. Start a fresh session and re-read all the files
B. Ask the agent to summarize what it has learned and continue in the same session
C. Switch to a model with a larger context window
D. Have the agent maintain a scratchpad file of key findings, reference it for subsequent questions, and delegate further verbose investigation to subagents

**Answer: D.** The symptom is context degradation; the documented countermeasures are scratchpad persistence and subagent isolation. A discards valid findings and repeats expensive exploration. B leaves the degraded context in place, and summarization alone loses the specific detail that has already started disappearing. C misreads degradation as a capacity limit.

---

### Q8 — Multi-agent codebase migration

The system crashes at roughly 70% completion. On restart, the coordinator re-runs every subagent from the beginning. Best design for recovery?

A. Have each agent export structured state to a known location and have the coordinator load a manifest on resume, injecting it into agent prompts
B. Use `--resume <session-name>` to restore the prior coordinator conversation
C. Increase the checkpoint frequency of the conversation transcript
D. Have the coordinator re-read the raw execution log and infer which subtasks completed

**Answer: A.** Structured state exports plus a manifest loaded on resume is the documented crash-recovery pattern. B restores conversation state, but subagents do not share the coordinator's session and prior tool results may be stale. C persists the transcript rather than structured work products. D substitutes inference over unstructured logs for an explicit manifest.

---

### Q9 — Structured extraction

Your pipeline reports 97% aggregate extraction accuracy. A stakeholder proposes auto-approving all high-confidence extractions and eliminating review for them. Best approach?

A. Auto-approve above the threshold and randomly sample 5% of all output for review
B. Analyze accuracy by document type and field, calibrate field-level confidence thresholds on a labeled validation set, and maintain stratified random sampling of high-confidence extractions
C. Auto-approve above the threshold and route only low-confidence extractions to human review
D. Raise the confidence threshold to 0.95 as a safety margin before automating

**Answer: B.** Aggregate accuracy can mask poor performance on particular document types or fields, so segmentation precedes automation, and calibration requires labeled data. A uses unstratified sampling, which under-represents the rare document types where errors concentrate. C removes the only mechanism for measuring error rates in auto-approved output and detecting novel patterns. D adjusts an uncalibrated number, which is not a meaningful probability.

---

### Q10 — Multi-agent research

Two credible industry reports give different market-size figures for the same sector. The document analysis agent currently selects the more recently published value. Best handling?

A. Instruct the analysis agent to continue preferring the more recent publication
B. Average the two values and report the range
C. Return both values explicitly annotated with source and publication/collection date, let the coordinator decide reconciliation, and structure the report to distinguish well-established from contested findings
D. Have the synthesis agent run a fresh web search to determine which figure is correct

**Answer: C.** Conflicts are annotated with attribution rather than arbitrarily resolved, and dates prevent a temporal difference from being read as a contradiction. A discards information the reader needs to judge the claim. B fabricates a figure appearing in neither source. D adds latency and is likely to surface a third figure rather than adjudicate.

---

### Q11 — Multi-agent research

Final reports read coherently, but citations are frequently misattributed or missing. Subagents currently return narrative prose summaries of what they found. Most effective fix?

A. Instruct the synthesis agent to add a citation to every claim
B. Append a complete list of consulted source URLs at the end of each report
C. Have the synthesis agent re-fetch each source to verify attribution
D. Require subagents to output structured claim-source mappings (claim, evidence excerpt, source name or URL, publication date) that downstream agents preserve through synthesis

**Answer: D.** Attribution is lost at the summarization step, so it must be preserved structurally from the point of discovery. A asks synthesis to cite mappings that no longer exist in its input, which produces fabricated attribution. B provides a bibliography without claim-level mapping. C is expensive and still leaves synthesis guessing which source supported which claim.

---

### Q12 — Support agent

`lookup_order` returns 40+ fields per order, of which about five are relevant to return handling. By turn 15, order JSON dominates the context window. Best fix?

A. Implement a PostToolUse hook that trims tool output to the relevant fields before results enter context
B. Run `/compact` when the context fills
C. Instruct the agent to ignore irrelevant fields in tool responses
D. Summarize accumulated tool results after every ten turns

**Answer: A.** Verbose tool output is trimmed before it accumulates, and a PostToolUse hook gives a deterministic guarantee. B is a Claude Code session command and is reactive rather than preventive. C consumes the tokens regardless of whether the model attends to them. D is reactive and risks compressing exact order values, which is the precision loss described in 5.1.

---

# Fast answer-selection heuristics for Domain 5

When two options look plausible, the correct one usually:

- Makes information **structurally persistent** (facts block, scratchpad, manifest, claim-source record) instead of instructing the model to remember or be careful.
- **Preserves information** (annotate both conflicting values, return partial results) instead of discarding it (pick one, return empty, terminate).
- **Trims or isolates at the source** (PostToolUse hook, subagent delegation, upstream structured output) instead of cleaning up downstream.
- Treats **larger context windows and stronger models as non-answers** for attention, degradation, and summarization problems.
- Uses **calibrated, field-level confidence to route review**, and never uncalibrated self-reported confidence to make a business decision.
- Is **proportionate**: prompt-level criteria and few-shot examples before classifiers and ML infrastructure, except where a deterministic business guarantee is required, in which case a programmatic hook or gate wins.

---

# Quick-reference cheat sheet

| Symptom in the stem | Root cause | Correct pattern |
|---|---|---|
| Precise values drift over a long chat | Progressive summarization | Persistent case facts block outside summarized history |
| Middle sections of aggregated input ignored | Lost-in-the-middle | Key findings first + explicit section headers |
| Context filled with tool JSON | Tool-result bloat | Trim at ingest via PostToolUse hook |
| Agent escalates easy cases, attempts hard ones | Unclear decision boundaries | Explicit escalation criteria + few-shot examples |
| Customer asked for a human | Explicit request | Escalate immediately + structured handoff |
| Policy silent on the request | Policy gap | Escalate |
| Multiple lookup matches | Ambiguity | Ask for an additional identifier |
| Coordinator can't recover from subagent failure | Generic error status | Structured error context + partial results |
| Empty results indistinguishable from timeouts | Collapsed error semantics | Separate access failure from valid empty result |
| Agent cites "typical patterns" after long session | Context degradation | Scratchpad file + subagent delegation + /compact |
| Crash loses all progress | No state persistence | Agent state exports + coordinator manifest on resume |
| 97% accuracy proposed as grounds to automate | Aggregate masks segments | Segment by doc type × field, calibrate, stratified sampling |
| Citations misattributed in final report | Attribution lost in summarization | Structured claim-source mappings preserved end to end |
