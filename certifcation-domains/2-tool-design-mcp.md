
# Claude Certified Architect – Foundations (CCAR-F)
## Domain 2: Tool Design & MCP Integration

**Weight: 18% → roughly 11 of 60 scored items.** Primary domain in Scenario 1 (Customer Support Resolution Agent), Scenario 3 (Multi-Agent Research System), and Scenario 4 (Developer Productivity). Since 4 of 6 scenarios are drawn at random, expect this domain in 2–3 of your 4 scenarios.

### Meta-heuristic for answering Domain 2 items

Nearly every Domain 2 item is a "what's the most effective fix" question, and the correct answer almost always follows one ordering:

**Fix the root cause at the lowest layer first.**
Tool description → tool naming/scoping → tool splitting → structured error metadata → architectural change (routing layers, classifiers, consolidation).

- Answers that reach for a classifier, an ML model, a keyword-parsing router, or speculative caching are distractors.
- Answers that rely on the model "trying harder" (system prompt exhortations, confidence self-reports) are also distractors.

---

# Task 2.1 — Tool Interfaces, Descriptions, and Boundaries

## Core Concept

The model never sees your implementation. At selection time it sees only **tool name + description + input schema**. Therefore the description *is* the routing logic. Selection failures among similar tools are a description-quality defect, not a model-capability defect.

A description that supports reliable selection contains five things:

1. **Purpose** in one line
2. **Input formats accepted**, with concrete examples of each
3. **Example queries** that should route here
4. **Edge cases** and what is returned when nothing matches
5. **Boundary statement**: when to use this *instead of* the named sibling tool

Item 5 is the one candidates forget, and it is the one the exam tests. Two tools each described accurately but in isolation will still misroute, because neither tells the model where the seam is.

Three escalating remedies, in the order the exam expects you to reach for them:

| Effort | Remedy | Use when |
|---|---|---|
| Low | **Expand descriptions** with inputs, examples, edge cases, boundaries | Tools are genuinely distinct; descriptions are thin |
| Medium | **Rename + rescope** to remove functional overlap | Names imply overlapping domains (`analyze_content` vs `analyze_document`) |
| Higher | **Split a generic tool** into purpose-specific tools with defined I/O contracts | One tool does several different jobs with one vague contract |

A fourth factor sits outside the tools entirely: **system prompt keyword sensitivity**. Wording like "always thoroughly analyze every document you retrieve" creates an unintended association with any tool named `analyze_document`, and that association can override well-written descriptions. When descriptions have already been fixed and misrouting persists, audit the system prompt.

## Canonical Skeleton

```
name: extract_web_results

description: >
  Extracts structured findings from web search result payloads returned by
  search_web. Accepts the raw result array (title, url, snippet, fetched_at).

  Use this for web-sourced content only. For PDFs, internal documents, or
  uploaded files, use analyze_document instead.

  Example queries that route here:
    - "pull the key claims out of these search hits"
    - "summarize what these 8 articles say about X"

  Edge cases: returns {results: [], reason: "no_extractable_content"} when
  snippets are empty or paywalled. This is a success, not an error.

  Returns: array of {claim, evidence_excerpt, source_url, published_date}
```

**Splitting pattern (memorize the canonical example):**

`analyze_document` (generic, ambiguous contract) splits into:

- `extract_data_points` → structured fields out of a document
- `summarize_content` → prose condensation
- `verify_claim_against_source` → boolean/evidence check of a specific assertion

Each gets its own input schema and output contract, which removes both selection ambiguity and output-shape inconsistency in one move.

## Exam Traps

- **"Add 5–8 few-shot examples to the system prompt."** Plausible and wrong when descriptions are thin. It adds permanent token overhead on every turn and leaves the root cause in place. Few-shot is the right answer for *ambiguous judgment calls* (Domain 4), not for *undifferentiated tools*.
- **"Build a routing layer that parses keywords and pre-selects the tool."** Over-engineered; it discards the model's language understanding and breaks on phrasing you did not anticipate.
- **"Consolidate the two tools into one `lookup_entity` that figures out the backend internally."** A legitimate architecture, which is what makes it a good distractor. It is never the correct *first step* when the stated problem is inadequate descriptions.
- **"Switch to a larger model."** Never correct in this domain.
- Assuming a longer description is automatically a better one. The exam rewards *differentiating* content (boundaries, contrasts), not volume.

## Key Facts to Memorize

- Tool descriptions are the primary mechanism LLMs use for tool selection.
- Minimal descriptions → unreliable selection among similar tools.
- Good description = purpose + input formats + example queries + edge cases + boundary vs siblings.
- Overlapping descriptions cause misrouting; rename to eliminate functional overlap.
- One generic tool serving three jobs → split into three purpose-specific tools with defined I/O contracts.
- System prompt keywords can create unintended tool associations that override descriptions.

---

# Task 2.2 — Structured Error Responses for MCP Tools

## Core Concept

MCP signals failure with the **`isError` flag**. But the flag alone only tells the agent *that* something failed. To recover intelligently, the agent needs to know *what kind* of failure it was, because each kind implies a different next action:

| Category | Example | Retryable | Correct agent behavior |
|---|---|---|---|
| **Transient** | Timeout, 503, rate limit | **Yes** | Retry, possibly with backoff |
| **Validation** | Malformed order ID, missing required field | No | Fix the input or ask the user |
| **Business** | Refund exceeds $500 policy cap; item outside return window | No | Explain to the customer / escalate |
| **Permission** | Agent not authorized for this account | No | Escalate to human |

A uniform `"Operation failed"` collapses all four into a guess, which produces the two classic failure modes the exam tests: **wasted retries on permanently-failing calls**, and **the agent reporting a policy decision to the customer as a system outage**.

Business errors need an extra field the others do not: a **customer-friendly explanation string**, because the agent will relay it verbatim to an end user. `POLICY_VIOLATION_REFUND_CAP_EXCEEDED` is unusable in a support conversation.

**Second core distinction: access failure vs. valid empty result.** A search that completes and finds zero matches is a *successful* call with an empty result set. A search that times out is a failure. If a tool reports both as "no results," the coordinator cannot tell whether to retry, try an alternate source, or accept that the answer is genuinely absent. This distinction appears in both Domain 2 and Domain 5.

**Third: local recovery before propagation.** A subagent should handle transient failures itself. It escalates to the coordinator only what it cannot resolve, and when it does, it includes what it attempted and any partial results it did gather.

## Canonical Skeleton

```json
{
  "isError": true,
  "errorCategory": "business",
  "isRetryable": false,
  "message": "This refund is $840, which exceeds the $500 limit an automated agent can approve. A specialist can authorize it.",
  "attempted": "process_refund(order_id=A-2291, amount=840.00)",
  "partialResults": { "order_verified": true, "customer_id": "C-5512" },
  "alternatives": ["escalate_to_human"]
}
```

And the empty-result counterpart, which is **not** an error:

```json
{
  "isError": false,
  "results": [],
  "reason": "no_matching_records",
  "query": "orders for C-5512 after 2026-01-01"
}
```

## Exam Traps

- **Catching the timeout and returning an empty result marked successful.** Silent error suppression. It guarantees incomplete output with no signal that anything went wrong. Always wrong.
- **Retry with exponential backoff, then return a generic `"search unavailable"` status.** The retry logic is fine; the generic final status is the defect, because it strips the context the coordinator needs.
- **Propagating the exception to a top-level handler that terminates the whole workflow.** One subagent failure should not kill a pipeline that could proceed on partial results.
- **Treating retries as universally useful.** Retrying a validation or business error can never succeed, and burns latency and tokens.
- **Assuming a human-readable message is enough.** The exam wants *machine-readable* `errorCategory` and `isRetryable` alongside prose, because the agent branches on structure.

## Key Facts to Memorize

- MCP uses the **`isError`** flag to communicate tool failure.
- Four categories: **transient / validation / business / permission**. Only transient is retryable.
- Structured error payload: `errorCategory`, `isRetryable` (or `retriable: false`), human-readable description, what was attempted, partial results, alternatives.
- Business rule violations need a **customer-friendly explanation** the agent can relay.
- **Empty result ≠ access failure.** Report them differently.
- Subagents recover locally from transient failures; they propagate only unresolvable errors, with partial results attached.

---

# Task 2.3 — Tool Distribution and `tool_choice`

## Core Concept

**Tool count is an independent variable in selection reliability.** The guide's benchmark: an agent with ~18 tools selects less reliably than one with 4–5, because every additional tool increases decision complexity. And an agent holding tools outside its specialization will eventually use them: a synthesis agent given web search tools will start doing its own searching, which breaks separation of concerns and observability.

The design principle is **least privilege, scoped by role**. Each subagent gets only what its role requires.

The exam's favorite nuance is the **scoped cross-role tool**. When a role has a high-frequency need that sits just outside its tool set, the answer is not "give it the full tool set" and not "route 100% through the coordinator." It is a *narrow, constrained* tool covering the common case, with complex cases still routed through the coordinator. Canonical example: the synthesis agent gets `verify_fact` for simple date/name/statistic lookups (the 85% case), while deeper investigations continue through the coordinator to the web search agent (the 15%).

Related: **replace generic tools with constrained alternatives**. `fetch_url` can retrieve anything; `load_document` validates that the URL is a document. The constrained version narrows the failure surface and sharpens selection.

## `tool_choice` — the three settings

| Setting | Guarantee | Use when |
|---|---|---|
| `"auto"` | Model may call a tool **or** return plain text | Normal conversational agents |
| `"any"` | Model **must** call some tool, but chooses which | You need guaranteed structured output and there are several valid schemas / the document type is unknown |
| `{"type": "tool", "name": "extract_metadata"}` | Model must call **that specific** tool | You need a specific tool to run first |

Critical mechanic: **`tool_choice` applies to the current request, not to a sequence.** You cannot encode "call A, then B, then C" in it. You force the first call, then handle subsequent steps in follow-up turns.

## Canonical Skeleton

```
Coordinator          → allowedTools: [Task, delegate/routing tools]
Web search agent     → search_web, load_document
Document analysis    → extract_data_points, verify_claim_against_source
Synthesis agent      → verify_fact   ← scoped cross-role tool, narrow contract
                       (complex verification → back through coordinator)
```

```python
# Guarantee metadata extraction runs before any enrichment tool
turn_1 = client.messages.create(
    tools=[extract_metadata, enrich_company, enrich_contacts],
    tool_choice={"type": "tool", "name": "extract_metadata"},
    messages=history,
)
# turn_2 onward: tool_choice="auto" (or "any"), with turn_1's result appended
```

## Exam Traps

- **"Give the synthesis agent access to all web search tools so it never needs a round trip."** Over-provisioning; violates separation of concerns. This is the distractor paired with the `verify_fact` answer.
- **"Batch all verification needs and send them to the coordinator at the end of the pass."** Sounds efficient, creates blocking dependencies, because later synthesis steps may depend on facts verified earlier.
- **"Have the search agent proactively cache extra context around each source in case synthesis needs it."** Speculative caching cannot reliably predict what will be needed.
- **Using `"any"` to force a *particular* tool.** `"any"` only guarantees *a* tool call. Forcing a named tool requires `{"type": "tool", "name": "..."}`.
- **Expecting `tool_choice` to enforce ordering across a whole workflow.** It governs one request. Cross-turn ordering guarantees come from hooks and prerequisite gates (Domain 1).
- **Using `"auto"` in an extraction pipeline**, then being surprised the model sometimes returns prose instead of calling the schema tool.

## Key Facts to Memorize

- ~18 tools degrades selection reliability; 4–5 per agent is the healthy range.
- Agents misuse tools outside their specialization.
- Scoped cross-role tool for high-frequency simple needs; coordinator routing for complex ones.
- Replace generic tools (`fetch_url`) with validated, constrained ones (`load_document`).
- `auto` = may call a tool. `any` = must call a tool, model picks. `{"type":"tool","name":X}` = must call X.
- `tool_choice` is per-request; sequences are handled in follow-up turns.

---

# Task 2.4 — MCP Server Integration

## Core Concept

**Scoping mirrors the CLAUDE.md hierarchy, and the exam tests the same failure.**

| Scope | File | Shared via version control? | Use for |
|---|---|---|---|
| **Project** | `.mcp.json` (repo root) | **Yes** | Shared team tooling |
| **User** | `~/.claude.json` | **No** | Personal / experimental servers |

The signature diagnostic item: a teammate clones the repo and does not have the Jira tools. Root cause is almost always that the server was configured at user scope on one developer's machine instead of project scope in the repo.

**Environment variable expansion** (`${GITHUB_TOKEN}`, `${JIRA_API_TOKEN}`) lets you commit `.mcp.json` to version control without committing secrets. Each developer supplies the variable in their own environment.

**Discovery model:** tools from *all* configured MCP servers are discovered at connection time and are available **simultaneously**. You do not enable a server per task. A project server and a personal server coexist in the same session. This is exactly why Task 2.3's tool-count discipline matters: adding servers casually is how an agent ends up with 18 tools.

**Tools vs. Resources.** Tools are for **actions**. Resources expose **content catalogs**: issue summaries, documentation hierarchies, database schemas. The purpose of a resource is to give the agent visibility into what data exists *without spending exploratory tool calls discovering it*. When the symptom is "the agent makes many list/search calls just to find out what's available," the answer is a resource, not a better tool.

**Adoption problem.** An MCP tool with a thin description loses to a built-in. If your MCP code-search server is more capable than `Grep` but the agent keeps reaching for `Grep`, the fix is to enhance the MCP tool's description to spell out its capabilities and output, so the model can see why it is the better choice. Same root cause as Task 2.1, applied across the built-in/MCP boundary.

**Buy vs. build.** Use existing community MCP servers for standard integrations (Jira, GitHub). Reserve custom servers for team-specific workflows that nothing off the shelf covers.

## Canonical Skeleton

```json
// .mcp.json — committed to the repo, shared by the whole team
{
  "mcpServers": {
    "jira": {
      "command": "npx",
      "args": ["-y", "@community/mcp-server-jira"],
      "env": {
        "JIRA_BASE_URL": "${JIRA_BASE_URL}",
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}"
      }
    }
  }
}
```

```json
// ~/.claude.json — personal, not shared
{
  "mcpServers": {
    "my-scratch-server": { "command": "node", "args": ["./experiments/server.js"] }
  }
}
```

Both sets of tools are live in the same session.

## Exam Traps

- **Putting a team server in `~/.claude.json`** and expecting teammates to get it.
- **Hardcoding the token in `.mcp.json`** rather than using `${VAR}` expansion. Any answer option that commits a literal secret is wrong.
- **"Have each developer enable the server manually per project"** as a remedy for a sharing problem. Project scope already solves it.
- **"Write a custom Jira MCP server"** when a community server exists. Reserve custom builds for team-specific workflows.
- **Believing servers must be activated or swapped per task.** All configured servers' tools are discovered at connection and available at once.
- **Exposing a catalog through repeated tool calls** instead of as a resource.
- **"Disable the built-in Grep tool"** to force MCP tool adoption. The fix is description quality, not removing capability.

## Key Facts to Memorize

- `.mcp.json` = project scope, version controlled. `~/.claude.json` = user scope, personal.
- `${ENV_VAR}` expansion in `.mcp.json` keeps credentials out of the repo.
- All configured servers' tools are discovered at connection time, available simultaneously.
- **Resources = content catalogs** (reduce exploratory calls). **Tools = actions.**
- Thin MCP descriptions lose to built-ins like Grep; enhance the description to fix adoption.
- Community servers for standard integrations; custom servers for team-specific workflows.

---

# Task 2.5 — Built-in Tools (Read, Write, Edit, Bash, Grep, Glob)

## Core Concept

Two pairs, each with a clean split:

| Tool | Searches | Example |
|---|---|---|
| **Grep** | **Contents** of files | Find all callers of `processRefund`; locate an error message string; find import statements |
| **Glob** | **File paths / names** | `**/*.test.tsx`; all `.tf` files; every `README.md` |

| Tool | Operation |
|---|---|
| **Read / Write** | Full-file load and full-file replace |
| **Edit** | Targeted modification anchored on **unique** text |

**The Edit failure mode is a named exam fact:** when the anchor text appears more than once, `Edit` cannot identify the target and fails. The prescribed fallback is **Read + Write**: load the full file, construct the modified content, write it back.

**Exploration strategy** is tested as much as tool selection. The correct pattern is **incremental**: start with `Grep` to find entry points, then `Read` to follow imports and trace flows. Reading every file upfront to "build understanding" exhausts the context window and is always the wrong answer.

**Wrapper module tracing** has a specific two-step recipe: first identify all exported names from the wrapper, then search for each name across the codebase. Searching for the wrapper's own module name finds only the import sites, not the actual usages.

## Canonical Skeleton

```
Understand an unfamiliar codebase:
  1. Glob  → map the shape ("**/*.py", "src/**/handlers/*")
  2. Grep  → find entry points and key symbols
  3. Read  → follow imports outward from those entry points only
  4. (verbose phases → delegate to Explore subagent, Domain 3/5)

Modify a file:
  Edit         → if unique anchor text exists
  Read + Write → if the anchor is ambiguous or the change is structural
```

## Exam Traps

- **Swapping Grep and Glob.** The single most common Domain 2 factual error. "Find all files named X" = Glob. "Find all files *containing* X" = Grep.
- **Reaching for `Bash` with `grep`/`find`/`sed`** when a built-in tool does the job. Distractor options frequently offer shell equivalents.
- **"Read all 40 files in the module first, then analyze."** Context exhaustion; contradicts incremental exploration.
- **"Retry Edit with more surrounding context."** The documented fallback is Read + Write.
- **"Use Glob to find every file that calls the function."** Glob cannot see file contents.
- Treating `Read` as the discovery tool. Discovery is Grep/Glob; Read is for following a trail you already found.

## Key Facts to Memorize

- **Grep = content. Glob = paths/filenames.**
- Read/Write = whole file; Edit = targeted, requires **unique** matching text.
- Edit fails on non-unique matches → **fallback to Read + Write**.
- Build understanding incrementally: Grep for entry points → Read to follow imports. Never read everything upfront.
- Trace wrapper usage by enumerating exported names first, then searching each name.

---

# Exam-Style Questions

## Q1 (Task 2.1)

*Multi-Agent Research System.* Your `analyze_document` tool is invoked for three different purposes: pulling out specific figures, producing summaries, and checking whether a claim is supported by a source. Its output format varies unpredictably across these uses, and downstream synthesis frequently receives prose where it expected structured fields. What is the most effective change?

A. Add a `mode` parameter to `analyze_document` with values `extract`, `summarize`, and `verify`.
B. Split `analyze_document` into `extract_data_points`, `summarize_content`, and `verify_claim_against_source`, each with its own input and output contract.
C. Add a post-processing step that reformats `analyze_document` output into a consistent schema before synthesis consumes it.
D. Instruct the document analysis agent in its system prompt to always return JSON regardless of the task.

**Answer: B.** A generic tool serving three distinct purposes with one loose contract produces exactly this symptom. Splitting into purpose-specific tools fixes both output inconsistency and selection ambiguity in one move, because each tool now has a defined I/O contract. **A** keeps the single ambiguous contract and relocates the ambiguity into a parameter the model must also select correctly. **C** treats the symptom downstream and leaves the tool undefined. **D** is prompt-based and probabilistic where a schema-level guarantee is available.

## Q2 (Task 2.1)

You rewrote both `extract_web_results` and `analyze_document` with detailed descriptions covering inputs, examples, edge cases, and explicit boundaries. Misrouting dropped but did not disappear: the agent still sends web search payloads to `analyze_document` roughly 6% of the time. Your system prompt includes the line "Thoroughly analyze every document and source you retrieve before drawing conclusions." What should you investigate first?

A. The input schemas, which may still accept overlapping types.
B. The system prompt, whose keyword phrasing may be creating an unintended association with `analyze_document`.
C. The order in which tools are declared in the tools array.
D. Whether the model needs few-shot examples of correct routing.

**Answer: B.** System prompt wording is keyword-sensitive and can create tool associations that override otherwise well-written descriptions. "Analyze every document and source" maps a phrase containing both *analyze* and *document* onto every retrieval, including web results. The guide names reviewing system prompts for keyword-sensitive instructions as a distinct skill. **A** is worth tightening eventually but does not explain a residual bias toward one specific tool. **C** is not a documented selection factor. **D** adds token overhead without addressing the identified conflict.

## Q3 (Task 2.2) — Select TWO

*Customer Support Resolution Agent.* When a refund exceeds the $500 policy cap, `process_refund` returns `isError: true` with the message "Operation failed." Logs show the agent retries the call three times, then tells the customer the system is experiencing technical difficulties. Select TWO changes that address this.

A. Return `errorCategory: "business"` with `isRetryable: false`.
B. Include a customer-friendly explanation string describing the policy limit and the path to approval.
C. Reduce the agent's retry count from three to one.
D. Move the refund cap check into the system prompt so the agent never attempts the call.
E. Return the error as a successful response with `refund_approved: false` so the agent stops retrying.

**Answer: A and B.** The two defects are that the agent cannot tell a permanent policy decision from a transient outage, and that it has no accurate language to give the customer. `errorCategory` plus `isRetryable: false` stops the futile retries; the customer-friendly string lets the agent explain what actually happened. **C** reduces wasted calls from three to one without fixing the misdiagnosis, and the customer still hears "technical difficulties." **D** is prompt-based enforcement of a financial rule, which has a non-zero failure rate; deterministic caps belong in a tool-call interception hook (Domain 1.5). **E** disguises a failure as a success, the silent-suppression anti-pattern.

## Q4 (Task 2.2)

*Multi-Agent Research System.* Your document analysis subagent returns `{"results": []}` in two very different situations: when the source repository is unreachable, and when the query ran successfully but matched nothing. The coordinator treats both identically and moves on. What should you change?

A. Have the subagent retry indefinitely until it gets a non-empty result.
B. Distinguish the two cases in the response, returning a structured error with failure type, attempted query, and partial results for access failures, while reporting genuine empty matches as successful.
C. Have the coordinator re-run every empty result once to determine which case it was.
D. Have the subagent return an error for both cases so the coordinator always investigates.

**Answer: B.** Access failures and valid empty results demand opposite coordinator responses: retry or find an alternative source, versus accept that the answer is absent and annotate coverage. The structured error payload carries exactly the fields the coordinator needs to choose. **A** never terminates and ignores that a genuinely empty result is a legitimate outcome. **C** doubles the cost of every legitimate empty result to recover information the subagent already had. **D** inverts the problem: the coordinator now investigates successful queries and cannot distinguish real coverage gaps from outages.

## Q5 (Task 2.3)

*Developer Productivity.* Your agent has accumulated 18 tools across four MCP servers plus the built-ins. Tool selection accuracy has dropped noticeably, with the agent choosing plausible-but-wrong tools on multi-step tasks. What is the most effective structural change?

A. Reorder the tools array so the most frequently used tools appear first.
B. Decompose the agent into role-specialized subagents, each scoped to the 4–5 tools its role requires, with a coordinator routing between them.
C. Write longer descriptions for all 18 tools.
D. Add a system prompt section listing all 18 tools with one-line summaries of when to use each.

**Answer: B.** Decision complexity scales with tool count; scoping each agent to the tools its role needs is the documented remedy. **A** is not a documented selection factor. **C** improves differentiation but does not reduce the number of candidates the model must weigh, and 18 verbose descriptions also consume significant context. **D** duplicates description content into the system prompt, adding tokens while leaving the same 18-way decision.

## Q6 (Task 2.3)

*Structured Data Extraction.* Your enrichment pipeline requires `extract_metadata` to run before any enrichment tool, because enrichment depends on the document type it returns. Occasionally the model calls `enrich_company` first and produces wrong results. Which configuration guarantees correct ordering?

A. Set `tool_choice: "any"` so the model is required to call a tool.
B. Set `tool_choice: {"type": "tool", "name": "extract_metadata"}` on the first request, then process the enrichment steps in follow-up turns.
C. Set `tool_choice: "auto"` and add "always call extract_metadata first" to the system prompt.
D. Declare `extract_metadata` as the only tool in the array and add the enrichment tools after the model responds.

**Answer: B.** Forced tool selection guarantees the named tool is what gets called on that request, and the remaining steps proceed on subsequent turns because `tool_choice` governs one request rather than a sequence. **A** guarantees *a* tool call, not *which* one, so `enrich_company` remains reachable on the first turn. **C** is probabilistic, which is what is already failing. **D** technically works but is a clumsier expression of the same idea, requiring you to rebuild the tools array between turns rather than using the purpose-built parameter.

## Q7 (Task 2.3)

Your extraction service handles several document types, each with its own schema tool. The type is unknown until the model reads the document. In about 7% of requests the model returns a conversational summary instead of calling any extraction tool, which breaks the downstream parser. What is the correct configuration?

A. `tool_choice: "auto"` with a stronger system prompt instruction.
B. `tool_choice: "any"`.
C. `tool_choice: {"type": "tool", "name": "extract_generic"}`.
D. Retry the request when the response contains no `tool_use` block.

**Answer: B.** `"any"` guarantees the model calls a tool while still letting it choose the schema appropriate to the document type it finds, which is exactly the requirement. **A** leaves the text-response path open. **C** forces one schema and defeats the purpose of having type-specific schemas. **D** is a workaround that doubles cost and latency on 7% of requests when a parameter eliminates the failure outright.

## Q8 (Task 2.3)

Your synthesis agent needs to verify claims while combining findings. Currently it returns control to the coordinator, which invokes the web search agent and re-invokes synthesis, adding 2–3 round trips and 40% latency. Evaluation shows 85% of verifications are simple fact-checks (dates, names, statistics) and 15% need deeper investigation. What is the most effective approach?

A. Give the synthesis agent a scoped `verify_fact` tool for simple lookups, with complex verification still routed through the coordinator.
B. Give the synthesis agent the full web search tool set.
C. Have synthesis accumulate all verification needs and send them to the coordinator as one batch at the end of its pass.
D. Have the web search agent cache extra context around every source during initial research.

**Answer: A.** Least privilege applied to the 85% case: a narrow tool that covers the common need without granting general-purpose search, while the existing coordination pattern handles the complex minority. **B** over-provisions the agent and invites cross-specialization misuse. **C** creates blocking dependencies, since later synthesis steps may depend on facts verified earlier. **D** relies on predicting what will need verification, which is not reliably possible.

## Q9 (Task 2.4)

*Developer Productivity.* You configured a Jira MCP server and it works on your machine. A new team member clones the repo and reports that no Jira tools are available. Your configuration lives in `~/.claude.json` with the API token written inline. What should you do?

A. Have the teammate copy your `~/.claude.json` entry into their own home directory.
B. Move the server configuration into a project-scoped `.mcp.json` committed to the repository, replacing the inline token with `${JIRA_API_TOKEN}`.
C. Document the setup steps in CLAUDE.md so each developer configures the server themselves.
D. Move the configuration to `.mcp.json` with the token inline so the setup works immediately on clone.

**Answer: B.** Project-scoped `.mcp.json` is the mechanism for shared team tooling, and environment variable expansion keeps the credential out of version control while still sharing the configuration. **A** reproduces the problem for every future hire and distributes your personal credential. **C** turns a solved problem into manual per-developer setup that will drift. **D** gets the scope right and commits a live secret, which is disqualifying regardless of convenience.

## Q10 (Task 2.4)

Your agent repeatedly makes exploratory calls to list projects, then list boards, then list issue types, before it can act on a user's request. These discovery calls consume a significant share of the context window on every session. Which MCP capability addresses this?

A. Add a `list_everything` tool that returns all catalogs in one call.
B. Expose the issue summaries, board structures, and schema hierarchies as MCP **resources**.
C. Cache the discovery results in the system prompt and refresh them weekly.
D. Increase the agent's context window allocation.

**Answer: B.** Resources exist to expose content catalogs so the agent has visibility into available data without spending exploratory tool calls on discovery. This is the textbook use case. **A** is still a tool call, still returns into the conversation, and now returns more at once. **C** hardcodes data that changes and bloats every request regardless of relevance. **D** is not an available lever and does not address the waste.

## Q11 (Task 2.4)

You deployed an MCP server exposing a semantic code search tool that outperforms text matching on your monorepo. In practice the agent still reaches for the built-in `Grep` almost every time. The MCP tool's description reads "Searches the codebase." What is the most effective fix?

A. Enhance the MCP tool's description to detail its capabilities, when it outperforms text search, its input format, and what it returns.
B. Remove `Grep` from the agent's allowed tools.
C. Add a system prompt instruction to prefer MCP tools over built-in tools.
D. Rename the MCP tool to `grep_advanced` so it sorts near the built-in.

**Answer: A.** Task 2.1's principle applied across the built-in/MCP boundary: a thin description gives the model no reason to prefer the MCP tool over a built-in whose behavior it understands well. Spelling out capabilities and outputs is the documented remedy. **B** removes a genuinely useful tool and will hurt on cases where literal matching is correct. **C** is a blanket rule that will misfire whenever a built-in is the better choice. **D** does not change what the model knows about the tool.

## Q12 (Task 2.5)

*Developer Productivity.* An engineer asks the agent to find every place a deprecated `calculateTax` function is called, then to update all test files that exercise it. Which tool selection is correct?

A. `Glob` for `**/calculateTax*`, then `Grep` for the test files.
B. `Grep` for `calculateTax` to find call sites, then `Glob` for `**/*.test.*` to enumerate test files.
C. `Read` on every file in the repository, filtering for both in memory.
D. `Bash` running `grep -r calculateTax .` followed by `find . -name "*.test.*"`.

**Answer: B.** Grep searches file *contents*, which is what finding call sites requires. Glob matches file *paths*, which is what enumerating test files requires. **A** inverts both. **C** exhausts the context window and contradicts incremental exploration. **D** reimplements both built-ins through the shell with no benefit; the exam expects the purpose-built tools.

## Q13 (Task 2.5)

The agent attempts to change a logging call inside `OrderService.ts` using `Edit`, anchoring on `logger.info("processing")`. The edit fails because that exact string appears in six methods in the file. What should it do?

A. Retry `Edit` with a longer anchor including more surrounding lines until the match is unique.
B. Use `Read` to load the full file, then `Write` the modified contents.
C. Use `Bash` with `sed` and a line-number address.
D. Split the file into smaller modules so future edits have unique anchors.

**Answer: B.** Read + Write is the documented fallback when `Edit` cannot find a unique anchor. **A** can sometimes work but is unreliable and may fail repeatedly if the surrounding methods are similarly structured; the guide names Read + Write as the fallback. **C** bypasses the built-in file tools for a brittle line-number-dependent shell operation. **D** is an unrelated refactor that does not accomplish the requested change.

---

# Fast Revision Card

- Description quality is the root cause of most selection failures. Expand → rename → split, in that order.
- System prompt keywords can override good descriptions.
- `isError` + `errorCategory` + `isRetryable` + customer-friendly text + attempted/partial/alternatives.
- Transient is the only retryable category. Empty result is not an error.
- Subagents recover locally; propagate only what they cannot resolve, with partial results.
- 4–5 tools per agent. Scoped cross-role tool for the high-frequency 85% case.
- `auto` may / `any` must-some / `{"type":"tool","name":X}` must-that. Per request, not per sequence.
- `.mcp.json` = project/shared. `~/.claude.json` = user/personal. `${VAR}` for secrets. All servers live at once.
- Resources = catalogs. Tools = actions.
- Grep = contents. Glob = paths. Edit needs a unique anchor; otherwise Read + Write.
- Distractor smells: classifiers, keyword routers, speculative caching, bigger models, "be more careful" prompts, confidence self-reports.
