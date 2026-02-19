---
name: cleanup
description: >-
  Use when the user asks to "clean up code", "simplify code", "refactor for
  clarity", or "run cleanup". Cleans up and simplifies code changed on the
  working branch.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, AskUserQuestion, Task, Skill
---

# Code Cleanup

Analyze all files changed on the current working branch and propose refinements across 7 categories. Never change what the code does — only how it does it. All original features, outputs, and behaviors must remain intact.

## Workflow

Follow these phases in order. Do NOT skip phases or start editing before the user approves.

1. **Scope Detection** — Identify changed files from branch diff
2. **Exploration Strategy** — Prompt user: sequential or parallel analysis
3. **Exploration** — Analyze files for refactor opportunities
4. **Findings Presentation** — Show numbered findings grouped by file
5. **Selection** — User cherry-picks which refactors to apply
6. **Execution** — Apply approved refactors with test verification
7. **Post-Execution** — Prompt user for superpowers follow-up steps
8. **Report** — Summary of changes, skipped items, and CLAUDE.md suggestions

---

## Phase 1: Scope Detection

Identify which files to analyze by diffing the current branch against its base.

1. Detect the base branch:
   - Run `git config branch.$(git branch --show-current).merge` to check for a tracking branch
   - If not set, check if `main` exists (`git rev-parse --verify main`), fall back to `master`
   - If ambiguous, use AskUserQuestion to ask the user which branch is the base
2. Find the divergence point: `git merge-base HEAD <base-branch>`
3. Get changed files: `git diff --name-only <merge-base>..HEAD`
4. Filter to code files only: `.py`, `.ts`, `.tsx`, `.js`, `.jsx`
5. If no code files changed, report "No code files changed on this branch — nothing to clean up." and stop

You must execute steps 1-3 and examine the output before concluding there are no changed files. Do not skip the git commands.

Store the list of changed files and the merge-base SHA for use in later phases.

## Phase 2: Exploration Strategy Prompt

**This phase is mandatory regardless of how many files changed.** Even for a single file, present the strategy prompt.

Before analyzing files, prompt the user to choose the exploration strategy. Present an AskUserQuestion with your recommendation based on file count:

- If **≤5 files** changed, recommend **Sequential**: "Analyze files one at a time in this session. Lower token cost, simpler."
- If **>5 files** changed, recommend **Parallel**: "Dispatch one analysis agent per file using parallel agents. Faster for large changesets."

Include the file count and list of files in the question context so the user can make an informed choice.

If the user selects **Parallel**, invoke the `superpowers:dispatching-parallel-agents` skill to dispatch one Explore-type subagent per file. Each subagent receives the file path, the diff for that file, and the 7 refactor categories to check.

## Phase 3: Exploration

For each changed file (sequentially or via parallel agents), perform the following analysis:

1. **Read the full file** to understand its structure
2. **Read the diff** (`git diff <merge-base>..HEAD -- <file>`) to understand what changed
3. **For consistency checks (Category 2):** Read 2-3 other files in the same module/directory to understand project patterns
4. **For test gap checks (Category 7):** Check if a corresponding test file exists and what it covers

Produce a structured list of findings. Each finding must include:

- **File path**
- **Category** (1-7, see below)
- **Description**: What the issue is and where (function names, line numbers)

### Refactor Categories

Analyze in this priority order:

#### Category 1: Structural Simplification

The highest-value refactoring. Look for:

- **Long functions (>40 lines)**: Break into smaller functions with descriptive names and single responsibilities
- **Duplicated patterns**: 3+ lines of similar logic in 2+ places → extract a shared helper
- **Complex inline expressions**: Hard-to-read conditions, transformations, or comprehensions → extract into a named function
- **God functions**: Functions doing setup + core logic + cleanup → split into phases
- **Repeated scaffolding**: Repeated try/except, if/else, or context-management patterns → extract into a decorator, context manager, or wrapper
- **Mixed module responsibilities**: A module handling 3+ unrelated concerns → propose splitting

When extracting:

- Place helpers in the same module unless 2+ modules need them
- Name extracted functions by what they return or accomplish, not by origin

#### Category 2: Consistency with Codebase

Code on the branch that doesn't match established patterns elsewhere in the project:

- Different error handling patterns than the rest of the codebase
- String manipulation where the project uses pathlib
- Manual loops where the project uses comprehensions (or vice versa)
- Different naming conventions than surrounding code
- Inconsistent use of project utilities or abstractions

This category requires reading neighboring files to identify the project's patterns.

#### Category 3: Dead Code and Unused Artifacts

- Unused imports
- Unreachable branches (conditions that can never be true)
- Orphaned functions or classes (defined but never called)
- Commented-out code (it belongs in version control, not in the source)
- Variables assigned but never read

#### Category 4: Naming and Readability

- **Ambiguous names**: Variables/functions that don't describe their value or behavior
- **Misleading names**: Names that suggest something different from what the code does
- **Useless comments**: Comments restating what code does, section dividers, decorative blocks, docstrings repeating the signature
- **Poor ordering**: Related statements separated by unrelated code → reorder to group
- **Deep nesting (>3 levels)**: Invert conditions and return early to flatten
- **Redundant branches**: `if x: return True else: return False` → `return x`

Keep only comments that explain a non-obvious _why_, reference external context, or warn about gotchas.

#### Category 5: Type Safety

- Missing type hints on function signatures and return types
- Use of `any` in TypeScript → replace with `unknown` and narrow, or define a proper type
- Raw dicts used for structured data → Pydantic models (Python) or interfaces (TypeScript)
- Missing `from __future__ import annotations` in Python files
- Bare `Exception` catches → specific exception types

#### Category 6: Magic Values and Constants

Extract literal strings and numbers into named constants when the name adds meaning beyond the literal value.

**Do extract:**

- Values used in 2+ places
- Values whose purpose is not obvious from context (e.g., `86400` → `CACHE_TTL_SECONDS`)
- Domain-specific strings (e.g., `"pantheon-shell"` → `JWT_ISSUER`)
- Cohesive sets of related values (all message roles, all event types)

**Do NOT extract:**

- Values used once in an obvious context with no grouping benefit
- Universally understood protocol values (`"GET"`, `"utf-8"`)
- Values in constant definition files or test assertions

**NEVER remove or inline existing constants.** This skill only adds or restructures.

**Where to put constants:**

- **Python**: `constants.py` at the package root, `UPPER_SNAKE_CASE`
- **TypeScript/JavaScript**: `constants.ts` at the source root, `UPPER_SNAKE_CASE` for primitives, `PascalCase` for object maps, named exports only
- Single-module constants: top of that file

#### Category 7: Test Gaps

Identify changed code paths that have no test coverage:

- New functions or methods with no corresponding test
- Modified branches or error paths not exercised by existing tests
- Edge cases visible in the diff that tests don't cover

This category does NOT propose production code changes. Instead, it identifies where tests should be written. During execution (Phase 6), test gap refactors invoke the `superpowers:test-driven-development` skill.

## Phase 4: Findings Presentation

Consolidate all findings into a single numbered report, grouped by file, then by category. Format:

    ## Cleanup Findings

    ### backend/src/agent.py

    1. [Structural] `run()` is 85 lines — extract `_build_history()` (lines 40-62) and `_handle_tool_calls()` (lines 70-95)
    2. [Structural] Event-emission pattern repeated on lines 30, 48, 72 — extract `_emit_event(type, data)`
    3. [Dead code] `_legacy_format()` on line 90 is unused
    4. [Consistency] Uses string concatenation for paths; rest of codebase uses pathlib

    ### backend/src/main.py

    5. [Naming] `x` on line 12 — unclear variable name in request handler
    6. [Test gaps] No test coverage for error path on lines 45-52

Each finding includes:

- The **category** in brackets
- **What** specifically changes and **where** (function names, line ranges)
- **Why** it improves the code (only when not obvious from the category)

Present this report to the user before proceeding to selection.

## Phase 5: Selection

After presenting findings, use AskUserQuestion with `multiSelect: true` to let the user cherry-pick which refactors to apply. Even if there is only a single finding, present it via AskUserQuestion so the user can approve or reject it before execution begins.

Since AskUserQuestion supports a maximum of 4 options per question, batch findings:

- Group by file when possible
- Include a **"Select all in this batch"** convenience option as the first choice in each batch
- Each option label includes the finding number and a short description

Example batch:

> "Which refactors should I apply for `backend/src/agent.py`?"
>
> - **All 4 refactors** — Apply findings #1-4
> - **#1 Extract `_build_history()` and `_handle_tool_calls()`**
> - **#2 Extract `_emit_event()`**
> - **#3 Remove `_legacy_format()`**

Then a second batch for the next file:

> "Which refactors for `backend/src/main.py`?"
>
> - **All 2 refactors** — Apply findings #5-6
> - **#5 Rename `x` in request handler**
> - **#6 Add test coverage for error path**

If a single file has more than 3 findings, split into multiple batches of 3 (plus "Select all" = 4 options).

Collect the full list of approved finding numbers before proceeding to execution.

## Phase 6: Execution

Apply each approved refactor one at a time. After each refactor, verify that tests still pass. Each approved finding must be applied as a separate, independently testable edit. Never combine multiple findings into a single edit, even if they affect the same file or function.

### Execution loop

For each approved finding (in the order they were numbered):

1. **Apply the refactor** using Edit/Write tools
2. **Run the test suite** using the project's test command (detected from CLAUDE.md or standard conventions: `pytest`, `npm test`, `vitest`, etc.)
3. **If tests pass:** Continue to the next refactor
4. **If tests fail:** Immediately undo all changes from this refactor with `git checkout -- <files...>` (listing every file touched by the refactor). Then present an AskUserQuestion:
   - **Skip this refactor** — Move on to the next approved finding
   - **Debug with systematic-debugging** — Invoke the `superpowers:systematic-debugging` skill to investigate root cause and attempt a fix
   - **Stop execution** — Halt all remaining refactors and proceed to post-execution

### Special handling: Category 7 (Test Gaps)

Test gap findings do NOT edit production code. Instead:

1. Invoke the `superpowers:test-driven-development` skill to write the missing test
2. Since the production code already exists and functions correctly, the test should pass against the existing code. If the test fails, investigate the test — not the production code
3. After the test is written and passing, continue the execution loop

### Commit strategy

After completing a logical group of refactors (typically all approved refactors for one file), create an atomic commit:

- **Commit message format:** `<TICKET-ID>: cleanup — <brief description>`
- Extract the ticket ID from the current branch name (per CLAUDE.md conventions)
- **Use `git commit --fixup <commit>`** when the cleanup directly improves code introduced by a specific commit on the branch (e.g., renaming a function added 2 commits ago)
- **Use a new commit** when the cleanup addresses pre-existing code issues or spans multiple unrelated concerns

Use best judgment on whether fixup or new commit is appropriate. When in doubt, prefer a new commit — it's easier to squash later than to unsquash.

## Phase 7: Post-Execution

After execution completes, present an AskUserQuestion regardless of how many refactors succeeded. If no refactors were successfully applied, adjust the message to reflect that no changes were made — the user may still want to run verification or review findings.

> "All approved refactors applied and committed. What next?"
>
> - **Run verification** — Invoke `superpowers:verification-before-completion` to run the full test suite and confirm nothing is broken
> - **Verification + code review** — Run verification, then invoke `superpowers:requesting-code-review` for an independent quality check of the cleanup changes
> - **Full pipeline** — Verification, code review, then invoke `superpowers:finishing-a-development-branch` for branch completion options (merge, PR, keep, discard)
> - **Done** — Skip post-execution steps; work is already committed

Execute whichever option the user selects. If verification fails at any point, report the failure and stop.

## Phase 8: Report

Always produce a summary after execution completes (regardless of post-execution choice):

- **Files modified**: Each file and what was done (e.g., "extracted `_build_history()` from `run()`", "removed 3 dead functions")
- **New functions/modules created**: List with one-line descriptions
- **Constants created**: List any new constants with names and values
- **Tests created**: List any new test functions from Category 7 refactors
- **Findings skipped or failed**: List findings that were rejected, skipped due to test failures, or failed debugging
- **Patterns observed**: Recurring issues seen across files that may apply to untouched files

### Suggest CLAUDE.md Standards

After the summary, suggest specific rules for CLAUDE.md to prevent the issues just fixed from recurring. Format as ready-to-paste bullet points. Only suggest standards supported by problems actually found during this cleanup run.

---

## Guardrails

- Never change what code does — only how it does it
- All original features, outputs, and behaviors must remain intact
- Do not over-simplify in ways that reduce clarity
- Do not combine unrelated concerns just to reduce line count
- Do not remove abstractions that serve a clear purpose
- Prefer clarity over brevity in all cases
- Only analyze files changed on the working branch
- Respect conventions documented in CLAUDE.md
- NEVER remove or inline existing constants
