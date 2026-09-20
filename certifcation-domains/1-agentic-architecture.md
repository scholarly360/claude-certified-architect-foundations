# Claude Certified Architect – Foundations (CCAR-F)
## Domain 1: Agentic Architecture & Orchestration

**Weight:** 27% of scored items (~16 of 60) — the heaviest domain.
**Appears in:** Scenario 1 (Customer Support Agent), Scenario 3 (Multi-Agent Research), Scenario 4 (Developer Productivity).
**Task statements covered:** 1.1 – 1.7

---

## 1. Core Concept

Every Domain 1 question reduces to one question: **where does control live — in the model, or in your code?**

Claude decides *what to do next* (model-driven reasoning over context). Your code decides *what is permitted to happen* (deterministic gates). Confusing these two is how candidates lose points.

### Five axes that generate almost every item

| Axis | Model side | Code side | When code wins |
|---|---|---|---|
| Loop control | Claude picks next tool | You read `stop_reason` | Always — loop flow is mechanical |
| Compliance | "Always verify first" in prompt | Prerequisite gate / hook | Financial, identity, policy consequences |
| Context flow | — | Explicit prompt injection | Always — subagents inherit nothing |
| Decomposition | Adaptive replanning | Fixed prompt chain | Predictable multi-aspect work |
| Communication | — | Hub-and-spoke through coordinator | Always — observability + error handling |

### 1.1 The agentic loop

The loop is driven by `stop_reason`, not by text.

- `stop_reason: "tool_use"` → execute the requested tools, append results to conversation history, call again.
- `stop_reason: "end_turn"` → the model is finished; present the final response.

Tool results must go back into the message history as `tool_result` blocks so the model can reason over new information on the next iteration. This is model-driven decision-making, not a pre-configured decision tree.

### 1.2 Coordinator–subagent orchestration

Hub-and-spoke: the coordinator is the single channel for all inter-subagent communication, error handling, and routing. Subagents never talk to each other.

The coordinator:
- decomposes the task
- selects *which* subagents to invoke based on query complexity (not always the full pipeline)
- aggregates results
- runs iterative refinement: evaluates synthesis output for gaps, re-delegates targeted queries, re-invokes synthesis until coverage is sufficient

**Signature failure mode:** subagents all succeed, output is still incomplete. Root cause is upstream — the coordinator decomposed too narrowly.

### 1.3 Subagent invocation and context passing

- Subagents are spawned via the **Task tool**. The coordinator's `allowedTools` must include `"Task"`.
- Subagents operate with **isolated context**: no automatic inheritance of parent history, no shared memory between invocations.
- Anything a subagent needs must be written into its prompt.
- Use structured formats that separate content from metadata (source URL, document name, page number, publication date) so attribution survives the handoff.
- Parallelism = **multiple Task calls in a single coordinator response**. Across separate turns is sequential, not parallel.
- `AgentDefinition` carries description, system prompt, and tool restrictions per subagent type.

### 1.4 Enforcement and handoff

Prompt instructions are probabilistic and have a non-zero failure rate. When a sequence must hold every time (verify identity before moving money), use a programmatic prerequisite gate.

For mid-process escalation, compile a structured handoff summary. The human receiving it does not have access to the conversation transcript.

### 1.5 Hooks

Two directions, and the exam tests that you know which is which:

- **`PostToolUse`** intercepts *incoming tool results* before the model processes them. Use for normalization (Unix timestamps vs ISO 8601 vs numeric status codes from different MCP servers).
- **Tool-call interception** intercepts *outgoing tool calls* before execution. Use to block policy violations (refund over $500) and redirect to escalation.

Hooks give deterministic guarantees. Prompts give probabilistic compliance.

### 1.6 Decomposition strategy

- **Prompt chaining** (fixed sequential passes) for predictable multi-aspect work.
- **Dynamic adaptive decomposition** for open-ended investigation where subtasks depend on what you find.
- Large single-pass work suffers **attention dilution**: inconsistent depth, missed obvious bugs, contradictory findings on identical code.

### 1.7 Session state

- `--resume <session-name>` continues a named prior conversation.
- `fork_session` creates independent branches from a shared baseline for exploring divergent approaches.
- Resume when prior context is mostly valid.
- Start fresh with an injected structured summary when prior tool results are stale.
- When resuming after code changes, tell the agent explicitly which files changed so re-analysis is targeted.

---

## 2. Canonical Skeletons

### A. The agentic loop

```python
messages = [{"role": "user", "content": user_input}]

while True:
    response = client.messages.create(
        model=MODEL, max_tokens=4096,
        system=SYSTEM_PROMPT, tools=TOOLS, messages=messages,
    )

    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason == "end_turn":
        return final_text(response)          # ← the ONLY normal exit

    if response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)   # hooks fire here
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result.content,
                    "is_error": result.is_error,
                })
        messages.append({"role": "user", "content": tool_results})  # ← feed back
        continue

    # max_tokens / stop_sequence → handle explicitly, don't fall through
```

An iteration cap belongs here only as a **safety backstop with an alert**, never as the primary termination mechanism.

### B. Hub-and-spoke coordinator

```
                    ┌─────────────┐
   user ───────────▶│ COORDINATOR │◀──── all errors, all results
                    └──────┬──────┘
         ┌─────────────────┼─────────────────┐
         │ Task            │ Task            │ Task     ← emitted in ONE response
         ▼                 ▼                 ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │  search  │      │ document │      │ synthesis│   isolated context each;
   │  agent   │      │ analysis │      │  agent   │   prompt carries everything
   └──────────┘      └──────────┘      └──────────┘
         ✗ ──────────── no peer edges ────────────✗
```

Coordinator responsibilities, in order:
**analyze → decompose (partition to avoid overlap) → select subagents → spawn in parallel → aggregate → evaluate for gaps → re-delegate targeted queries → finalize.**

### C. Enforcement decision ladder

```
Does a violation have financial / identity / compliance consequences?
        │
        ├── YES → Is it about ORDERING (X must run before Y)?
        │            ├── YES → prerequisite gate blocking downstream tool
        │            └── NO  → tool-call interception hook (threshold/policy)
        │
        └── NO  → Is the problem tool SELECTION or JUDGMENT calibration?
                     ├── selection  → fix tool descriptions first
                     └── judgment   → explicit criteria + 2–4 few-shot examples
```

### D. Decomposition pattern selection

| Signal in the stem | Pattern |
|---|---|
| Known aspects, predictable structure, "review all N files" | Prompt chaining: per-unit local pass + separate cross-unit integration pass |
| Open-ended, "explore", "add comprehensive tests to legacy code" | Adaptive: map structure → identify high-impact areas → prioritized plan that adapts |
| Two competing approaches from the same baseline | `fork_session` |
| Inconsistent depth / contradictory findings | Split into focused passes (not a bigger model) |

### E. Structured handoff payload

```json
{
  "customer_id": "...",
  "verified": true,
  "issue_summary": "...",
  "root_cause": "...",
  "actions_taken": ["get_customer", "lookup_order"],
  "amount_at_issue": 640.00,
  "policy_blocker": "refund exceeds $500 auto-approval threshold",
  "recommended_action": "approve partial refund + store credit"
}
```

---

## 3. Exam Traps

**Trap 1 — Loop termination by vibes.**
Distractors offer parsing the assistant's text for "task complete," checking whether text content exists, or capping iterations as the primary stop. All wrong. `stop_reason` is the contract.

**Trap 2 — Assumed context inheritance.**
Any option implying a subagent "already has" parent history, or that memory persists between invocations, is wrong. Subagent context is explicit or it doesn't exist.

**Trap 3 — Fake parallelism.**
"Invoke the subagents in consecutive turns" is sequential. Parallel = multiple Task calls in one response.

**Trap 4 — Blaming the wrong layer.**
When the coordinator's log shows narrow decomposition and subagents completed successfully, the answer is the coordinator. Options that tune the search agent, analysis agent, or synthesis agent are distractors describing components working correctly inside a bad assignment.

**Trap 5 — Prompting a deterministic requirement.**
When money, identity, or policy is at stake, "strengthen the system prompt," "add few-shot examples showing correct order," and "add a note in CLAUDE.md" all lose to a programmatic gate. Non-zero failure rate is disqualifying.

**Trap 6 — The inverse trap.**
Not everything gets a hook. When the failure is *calibration* (over-escalating easy cases, picking the wrong similar tool), the proportionate first fix is explicit criteria, better descriptions, or few-shot examples. Infrastructure answers (train a classifier, build a routing layer, deploy an ML model) are over-engineered distractors.

**Trap 7 — Bigger model / bigger context as a fix.**
Attention dilution is not a context-window shortage. Splitting into focused passes is the answer.

**Trap 8 — Confidence and sentiment as control signals.**
Self-reported confidence scores are poorly calibrated (the agent is already wrongly confident). Sentiment does not correlate with case complexity. Both are recurring wrong answers.

**Trap 9 — Error handling extremes.**
Returning empty-as-success suppresses failure. Terminating the whole workflow on one subagent timeout is over-reaction. Generic "service unavailable" hides recovery-relevant context. Correct: structured error context (failure type, attempted query, partial results, alternatives) with local retry for transient issues first.

**Trap 10 — Over-provisioning tools.**
"Give the synthesis agent all the search tools" violates scoped access. Correct shape: a narrow cross-role tool for the high-frequency simple case, coordinator routing for the rest.

**Trap 11 — Procedural coordinator prompts.**
Step-by-step instructions to subagents reduce adaptability. Specify goals and quality criteria instead.

**Trap 12 — Resuming onto stale state.**
After significant code changes, resuming with stale tool results is less reliable than a fresh session seeded with a structured summary. If you do resume, state explicitly which files changed.

---

## 4. Key Facts to Memorize

- `stop_reason: "tool_use"` → continue loop. `stop_reason: "end_turn"` → terminate. Nothing else terminates the loop normally.
- Tool results are appended to conversation history as `tool_result` blocks in a **user-role** message.
- Coordinator `allowedTools` **must include `"Task"`** or it cannot spawn subagents.
- Subagents: isolated context, no automatic inheritance, no shared memory across invocations.
- Parallel subagents = multiple Task calls **in one response**.
- `AgentDefinition` carries description, system prompt, and tool restrictions per subagent type.
- `PostToolUse` = normalize incoming results. Tool-call interception = block outgoing actions.
- Hooks give deterministic guarantees; prompts give probabilistic compliance.
- Prerequisite gate example: block `process_refund` until `get_customer` returns a verified customer ID.
- Threshold example: block refunds over **$500**, redirect to human escalation.
- Roughly **4–5 tools per agent**; ~18 degrades selection reliability.
- Hub-and-spoke rationale, memorize all three: **observability, consistent error handling, controlled information flow.**
- Prompt chaining = predictable multi-aspect. Dynamic decomposition = open-ended investigation.
- Large reviews: per-file local pass + separate cross-file integration pass.
- `--resume <session-name>` = continue named session. `fork_session` = divergent branches from shared baseline.
- Escalation handoffs need: customer ID, root cause, amount, recommended action. The human has no transcript.
- Coordinator prompts specify **goals and quality criteria**, not procedures.

---

## 5. Exam-Style Questions

### Q1
Your support agent's loop sometimes ends before the work is finished. The implementation terminates when the assistant's response contains any text block, on the assumption that Claude only writes prose when it's done. What is the correct fix?

- **A.** Terminate only when `stop_reason` is `"end_turn"`; continue executing tools and appending results while it is `"tool_use"`.
- **B.** Instruct Claude in the system prompt to emit `TASK_COMPLETE` as its final line, and terminate on that marker.
- **C.** Set a maximum of 15 iterations and return whatever the agent has produced at that point.
- **D.** Terminate when two consecutive iterations request the same tool with identical inputs.

**Correct Answer: A.**
Claude routinely emits text blocks alongside tool_use blocks while reasoning mid-task, so text presence is not a completion signal. `stop_reason` is the explicit, model-provided contract for loop control. B substitutes a parsed natural-language signal for a structured one and inherits the model's non-zero compliance failure rate. C makes an arbitrary cap the primary stopping mechanism rather than a safety backstop. D is a loop-detection heuristic for a different problem and would not fire on premature termination.

### Q2
Your research coordinator is configured with `allowedTools: ["WebSearch", "Read", "Write"]` and a system prompt instructing it to delegate web research and document analysis to specialist subagents. In production it performs all research itself and never delegates. What is the root cause?

- **A.** The subagent `AgentDefinition` entries lack sufficiently detailed descriptions.
- **B.** The coordinator's `allowedTools` omits `"Task"`, so it has no mechanism to spawn subagents.
- **C.** The subagents are not inheriting the coordinator's conversation history.
- **D.** The system prompt needs few-shot examples demonstrating delegation.

**Correct Answer: B.**
Spawning a subagent is a tool call against the Task tool. Without `"Task"` in `allowedTools`, delegation is mechanically impossible regardless of prompt quality, and the model falls back to the tools it does have. A and D try to improve a decision the coordinator cannot execute. C describes expected behavior (subagents never inherit context automatically) and is unrelated to whether delegation occurs.

### Q3
Your agent receives order data from three MCP servers. One returns `created_at` as a Unix epoch integer, one as an ISO 8601 string, and one as `"status": 3` with a numeric code. The agent frequently miscalculates return-window eligibility. What is the most appropriate fix?

- **A.** Add a `PostToolUse` hook that normalizes timestamps and status codes into a single canonical format before results reach the model.
- **B.** Document each server's format convention in the system prompt and instruct Claude to convert as needed.
- **C.** Add a tool-call interception hook that rewrites the request parameters before each tool executes.
- **D.** Add few-shot examples showing correct epoch-to-date conversion for each format.

**Correct Answer: A.**
This is data arriving *from* tools, so the interception point is `PostToolUse` — transforming results before the model processes them. Normalization is a deterministic transformation and belongs in code. B and D push arithmetic and format inference onto the model probabilistically, which is exactly the current failure. C intercepts the wrong direction: the outgoing call is fine; the returned payload is the problem.

### Q4
Your research system consistently produces reports with heavy redundancy. Three search subagents run in parallel on "renewable energy storage economics," and roughly 60% of their cited sources are identical. Each subagent performs correctly against its instructions. What should you change?

- **A.** Deduplicate sources in the synthesis agent before report generation.
- **B.** Reduce to a single search subagent to eliminate overlap entirely.
- **C.** Have the coordinator partition scope across subagents, assigning distinct subtopics or source types to each.
- **D.** Add a shared cache the search subagents consult before issuing queries.

**Correct Answer: C.**
Duplication originates in the coordinator's decomposition, which assigned overlapping scope. Partitioning by subtopic or source type addresses the cause. A treats the symptom downstream and wastes tokens and latency already spent. B discards parallelism and coverage to solve a partitioning problem. D introduces shared state between subagents, which breaks context isolation and the hub-and-spoke communication model.

### Q5
A single-pass review of a 14-file pull request produces detailed feedback on some files, superficial comments on others, and flags a pattern as a defect in one file while approving identical code in another. Which restructuring addresses the cause?

- **A.** Upgrade to a model with a larger context window so all files fit comfortably.
- **B.** Run per-file passes for local issues, then a separate integration pass for cross-file data flow.
- **C.** Run the same review three times and report only findings that appear in at least two runs.
- **D.** Require developers to open pull requests of no more than four files.

**Correct Answer: B.**
The symptoms describe attention dilution across many units in one pass. Per-file passes guarantee consistent depth; a dedicated integration pass recovers the cross-file issues that per-file analysis would miss. A misdiagnoses an attention-quality problem as a capacity problem. C suppresses genuine bugs that are only caught intermittently. D shifts cost onto developers without improving the review system.

### Q6
You spent a long session having Claude map a legacy billing module. Overnight, another team refactored four of the files you analyzed and renamed two modules. You need to continue the investigation today. What is the most reliable approach?

- **A.** Resume the session with `--resume` and proceed; the agent will re-read files as needed.
- **B.** Resume the session and tell the agent explicitly which files changed so it re-analyzes those targets.
- **C.** Start a fresh session seeded with a structured summary of the prior findings, then re-explore the changed modules.
- **D.** Use `fork_session` to branch from the prior analysis and continue on the branch.

**Correct Answer: C.**
Prior tool results are now stale for a meaningful portion of the analysis, including renamed modules that invalidate earlier structural conclusions. A fresh session with an injected structured summary keeps the durable findings while discarding stale state. B is the correct technique when prior context is *mostly* valid and only a small, identifiable set of files changed; here the renames undermine the baseline. A silently reasons over stale results. D forks the stale baseline and propagates the problem into both branches.

### Q7
Your synthesis subagent produces fluent reports, but roughly a third of claims cannot be traced to a source. The search and document-analysis subagents each return a prose summary of their findings. What should you change first?

- **A.** Instruct the synthesis agent in its system prompt to always cite its sources.
- **B.** Have upstream subagents return structured findings where each claim is paired with source URL, document name, excerpt, and publication date, and require synthesis to preserve those mappings.
- **C.** Give the synthesis agent a web search tool so it can re-locate sources for unattributed claims.
- **D.** Add a validation pass that rejects any report containing uncited claims.

**Correct Answer: B.**
Attribution is lost upstream: prose summaries compress away claim-to-source mappings, so by the time synthesis runs the information no longer exists. Structured claim-source mappings preserve provenance through the handoff. A instructs the agent to cite data it was never given. C over-provisions the synthesis agent and asks it to reconstruct attribution speculatively. D detects the failure without preventing it, and gives synthesis no path to comply.

### Q8 (Select TWO)
Your team proposes letting the document-analysis subagent send its output directly to the synthesis subagent, bypassing the coordinator, to cut latency. Which two consequences most directly argue against this?

- **A.** Synthesis would no longer be able to parse the analysis output format.
- **B.** Errors from document analysis would no longer reach a single place with consistent handling and recovery logic.
- **C.** The coordinator loses observability into inter-agent information flow, making failures harder to diagnose.
- **D.** Subagents would begin inheriting each other's full conversation history.
- **E.** The Task tool cannot be used to spawn more than one subagent per response.

**Correct Answers: B and C.**
Hub-and-spoke exists for observability, consistent error handling, and controlled information flow; a peer edge removes the first two directly. A is unfounded — the format is whatever the agents are defined to produce. D is false: subagents never inherit context automatically, which is precisely why the coordinator must route information explicitly. E is false and inverts the actual guidance, since parallel spawning depends on emitting multiple Task calls in one response.

---

## Readiness Check

If you can look at any Domain 1 stem and immediately classify it as **loop control / enforcement layer / context passing / decomposition / error propagation / session state**, you're where you need to be.

The distractor patterns are consistent across the whole exam:
1. Probabilistic fixes for deterministic requirements
2. Infrastructure for prompt-level problems
3. Downstream patches for upstream causes
