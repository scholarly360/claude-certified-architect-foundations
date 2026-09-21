# CCAR-F Study Guide — Domain 4: Prompt Engineering & Structured Output

> **Exam:** Claude Certified Architect – Foundations (CCAR-F), Exam Guide v1.0, effective July 2026
> **Domain weight:** 20% (~12 of 60 scored items)
> **Primary scenarios:** Scenario 5 (Claude Code for CI/CD) and Scenario 6 (Structured Data Extraction)

---

## Task Statement Map

| # | Task Statement | One-line essence |
|---|---|---|
| 4.1 | Explicit criteria to improve precision | Categorical criteria beat "be conservative" |
| 4.2 | Few-shot prompting | 2–4 targeted examples for ambiguous cases |
| 4.3 | Structured output via tool use + JSON schema | Schema guarantees syntax, not semantics |
| 4.4 | Validation, retry, feedback loops | Retry fixes format errors, never missing info |
| 4.5 | Batch processing | 50% cheaper, up to 24h, no SLA |
| 4.6 | Multi-instance / multi-pass review | Independent reviewer + per-file + integration pass |

---

## 4.1 Explicit Criteria to Improve Precision

### Core Concept

Precision problems in review and extraction prompts are almost never solved by telling the model to be more careful. "Be conservative," "only report high-confidence findings," and "avoid false positives" are calibration instructions with no categorical content, so the model has no new decision boundary to apply. What works is defining the *category* of thing to report and the category to skip, in terms the model can evaluate against the artifact in front of it.

The exam frames this as a trust problem: one high-false-positive category poisons developer confidence in the categories that are actually accurate. So the correct move when one category misfires is to disable that category temporarily while you rewrite its criteria, not to dial down the whole review.

### Canonical Skeleton

```text
REPORT these categories:
  - Bugs: logic errors, null/undefined dereferences, off-by-one, unhandled async rejection
  - Security: injection, missing authz check, secrets in source
  - Comment accuracy: flag ONLY when the documented behavior contradicts
    what the code actually does (not when a comment is merely terse or stale in tone)

SKIP these categories:
  - Style and formatting (handled by the linter)
  - Patterns that are locally consistent with the surrounding file
  - Speculative performance concerns without a measured hot path

SEVERITY (with an example of each):
  BLOCKER  - <concrete code example>
  MAJOR    - <concrete code example>
  MINOR    - <concrete code example>

OUTPUT: one finding per issue, using the schema below.
```

### Exam Traps

- Choosing "instruct the model to only report findings it is highly confident about." This is the single most common distractor in this task statement and it is always wrong.
- Choosing "raise the severity threshold" when the problem is that a category's *definition* is wrong.
- Assuming a larger or stronger model fixes precision. Precision here is a specification problem, not a capability problem.
- Confusing this with 4.2. Explicit criteria define *what counts*; few-shot examples demonstrate *how to judge and format*. When a question says the rules are clear but the output shape or the ambiguous cases are inconsistent, the answer is few-shot, not more criteria.

### Key Facts to Memorize

- Specific categorical criteria beat confidence-based filtering, always.
- Severity consistency requires a concrete code example anchored to each level, not adjectives.
- Disabling a bad category is a legitimate interim answer; it preserves trust in the rest.
- False positives are costed in developer trust, not just wasted time.

---

## 4.2 Few-Shot Prompting

### Core Concept

Few-shot examples are the guide's designated fix when detailed prose instructions already exist but output is still inconsistent, especially at the ambiguous edges. The three jobs few-shot does on this exam: lock the output format, demonstrate judgment on genuinely ambiguous cases, and reduce hallucination in extraction by showing how odd source formats map to schema fields.

The critical nuance is generalization. A good example shows *why* one choice was made over a plausible alternative, which lets the model extend the judgment to novel patterns. A bad example set is just an enumeration of special cases, which the model pattern-matches without generalizing.

### Canonical Skeleton

```xml
<example>
  <input>Measurement recorded as "a couple of teaspoons, maybe 10 mL"</input>
  <reasoning>Source gives an informal range plus an explicit metric value.
    Prefer the explicit numeric value; flag precision as low rather than null.</reasoning>
  <output>{"amount": 10, "unit": "mL", "precision": "approximate"}</output>
</example>

<example>
  <input>Citation appears only in a bibliography, not inline</input>
  <reasoning>Bibliography-only sources still count as cited; map to the
    same field as inline citations rather than returning null.</reasoning>
  <output>{"sources": [...], "citation_style": "bibliography"}</output>
</example>
```

Two to four examples. Each one should target an ambiguity that actually appeared in your failure logs.

### Exam Traps

- Padding to 5–8 examples. The guide's own sample question marks a large few-shot block wrong when the real root cause lay elsewhere (there, thin tool descriptions). Few-shot is not the universal answer.
- Using few-shot to fix a problem that is structural, e.g. using examples to enforce a JSON shape that tool use would guarantee. If the question mentions parse failures or malformed JSON, the answer is tool use with a schema, not examples.
- Examples that only cover the easy cases. Exam-correct examples cover the ambiguous boundary.
- Believing few-shot can make up for information that is not in the document. It cannot.

### Key Facts to Memorize

- 2–4 targeted examples for ambiguous scenarios.
- Show the reasoning for choosing one action over a plausible alternative.
- Few-shot **is** the named fix for: inconsistent output format, ambiguous tool selection, varied document structures, null extraction of fields that *are* present, and false positives where acceptable patterns must be distinguished from real issues.
- Few-shot is **not** the fix for: JSON syntax errors, missing source information, or ordering guarantees.

---

## 4.3 Structured Output via Tool Use and JSON Schemas

### Core Concept

The reliable way to get schema-compliant output from Claude is to define a tool whose `input_schema` *is* your target schema, then read the `input` object off the returned `tool_use` block. The model is constrained to the schema, which eliminates JSON syntax errors and the whole class of "strip the markdown fences and hope" parsing code.

The boundary you must hold in your head: the schema constrains **syntax and shape**, not **truth**. Line items that don't sum to the stated total, a vendor name landing in the `customer` field, a date that's plausible but wrong — all of these are perfectly schema-valid. Semantic correctness is 4.4's problem.

Schema design carries its own reliability weight. A field marked `required` when the source document may not contain it creates pressure to fabricate. Make it nullable. A closed enum on an open-world category forces miscategorization. Add `"other"` plus a detail string, and `"unclear"` for genuine ambiguity.

### Canonical Skeleton

```json
{
  "name": "extract_invoice",
  "description": "Extract structured invoice fields from the document text.",
  "input_schema": {
    "type": "object",
    "properties": {
      "invoice_number": { "type": "string" },
      "vendor_name":    { "type": "string" },
      "po_number":      { "type": ["string", "null"],
                          "description": "Null if no PO is referenced in the document." },
      "category":       { "type": "string",
                          "enum": ["goods", "services", "subscription", "other", "unclear"] },
      "category_detail":{ "type": ["string", "null"],
                          "description": "Required when category is 'other'." },
      "line_items":     { "type": "array", "items": { "...": "..." } },
      "stated_total":     { "type": "number" },
      "calculated_total": { "type": "number",
                            "description": "Sum of line items, computed independently." }
    },
    "required": ["invoice_number", "vendor_name", "category", "stated_total"]
  }
}
```

Paired with `tool_choice`:

| Setting | Behavior | When the exam wants it |
|---|---|---|
| `{"type": "auto"}` | Model may call a tool or may reply with text | Conversational agents; **never** when you need guaranteed structure |
| `{"type": "any"}` | Model must call *some* tool, its choice | Multiple extraction schemas, document type unknown |
| `{"type": "tool", "name": "extract_metadata"}` | Model must call that specific tool | Guaranteeing a particular extraction runs first, before enrichment |

Forcing a tool covers the *first* step only. Subsequent steps happen in follow-up turns.

Add format normalization rules in the prompt alongside the schema (e.g. "render all dates as ISO 8601 regardless of source format") — the schema types a field as a string, it does not normalize it.

### Exam Traps

- "Strict JSON schemas eliminate extraction errors." False. They eliminate *syntax* errors.
- Marking everything `required` for completeness. This is the named cause of fabricated values.
- Using `tool_choice: "auto"` in an extraction pipeline. Auto permits a text reply, which is exactly the failure you were trying to design out.
- Confusing `"any"` with forced. `"any"` guarantees *a* tool call; it does not choose which.
- Reaching for prompt-only JSON instructions plus a regex/fence-stripping parser. That's the pre-tool-use pattern and it's always the wrong answer here.
- Assuming forced tool choice sequences a multi-step pipeline. It doesn't; it pins one call.

### Key Facts to Memorize

- Tool use + JSON schema = guaranteed schema-compliant, syntax-error-free output.
- Nullable/optional fields are the anti-hallucination mechanism for absent information.
- `enum` + `"other"` + detail string = extensible categorization; `"unclear"` = ambiguity valve.
- `auto` / `any` / forced — know all three behaviors cold; near-certain to be tested.
- In Claude Code CI contexts the equivalent levers are `--output-format json` and `--json-schema`.

---

## 4.4 Validation, Retry, and Feedback Loops

### Core Concept

Because tool use removes syntax errors, the validation layer that remains is semantic: Pydantic or JSON Schema business-rule checks. The retry pattern is *retry with error feedback* — send back the original document, the failed extraction, and the specific validation error text, so the model has something concrete to correct against.

The judgment the exam tests is knowing when retry is futile. Retry succeeds on format mismatches and structural output errors, where the information exists and was rendered wrong. Retry fails when the information is simply not in the source, for example when a required value lives in a referenced external document that was never supplied. Retrying that burns tokens and eventually produces a fabrication. The correct handling is to surface it: null the field, flag it, route to human review, or fetch the missing document.

Self-correction can be designed into the schema itself. Extract `calculated_total` alongside `stated_total` so a mismatch is detectable without a second call. Add a `conflict_detected` boolean when the source may state inconsistent values. Add a `detected_pattern` field to each review finding so that when developers dismiss findings, you can aggregate dismissals by pattern and see systematically which construct is generating false positives.

### Canonical Skeleton

```python
result = extract(document)                  # tool_use, schema-constrained

try:
    validated = InvoiceModel(**result)      # semantic validation
except ValidationError as e:
    if information_absent(e):               # value not in source
        route_to_human(result, reason=e)    # do NOT retry
    else:
        result = extract(
            document,
            failed_extraction=result,
            validation_errors=str(e),       # specific, not "it was invalid"
        )
        validated = InvoiceModel(**result)  # bounded retries, then escalate
```

### Exam Traps

- Retrying with the same prompt and no error detail. Without the specific error the retry is a coin flip.
- Unbounded retries on missing information. This is the marquee trap of 4.4.
- Believing validation failures indicate a schema problem when they indicate a source problem.
- Choosing "add the validation rules to the schema" for a cross-field arithmetic rule. JSON Schema is not where line-item summation lives; that's Pydantic/application validation, or the `calculated_total` pattern.
- Treating dismissed findings as noise instead of instrumenting them with `detected_pattern`.

### Key Facts to Memorize

- Retry payload = original document + failed extraction + specific validation errors.
- **Retryable:** format mismatches, structural output errors. **Not retryable:** information absent from source.
- `calculated_total` vs `stated_total` → discrepancy flag without a second model call.
- `conflict_detected` boolean → inconsistent source data.
- `detected_pattern` → systematic false-positive analysis from dismissal data.

---

## 4.5 Batch Processing Strategy

### Core Concept

The Message Batches API trades latency for cost: 50% savings, up to a 24-hour processing window, no guaranteed latency SLA. That single trade determines every answer in this task statement. Latency-tolerant and non-blocking work (overnight technical debt reports, weekly audits, nightly test generation) goes to batch. Anything a human is waiting on (a pre-merge check that gates a developer) stays synchronous. The 50% saving never justifies blocking a developer for an unbounded window.

Two additional constraints: the batch API does not support multi-turn tool calling inside a single request, so any workflow that needs to execute a tool mid-request and feed results back cannot be batched as one call. And `custom_id` is how you correlate requests to responses, which is also how you identify and resubmit only the failures.

### Canonical Skeleton

```text
1. Refine the prompt on a small sample synchronously   # maximize first-pass yield
2. Submit batch, one custom_id per document
3. Poll for completion
4. Triage results by custom_id:
      succeeded       -> downstream
      context limit   -> chunk the document, resubmit
      validation fail -> retry-with-feedback, resubmit
5. Resubmit ONLY the failed custom_ids
```

**SLA arithmetic** (called out explicitly in the guide): with a 30-hour end-to-end commitment and a 24-hour worst-case processing window, submitting every 4 hours gives a worst case of 4 + 24 = 28 hours, leaving headroom. Expect a question that hands you an SLA and asks for the submission cadence.

```text
worst_case = submission_interval + 24h  ≤  SLA
```

### Exam Traps

- Moving a blocking workflow to batch "because batches usually finish fast." Usually is not an SLA.
- A hybrid design that starts a batch and falls back to synchronous on timeout. Added complexity for a problem solved by correct routing. The guide marks this wrong.
- "Batch results come back out of order so we can't use it." Misconception; `custom_id` handles correlation.
- Resubmitting the whole batch after partial failure instead of just the failed `custom_id`s.
- Assuming an agentic tool-calling loop can be batched as one request.

### Key Facts to Memorize

- 50% cost reduction, up to 24 hours, no latency guarantee.
- `custom_id` correlates request and response, and scopes resubmission.
- No multi-turn tool calling within a single batch request.
- Sample-then-batch: refine on a sample first to cut resubmission cost.
- Blocking = synchronous. Non-blocking + latency-tolerant = batch.

---

## 4.6 Multi-Instance and Multi-Pass Review Architectures

### Core Concept

Two independent failure modes, two different fixes.

**Self-review bias.** A model that just generated code carries its generation reasoning in context. Asked to review that code in the same session, it is disposed to re-endorse the decisions it already justified. The fix is a second, independent instance with no access to that reasoning. Notably, "ask it to review its own work more carefully" and "turn on extended thinking" are both weaker than independence, because neither removes the prior commitment.

**Attention dilution.** One pass over 14 files produces uneven depth: thorough on some files, superficial on others, and sometimes self-contradictory, flagging a pattern in one file that it approves in another. The fix is decomposition — a focused per-file pass for local issues, then a separate integration pass for cross-file data flow. A bigger context window does not fix this; the files fit, the attention doesn't distribute.

The third technique is confidence-calibrated routing: have the model self-report confidence per finding so findings can be routed, not filtered. Note the tension with 4.1 — self-reported confidence is a poor *filter* for precision, but it is usable as a *routing* signal once calibrated against a labeled set (which is where this hands off to Domain 5.5).

### Canonical Skeleton

```text
Generation instance ──> code
                          │
                          ▼
Independent review instance (fresh context, review criteria from 4.1)
                          │
        ┌─────────────────┴─────────────────┐
        ▼                                   ▼
  Per-file passes                   Cross-file integration pass
  (local: bugs, security,           (data flow, contract mismatches,
   comment accuracy)                 duplicated/conflicting logic)
        └─────────────────┬─────────────────┘
                          ▼
              Merge + dedupe findings
```

### Exam Traps

- "Instruct the model to critically review its own output." Same session, same commitment, weaker result.
- "Use a larger context window model." Named explicitly as a misconception about attention quality.
- "Run three passes and only report issues appearing in at least two." Consensus voting suppresses real bugs that are caught intermittently. Always wrong on this exam.
- "Require developers to split the PR into smaller pieces." Shifts burden to humans without improving the system.
- Dropping the integration pass. Per-file only will miss cross-file data flow issues by construction.

### Key Facts to Memorize

- Independent instance > self-review instruction > extended thinking, for catching subtle issues.
- Per-file passes for local issues + separate integration pass for cross-file issues.
- Larger context ≠ better attention distribution.
- Consensus/voting across passes is a distractor, not a technique.
- Confidence as a routing signal is fine; confidence as a precision filter is not.

---

## Cross-Cutting Distractor Taxonomy

Domain 4 items reward recognizing distractor *shapes* as much as knowing the content. Five recur throughout the guide's own samples:

1. **Over-engineering.** Train a classifier, add an ML routing layer, build a preprocessing pipeline — offered before the cheap prompt-level fix has been tried. Nearly always wrong when the stem implies a first step.
2. **Confidence theater.** Self-reported confidence scores used as a quality gate.
3. **Bigger hammer.** Larger model, larger context window, more examples — when the problem is specification or architecture.
4. **Burden shifting.** Make developers change their behavior instead of fixing the system.
5. **Invented features.** Flags, env vars, and config files that don't exist (`--batch`, `CLAUDE_HEADLESS`, `.claude/config.json`). If you don't recognize a flag, be suspicious of it.

**Counter-heuristic:** identify the root cause stated in the stem, then pick the smallest proportionate intervention that addresses *that* cause. The guide's explanations consistently reward proportionality.

---

## Exam-Style Questions

### Q1

Your CI review prompt flags roughly 40% false positives in the "comment accuracy" category, and developers have started ignoring all review output including the security findings, which are accurate. What is the most effective response?

- A. Instruct the model to report comment issues only when highly confident.
- B. Temporarily disable the comment accuracy category and rewrite its criteria to flag only cases where documented behavior contradicts actual code behavior.
- C. Raise the minimum severity threshold across all categories.
- D. Switch to a larger model for the review pass.

**Answer: B.** The stem contains two problems: one broken category and eroded trust in the good ones. Disabling the bad category restores trust immediately while you fix the criteria, and the rewritten criterion is categorical rather than confidence-based. A is the confidence-theater distractor and provides no new decision boundary. C degrades the categories that are working. D treats a specification problem as a capability problem.

---

### Q2

An extraction pipeline uses a prompt instructing Claude to return JSON. About 8% of responses fail to parse because of markdown fences, trailing commas, or truncated objects. What is the correct fix?

- A. Add few-shot examples showing correctly formatted JSON.
- B. Add a post-processing step that strips fences and repairs common JSON errors.
- C. Define the schema as a tool's `input_schema` and read the structured result from the `tool_use` block.
- D. Lower temperature and increase `max_tokens`.

**Answer: C.** Tool use with a JSON schema constrains the model to the schema and eliminates syntax errors as a class. A reduces the rate but doesn't eliminate it. B is the pre-tool-use workaround and leaves the failure mode in place. D is incidental — truncation is a `max_tokens` issue, but it doesn't address fences or malformed structure.

---

### Q3

You extract from three document types and cannot reliably determine the type before the call. Each type has its own extraction schema. You need guaranteed structured output on every call. Which `tool_choice` configuration?

- A. `{"type": "auto"}`
- B. `{"type": "any"}`
- C. `{"type": "tool", "name": "extract_invoice"}`
- D. No `tool_choice`; instruct the model in the system prompt to always call a tool.

**Answer: B.** `"any"` forces a tool call while letting the model choose which schema fits the document. A permits a plain text reply, which is the failure you're designing out. C forces one schema onto documents that may be a different type. D is probabilistic where a deterministic control exists.

---

### Q4

After switching to tool use with a strict schema, JSON parse failures drop to zero. However, downstream reconciliation still rejects about 5% of invoices because line items do not sum to the stated total. What does this indicate?

- A. The schema needs additional required fields.
- B. Schema enforcement guarantees syntax, not semantics; this class of error requires a separate validation layer.
- C. `tool_choice` should be changed to forced tool selection.
- D. The model needs more few-shot examples of correctly summed invoices.

**Answer: B.** This is the core distinction of 4.3/4.4. A schema cannot express or enforce a cross-field arithmetic relationship over an unbounded array. The designed fix is application-level validation, with `calculated_total` alongside `stated_total` so the discrepancy is visible in the extraction itself. A and D don't address the arithmetic. C changes which tool is called, not what is checked.

---

### Q5 *(Select TWO)*

Validation fails on a required `po_number` field for about 12% of documents. Inspection shows those documents genuinely have no PO reference — it lives in a separate procurement system. Your retry loop currently retries three times with validation feedback. What should change?

- A. Make `po_number` nullable and stop treating its absence as a validation failure.
- B. Increase the retry count to five.
- C. Add explicit instruction that the model must never leave `po_number` empty.
- D. Detect absent-information failures and route them out of the retry loop.
- E. Add few-shot examples showing PO numbers extracted from similar invoices.

**Answer: A and D.** Retry cannot recover information that is not in the source; it will eventually pressure the model into fabricating a PO number. Making the field nullable removes the fabrication pressure, and routing absent-information failures out of the loop stops the token burn. B extends a futile loop. C actively increases hallucination risk. E teaches extraction of a value that isn't present.

---

### Q6

Your team wants to cut costs on two workflows: a pre-merge security check that gates developer merges, and a weekly architecture drift report reviewed on Monday mornings. Which approach is correct?

- A. Move both to the Message Batches API for the 50% savings.
- B. Move the weekly report to batch; keep the pre-merge check synchronous.
- C. Keep both synchronous to avoid `custom_id` correlation complexity.
- D. Move both to batch with a synchronous fallback if a batch exceeds two hours.

**Answer: B.** Batch offers 50% savings with up to 24 hours of processing and no latency guarantee. The weekly report is latency-tolerant and non-blocking; the pre-merge check has a developer waiting on it. A blocks developers for an unbounded window. C rests on a misconception — `custom_id` handles correlation. D adds a dual-path system to solve a problem that correct routing already solves.

---

### Q7

You must process 500 documents per day through a pipeline with a 30-hour end-to-end SLA, using the Message Batches API. What submission cadence guarantees the SLA?

- A. One batch daily at midnight.
- B. Batches every 4 hours.
- C. Batches every 12 hours.
- D. Submit each document individually as it arrives.

**Answer: B.** Worst case is the wait for the next submission window plus the maximum processing window. At 4 hours: 4 + 24 = 28 hours, inside the 30-hour SLA with headroom for failure resubmission. At 12 hours: 12 + 24 = 36, which breaches. A can reach nearly 48 hours for a document arriving just after submission. D forfeits batching entirely.

---

### Q8

A batch of 100 documents returns 94 successes and 6 failures, all exceeding the context limit. What is the correct handling?

- A. Resubmit the full batch with a larger `max_tokens`.
- B. Identify the 6 by `custom_id`, chunk those documents, and resubmit only those.
- C. Reprocess all 100 synchronously.
- D. Drop the 6 and flag them for manual entry.

**Answer: B.** `custom_id` exists precisely to identify which requests failed. Chunking addresses the actual cause (input length), and resubmitting only the failures preserves the cost advantage. A resubmits 94 successful documents and confuses output limits with input limits. C discards the savings. D escalates to humans before an automated remedy has been tried.

---

### Q9

Claude Code generates a new authentication module. You ask the same session to review its own output for security issues; it reports no findings. An independent audit later finds two real vulnerabilities. What is the most effective architectural change?

- A. Add an instruction telling the model to critically challenge its own assumptions during review.
- B. Enable extended thinking on the review request.
- C. Run the review in a separate Claude instance without the generation session's reasoning context.
- D. Run the same self-review three times and report any issue appearing in at least two runs.

**Answer: C.** A model retains its generation reasoning and is disposed to re-endorse decisions it already justified in-session. Independence removes that commitment. A and B both operate inside the compromised context and are explicitly weaker. D is consensus voting, which suppresses intermittently-detected real bugs.

---

### Q10

A PR touching 14 files produces a review with detailed feedback on some files, superficial comments on others, and a pattern flagged as a defect in one file while identical code passes in another. What is the correct restructuring?

- A. Per-file passes for local issues, plus a separate integration pass for cross-file data flow.
- B. Switch to a model with a larger context window.
- C. Require developers to submit PRs of no more than 4 files.
- D. Increase `max_tokens` so the review isn't truncated.

**Answer: A.** The symptoms are textbook attention dilution, and the fix is decomposition: consistent depth per file, plus a dedicated pass for the issues that only appear across files. B confuses context capacity with attention quality. C shifts burden onto developers without improving the system. D addresses output length, not review depth or the contradiction.

---

### Q11

Developers dismiss about 30% of findings. You want to determine systematically which code constructs generate false positives. What should you add to the finding schema?

- A. A `confidence` float on every finding.
- B. A `detected_pattern` field naming the construct that triggered the finding.
- C. A free-text `notes` field for developer feedback.
- D. A `severity` enum.

**Answer: B.** Recording the triggering construct makes dismissals aggregable, so you can see that (for example) a particular async pattern accounts for most of the dismissals and rewrite that criterion. A gives you the model's self-assessment, which is the wrong signal and not tied to a construct. C produces unstructured data that doesn't aggregate. D already exists in a standard finding schema and doesn't identify the trigger.

---

### Q12

Extraction returns null for `methodology` in about 25% of research papers that visibly contain methodology information, sometimes in a labeled section, sometimes embedded in the results narrative. Prompt instructions already describe the field in detail. What is the most effective fix?

- A. Mark `methodology` as required so the model cannot return null.
- B. Add few-shot examples showing correct extraction from both labeled-section and embedded-narrative documents.
- C. Switch `tool_choice` to forced selection.
- D. Add a retry loop that re-requests the field when it comes back null.

**Answer: B.** Instructions already exist and are being applied inconsistently across document structures, which is the exact signature for few-shot. Examples spanning both structural variants let the model generalize to the embedded case. A creates fabrication pressure on the papers that genuinely lack a methodology section. C controls which tool is called, not extraction quality. D retries a failure whose cause is structural interpretation, not transient error.

---

## Last-Minute Cram Sheet

| If the stem says… | The answer is usually… |
|---|---|
| "False positives," "developers ignore findings" | Explicit categorical criteria; disable the noisy category temporarily |
| "Instructions exist but output is inconsistent" | 2–4 few-shot examples covering the ambiguous cases |
| "JSON parse errors," "malformed output" | Tool use with JSON schema |
| "Must always return structured output, type unknown" | `tool_choice: "any"` |
| "This extraction must run first" | Forced `tool_choice` for that tool |
| "Model fabricates values for missing fields" | Make the field nullable/optional |
| "Values don't sum," "wrong field" | Semantic validation layer; `calculated_total` vs `stated_total` |
| "Retries keep failing on the same field" | Check if info is absent from source; stop retrying |
| "Cut costs" + blocking workflow | Keep synchronous |
| "Cut costs" + overnight/weekly | Message Batches API |
| "Some batch items failed" | Resubmit only failed `custom_id`s, with a fix |
| "Same session reviewed its own code" | Independent review instance |
| "Many files, uneven/contradictory review" | Per-file passes + cross-file integration pass |

**Highest-yield items:** the `auto`/`any`/forced distinction, syntax-vs-semantics on schema enforcement, retryable-vs-not, the batch latency trade, independent review over self-review, and the five distractor shapes.
