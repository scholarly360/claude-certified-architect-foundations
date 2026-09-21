# Claude Certified Architect – Foundations (CCAR-F)
# Domain 3: Claude Code Configuration & Workflows — Study Guide

## Domain 3 at a glance

- **Weight:** 20% of scored items → roughly **12 of 60 questions**
- **Primary domain for:** Scenario 2 (Code Generation with Claude Code) and Scenario 5 (Claude Code for CI/CD)
- **Secondary domain for:** Scenario 4 (Developer Productivity)
- 4 of 6 scenarios are drawn at random, so you are very likely to see at least one Domain 3 scenario block.

**Task statements:**
- 3.1 CLAUDE.md hierarchy, scoping, modular organization
- 3.2 Custom slash commands and skills
- 3.3 Path-specific rules
- 3.4 Plan mode vs direct execution
- 3.5 Iterative refinement
- 3.6 CI/CD integration

> **Unlocking framing:** Every question is asking *"Which configuration layer is the right home for this instruction, and is the mechanism deterministic or probabilistic?"*

---

## 3.1 — CLAUDE.md hierarchy, scoping, and modular organization

### Core Concept

CLAUDE.md is **always-loaded context, not enforced configuration**. Claude reads it and tries to follow it, but to block an action regardless of what Claude decides, you need a hook. Exam items constantly test "shape behavior" (CLAUDE.md, rules, skills) vs "guarantee behavior" (hooks, permission settings).

| Scope | Location | Shared with |
|---|---|---|
| Managed / enterprise policy | OS-specific managed path | All users in the org |
| User | `~/.claude/CLAUDE.md` | Just you, all projects |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team, via version control |
| Local | `./CLAUDE.local.md` | Just you, this project (gitignore it) |
| Directory | subdirectory `CLAUDE.md` | Team, loads on demand |

Files in the working directory and above load at launch. Files in subdirectories load on demand when Claude reads files in those directories.

### Canonical Skeleton

```
repo/
├── CLAUDE.md                  # universal, always-on standards (<200 lines)
│   └── @docs/api-standards.md # modular import
├── CLAUDE.local.md            # gitignored personal prefs
├── .claude/
│   ├── rules/
│   │   ├── testing.md         # paths: ["**/*.test.*"]
│   │   ├── api-conventions.md # paths: ["src/api/**/*"]
│   │   └── security.md        # no paths → always loaded
│   ├── commands/review.md     # project slash command
│   └── skills/audit/SKILL.md  # project skill
└── packages/frontend/CLAUDE.md  # directory-level, loads on demand
```

### Exam Traps

- **"Put it in `~/.claude/CLAUDE.md` so the whole team picks it up."** The most-tested misconception. User scope is machine-local and never travels through version control. "A new team member isn't getting the instruction" = scope diagnosis → move it to project scope.
- **"Use `@import` to reduce context/token usage."** False. Imported files are expanded and loaded at launch. Imports help organization, not token cost. The token-reduction mechanism is **path-scoped rules**.
- **"CLAUDE.md will ensure the agent always does X before Y."** No. Words like *guaranteed*, *must never*, *deterministic*, or money/identity/destructive actions → hook or permission rule.
- Confusing CLAUDE.md (context/conventions) with `.claude/commands/` (invocable commands). A `.claude/config.json` with a commands array **does not exist** (official sample distractor).

### Key Facts to Memorize

- `~/.claude/CLAUDE.md` = user, not shared. `./CLAUDE.md` or `./.claude/CLAUDE.md` = project, shared. `CLAUDE.local.md` = personal + gitignored.
- `@path/to/file` import syntax; relative paths resolve against the importing file; max recursion depth of **four hops**.
- `/memory` lists and opens memory files; `/context` shows which ones actually loaded. **Blueprint keys `/memory`** for verifying loaded files and diagnosing inconsistent behavior.
- Target **under 200 lines** per CLAUDE.md; longer files consume context and reduce adherence.
- Project-root CLAUDE.md **survives `/compact`** (re-read from disk and re-injected). Conversation-only instructions do not.

### Exam-Style Questions

**Q1.** Your team's CLAUDE.md instructs Claude to use `pnpm` and run the closest test before reporting completion. It works on your machine. A new engineer clones the repo and Claude consistently uses `npm`. What is the most likely cause?

- A. The new engineer's Claude Code version doesn't support memory files.
- B. The instruction lives in `~/.claude/CLAUDE.md` rather than a project-level CLAUDE.md committed to the repo.
- C. The instruction needs to be converted to a hook to apply to other users.
- D. The new engineer must run `/init` before CLAUDE.md files are read.

**Answer: B.** User-level memory is machine-local and never shared through version control. Moving it to `./CLAUDE.md` or `./.claude/CLAUDE.md` and committing it fixes the issue. A has no basis in the scenario. C confuses guidance with enforcement — the real problem is the file never reached the teammate. D is false; CLAUDE.md loads automatically and `/init` only generates a starter file.

**Q2.** A monorepo has a 900-line root CLAUDE.md covering Terraform, React, and Go conventions. Sessions in any package load all of it, and adherence has degraded. Which change best reduces per-session context while keeping conventions shared with the team?

- A. Split the content into separate files and reference them from CLAUDE.md with `@import`.
- B. Move each section into `.claude/rules/` files with `paths` frontmatter scoping them to matching globs.
- C. Move the content into `~/.claude/CLAUDE.md` on each developer's machine.
- D. Delete the conventions and rely on Claude inferring them from existing code.

**Answer: B.** Path-scoped rules load only when Claude works with matching files, which actually reduces context. A is the classic trap — imports load at launch, so token cost is unchanged. C loses team sharing and doesn't reduce context. D discards the standards.

---

## 3.2 — Custom slash commands and skills

### Core Concept

Commands and skills are **on-demand** context; CLAUDE.md is **always-loaded** context. That trade-off is the decision criterion:

- **CLAUDE.md** → universal standards needed in every session
- **Skill / command** → task-specific procedure or checklist invoked when needed

Scoping mirrors CLAUDE.md:
- `.claude/commands/` and `.claude/skills/` → project-scoped, version-controlled
- `~/.claude/commands/` and `~/.claude/skills/` → personal

Custom commands have been merged into skills: `.claude/commands/deploy.md` and `.claude/skills/deploy/SKILL.md` both create `/deploy`. Skills add a directory for supporting files, frontmatter for invocation control, and automatic loading when relevant.

### Canonical Skeleton

```yaml
---
name: codebase-audit
description: Audit module structure and report dependency risks
argument-hint: "[module-path]"
context: fork              # run in an isolated subagent
agent: Explore             # which subagent type executes it
allowed-tools: Read Grep Glob
disable-model-invocation: true   # only the human can trigger it
---

Audit $ARGUMENTS:
1. Map the module's exports and imports
2. Identify circular dependencies
3. Return a summary with file references
```

### Exam Traps

- **`context: fork` is not a fork of your conversation.** The forked subagent does not see conversation history, so the skill must stand on its own. A guidelines-only skill with no actionable task returns nothing useful when forked.
- **Editing the shared project skill to suit yourself.** Keyed pattern: create a **variant in `~/.claude/skills/` under a different name** so teammates are unaffected.
- **Skill vs CLAUDE.md inversion.** Multi-step procedure in CLAUDE.md burns context every session; a universal standard in a skill only loads when invoked.
- **`allowed-tools` semantics.** Blueprint describes it as *restricting* tool access during skill execution. Current product: it *grants pre-approval* for the invoking turn and does not restrict; `disallowed-tools` restricts. **On the exam, answer per the blueprint.** In production, use `disallowed-tools` or permission deny rules.

### Key Facts to Memorize

- `.claude/commands/` = project, shared. `~/.claude/commands/` = personal.
- Skill layout: `.claude/skills/<name>/SKILL.md`. For personal/project skills, the command name comes from the **directory name**, not frontmatter `name`.
- Frontmatter fields named in the blueprint: `context: fork`, `allowed-tools`, `argument-hint`.
- `agent:` selects the subagent type for a fork: `Explore`, `Plan`, or `general-purpose`.
- `argument-hint` prompts the developer for parameters at autocomplete when they invoke without arguments.

### Exam-Style Questions

**Q3.** Your team's `/analyze-deps` skill produces several thousand tokens of dependency-graph output. After running it, Claude loses track of the feature developers were working on. What is the most effective fix?

- A. Add `argument-hint` so developers scope the analysis to one module.
- B. Add `context: fork` to the skill's frontmatter so it runs in an isolated subagent and returns only its summary.
- C. Move the skill's instructions into CLAUDE.md so they load once at session start.
- D. Instruct developers to run `/compact` immediately after invoking the skill.

**Answer: B.** `context: fork` isolates verbose output in a subagent and returns only the result. A reduces size but doesn't isolate. C loads the procedure into every session. D is reactive, lossy cleanup after the context is already polluted.

**Q4.** A project skill `/commit` enforces team commit conventions. You want a version that also runs your personal pre-commit lint script, without changing behavior for teammates. What should you do?

- A. Add your lint step to `.claude/skills/commit/SKILL.md` behind a conditional.
- B. Create a skill with a different name in `~/.claude/skills/`.
- C. Add the lint step to `~/.claude/CLAUDE.md`.
- D. Override the project skill by creating `~/.claude/skills/commit/SKILL.md`.

**Answer: B.** Personal variants under a distinct name leave the shared skill untouched and avoid ambiguity. A modifies shared config. C puts a procedure into always-loaded context. D creates a name collision — exactly what "different name" guidance avoids.

---

## 3.3 — Path-specific rules for conditional convention loading

### Core Concept

`.claude/rules/*.md` files can carry YAML frontmatter with a `paths` field of glob patterns. These rules load only when Claude reads files matching the patterns. Rules **without** `paths` load unconditionally.

> **Decision criterion:** Glob-based rules beat directory-based CLAUDE.md whenever files sharing a convention are **scattered across directories**.

### Canonical Skeleton

```markdown
---
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
---

# Testing conventions

- Use React Testing Library, never enzyme
- One assertion concept per test
- Mock at the network boundary with MSW, not module mocks
```

### Exam Traps

- **Subdirectory CLAUDE.md for test conventions.** Co-located tests (`Button.test.tsx` beside `Button.tsx`) are everywhere; a directory-bound file can't cover them without duplication.
- **Consolidating under headers in root CLAUDE.md.** Relies on inference. Stems with "automatically" or "deterministic" rule this out.
- **Using skills for automatic path-based conventions.** Skills need invocation or a model decision. (The product has added `paths` to skill frontmatter, but the **blueprint's keyed answer is `.claude/rules/`**.)
- **User-level rules for team conventions.** `~/.claude/rules/` is personal and loads before project rules. Not shared.

### Key Facts to Memorize

- Frontmatter key is `paths` — a YAML list of globs.
- `**/*.test.tsx` = all test files anywhere. `terraform/**/*` = everything under a directory. `src/**/*.{ts,tsx}` = brace expansion.
- Rules with no `paths` load at launch with the same priority as `.claude/CLAUDE.md`.
- `.claude/rules/` is committed → **team-shared** mechanism.

### Exam-Style Question

**Q5.** Your repo has Terraform under `infra/` plus generated Terraform in three service directories, and test files co-located throughout `src/`. You need Terraform conventions applied whenever Claude touches any `.tf` file, and test conventions whenever it touches any test file, without loading either set in unrelated sessions. What is the most maintainable approach?

- A. Two `.claude/rules/` files with `paths: ["**/*.tf"]` and `paths: ["**/*.test.*"]` respectively.
- B. A CLAUDE.md in `infra/`, one in each service directory, plus one per `src/` subdirectory containing tests.
- C. Two sections in root CLAUDE.md, with a preamble telling Claude which section applies to which file type.
- D. Two skills that developers invoke before editing Terraform or test files.

**Answer: A.** Glob rules apply by file type regardless of location and load only when matching files are read. B duplicates content and misses directories you forget. C loads everything every session and depends on inference. D depends on developers remembering to invoke.

---

## 3.4 — Plan mode vs direct execution

### Core Concept

Plan mode is a **read-only research phase**: Claude reads files, runs exploratory commands, and writes a plan, but does not edit source. Edits stay blocked until you approve the plan.

Selection criterion = **complexity that is known in advance**, not complexity that might emerge.

| Signal in the stem | Answer |
|---|---|
| Multi-file, dozens of files, migration, restructuring | Plan mode |
| "Multiple valid approaches" / "architectural decision" | Plan mode |
| Unfamiliar or legacy codebase, service boundaries | Plan mode |
| Single-file fix with a clear stack trace | Direct execution |
| Add one validation conditional to one function | Direct execution |
| Verbose discovery threatening the context window | Explore subagent |

### Canonical Skeleton

**Plan (investigate) → approve → direct execution (implement).** The blueprint explicitly rewards this combination (plan a library migration, then execute the planned approach).

### Exam Traps

- **"Start direct, switch to plan mode if complexity appears."** Wrong when the stem already states the complexity.
- **"Use a model with a larger context window instead."** Doesn't fix attention dilution or the need for a design decision.
- **"Let the implementation reveal the natural boundaries."** Risks costly rework when dependencies surface late.
- **Plan mode ≠ Plan subagent.** Plan mode is a permission mode; `Explore` and `Plan` are subagent types.

### Key Facts to Memorize

- Enter plan mode: **Shift+Tab**, `/plan` prefix on one prompt, or `claude --permission-mode plan`.
- **Explore subagent:** fast, read-only (Write/Edit denied), for file discovery and code search; keeps exploration results out of the main context.
- Explore and Plan subagents **skip CLAUDE.md and git status** to keep research fast and cheap; other built-in and custom subagents load both.
- Approving a plan exits plan mode and switches to the permission mode named by the approval option you chose.

### Exam-Style Questions

**Q6.** A production stack trace points to a null dereference on one line in `dateUtils.ts`. The fix is one conditional. Which approach is appropriate?

- A. Plan mode, to explore how dates are handled everywhere before changing anything.
- B. Direct execution.
- C. Plan mode, then a second plan-mode pass to validate the first plan.
- D. Spawn an Explore subagent to map the module before editing.

**Answer: B.** Clear scope, identified cause, single file. Plan mode's value (exploration, design choice) isn't needed. C compounds cost. D is for verbose discovery, which isn't present.

**Q7.** You are migrating from an internal HTTP client to a new library. Grep shows 45+ call sites, two viable adapter strategies with different retry semantics, and several call sites in a legacy module nobody understands. What is the best workflow?

- A. Direct execution with a detailed upfront instruction specifying exactly how each call site should be rewritten.
- B. Plan mode to explore and choose an approach, using the Explore subagent for legacy-module discovery, then direct execution to implement the approved plan.
- C. Split the work into 45 separate single-file direct-execution sessions.
- D. Direct execution, switching to plan mode if retry semantics turn out to matter.

**Answer: B.** Every plan-mode signal is present: many files, multiple valid approaches, an architectural choice. Explore isolates verbose legacy discovery. A presumes you know the design without exploring. C loses the cross-file view the retry decision needs. D ignores stated complexity.

---

## 3.5 — Iterative refinement techniques

### Core Concept

| Symptom in the stem | Technique |
|---|---|
| Prose description interpreted inconsistently across runs | 2–3 concrete input/output examples |
| Behavior needs to converge on a spec | Test-driven iteration: write tests first, share failures |
| Unfamiliar domain; you don't know what you don't know | Interview pattern: have Claude ask questions first |
| Several problems found at once | Batch if they interact; sequence if independent |

### Canonical Skeleton

```
1. Write test suite (expected behavior, edge cases, performance)
2. Ask Claude to implement
3. Run tests → paste failures verbatim
4. Repeat until green
5. For a persistent edge case: give input + expected output, not more prose
```

### Exam Traps

- **Restating the prose more emphatically or in more detail.** When prose already produced inconsistent results, more prose is not the fix — examples are.
- **Fixing interacting issues one at a time.** If fixing A could re-break B, send both in one detailed message. Sequential is for independent problems.
- **"Be more careful."** Vague self-instruction = same failure pattern as "be conservative" in Domain 4.
- **Skipping the interview pattern** in unfamiliar domains (cache invalidation, failure modes) and jumping to implementation.

### Key Facts to Memorize

- Keyed count: **2–3 input/output examples** (and 2–4 few-shot examples in Domain 4). Options with 8–10 examples are usually "more is better" distractors.
- Test-driven iteration covers expected behavior, edge cases, **and** performance requirements.
- For a null-handling edge case in a migration script: a specific test case with example input and expected output.

### Exam-Style Questions

**Q8.** You describe a log-normalization transform in three paragraphs of prose. Across five runs Claude produces five different output shapes, each defensible under your description. What is the most effective next step?

- A. Rewrite the description with more precise language and more detail.
- B. Provide 2–3 concrete input/output example pairs showing the exact transformation.
- C. Increase the model's thinking budget so it reasons more carefully.
- D. Split the transform into five smaller prose-described steps.

**Answer: B.** The ambiguity is in the description format, not reasoning effort. A repeats what failed. C targets reasoning depth when the target is underspecified. D multiplies the ambiguity.

**Q9.** A review surfaces four problems in a generated caching layer: wrong TTL, invalidation misses a key prefix, lock not released on error, and a variable name violating the style guide. The first three interact. How should you iterate?

- A. Send all four issues in one message.
- B. Send the three interacting issues in one detailed message, then the naming issue separately.
- C. Send each of the four issues sequentially, verifying after each.
- D. Send only the naming issue and regenerate the caching layer from scratch.

**Answer: B.** Interacting problems are fixed together so fixes are designed against each other; the independent style issue is handled separately. A is defensible but adds unrelated noise. C risks fix-then-re-break across coupled issues. D discards working code.

---

## 3.6 — CI/CD integration

### Core Concept

Non-interactive mode + structured output + project context + independent review:

1. `-p` / `--print` → process never waits for input
2. `--output-format json` (+ `--json-schema` when parsing findings) → machine-readable output
3. CLAUDE.md → review criteria, testing standards, fixture conventions
4. **Independent instance** reviews code that a different session generated

With `--output-format json`, text is in `.result`. With `--json-schema`, schema-conforming output is in `.structured_output`.

### Canonical Skeleton

```bash
# PR review job
gh pr diff "$PR" | claude -p \
  --output-format json \
  --json-schema "$(cat .claude/schemas/findings.json)" \
  --append-system-prompt "You are reviewing for security and correctness." \
  | jq -c '.structured_output.findings[]' \
  | post-inline-comments
```

**Re-runs after new commits:** include prior findings in context and instruct Claude to report only new or still-unaddressed issues.

### Exam Traps

- **Invented flags/env vars.** `CLAUDE_HEADLESS=true` and `--batch` **do not exist**. `< /dev/null` is a Unix workaround, not the answer. **Always `-p`.**
- **Self-review.** The generating session retains its reasoning and is less likely to question itself. An independent instance catches more.
- **Duplicate PR comments on re-review.** Fix = pass prior findings + "report only new/unresolved," not downstream dedup or suppressing re-review.
- **Low-value or duplicate generated tests.** Fix = provide existing test files in context **and** document testing standards, valuable-test criteria, and fixtures in CLAUDE.md.
- **Batch API for blocking checks.** Message Batches API = 50% savings, up to 24h processing, **no latency SLA** → fine for overnight jobs, wrong for blocking pre-merge checks.

### Key Facts to Memorize

- `-p` = `--print`. Exit code 0 on success, non-zero on failure.
- `--output-format`: `text` (default), `json`, `stream-json`.
- `.result` = text output. `.structured_output` = schema-conforming output. Parse with `jq`.
- Capture `session_id` from JSON output → pass to `--resume` for a specific conversation; `--continue` for the most recent.
- `--allowedTools "Read,Grep,Glob"` for locked-down CI. `--append-system-prompt` adds a reviewer persona without replacing defaults.
- `--bare` skips hooks, skills, commands, subagents, plugins, MCP servers, auto memory, and CLAUDE.md → hermetic, reproducible runs. **Tension:** if a stem asks about *providing project context to CI*, the answer is **CLAUDE.md**; `--bare` is for context-free runs.

### Exam-Style Questions

**Q10.** Your GitHub Actions job runs `claude "Review the diff for security issues"` and times out after 6 hours with no output. Logs show the process waiting on input. What is the correct fix?

- A. `claude -p "Review the diff for security issues"`
- B. Export `CLAUDE_HEADLESS=true` before the command.
- C. Append `< /dev/null` to the command.
- D. Add `--batch` to the command.

**Answer: A.** `-p` is the documented non-interactive mode. B and D don't exist. C may stop the hang but doesn't enter print mode or give a usable stdout contract.

**Q11.** Your CI review posts one large free-form prose comment per PR. You want findings posted as inline comments anchored to file and line. Which change enables this most directly?

- A. Ask in the prompt for `file:line — issue` formatting and parse with a regex.
- B. Run with `--output-format json` and `--json-schema` defining an array of findings (file, line, severity, suggestion), then read `structured_output`.
- C. Run with `--output-format stream-json` and reassemble the text deltas.
- D. Split the review into one `claude -p` invocation per changed file.

**Answer: B.** Schema-conforming, machine-parseable findings are what an automated commenter needs. A relies on formatting compliance and brittle parsing. C is streaming transport, not a schema guarantee. D is a valid multi-pass strategy for attention dilution but doesn't give line anchors or a parseable payload.

**Q12 (multiple response — select TWO).** A generation job writes new code, and a later step in the same pipeline reviews it. Reviews rarely flag anything, and when developers push fixes, the re-run repeats every previous comment. Which two changes address these problems?

- A. Run the review in an independent Claude Code instance rather than continuing the generating session.
- B. Increase `--max-turns` on the review invocation.
- C. Include the prior review's findings in the re-run's context and instruct Claude to report only new or still-unaddressed issues.
- D. Switch the review step to the Message Batches API for 50% savings.
- E. Instruct the generating session to critically re-examine its own output before finishing.

**Answers: A and C.** An independent instance lacks the generator's reasoning context and catches more (which also rules out E). Passing prior findings with a "new/unresolved only" instruction fixes duplicates. B addresses turn budget, not quality. D is the wrong latency profile and doesn't touch either symptom.

---

## Domain 3 Rapid-Fire Memorization Sheet

| Need | Mechanism |
|---|---|
| Team-wide always-on standards | Project `CLAUDE.md` (committed) |
| Personal preferences, all projects | `~/.claude/CLAUDE.md` |
| Personal, this project, uncommitted | `CLAUDE.local.md` (gitignored) |
| Conventions for a file type scattered across dirs | `.claude/rules/*.md` with `paths` globs |
| Conventions for one directory tree | Directory-level `CLAUDE.md` |
| Invocable team workflow | `.claude/commands/` or `.claude/skills/` (committed) |
| Invocable personal workflow | `~/.claude/commands/` or `~/.claude/skills/` |
| Keep verbose skill output out of main context | `context: fork` (+ `agent: Explore`) |
| Keep verbose discovery out of main context | Explore subagent |
| Prompt for missing skill parameters | `argument-hint` |
| Constrain a skill's tools (blueprint answer) | `allowed-tools` |
| Guarantee a step happens | Hook / permission rule — never CLAUDE.md |
| Design before large-scale change | Plan mode |
| Well-scoped single-file change | Direct execution |
| Non-interactive CI run | `-p` / `--print` |
| Machine-parseable CI findings | `--output-format json` + `--json-schema` → `structured_output` |
| Continue a specific CI conversation | Capture `session_id` → `--resume` |
| Which memory files are loaded | `/memory` (blueprint) / `/context` (current product) |
| Shrink a bloated session | `/compact` |

---

## Where the Blueprint and Current Product Diverge

Answer per the blueprint on exam day, but know the real behavior:

1. **`allowed-tools`**
   - Blueprint: restricts tool access during skill execution.
   - Product today: grants pre-approval for the invoking turn; does **not** restrict. `disallowed-tools` restricts.
2. **Verifying loaded memory**
   - Blueprint: `/memory`.
   - Product today: `/memory` lists/opens memory files; `/context` shows which files actually loaded this session.

