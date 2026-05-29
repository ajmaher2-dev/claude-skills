---
name: feature-implement
description: Implement one or more features end-to-end — execute tasks, run tests, review code quality, update docs, mark completed. Stops only when human input is required.
user_invocable: true
---

# /feature-implement

Implement one or more features fully autonomously. Reads the feature plan, executes every task, runs quality checks, updates docs, and marks as completed. Processes features sequentially. Stops only when a decision requires human input.

This skill is **repo-agnostic.** It reads `.claude/repo-conventions.yaml` at Step 0 (schema: `~/.claude/repo-conventions-schema.md`) and parameterises lint/test/build commands and pattern-specific guidance against the active repo's conventions. Examples below use EmailPOC values for concreteness; substitute your repo's values.

## Usage

```
/feature-implement <feature-path>
/feature-implement <path1> <path2> <path3> ...
```

---

## Step 0 — Read repo conventions (run once before any feature)

Read `.claude/repo-conventions.yaml` at the repo root. Cache values for:
- `package`, `backend.test_dir`, `backend.api_main`, `backend.events_module`, `backend.migrations.{tool,dir}`, `backend.patterns.*`
- `frontend.dirs`, `frontend.test_framework`, `frontend.visual_regression`, `frontend.visual_safety_net`, `frontend.build_command`
- `docs.*`
- `ci.*` (lint, format, pre_push, smoke_test, architecture_guards if declared)
- `testing.runner`, `testing.pre_existing_failures`, `testing.env_gaps` (if declared)

If `.claude/repo-conventions.yaml` is missing: auto-detect what you can; ask the user for the rest. Suggest authoring the file at the end of the run.

Throughout this skill, `<package>`, `<test_dir>`, `<frontend.dirs>`, `<ci.lint>`, etc. refer to values from this file. Pattern-specific commands (architecture guards, UI commands like `make ui-a11y`) **fire only if the relevant conventions key is declared**; otherwise note the skip in the Stage 5 pre-flight summary.

---

## Batch Mode

When multiple paths are provided, process each feature sequentially. Each feature runs through the full pipeline independently.

### Step 1 — Validate input

For each path:
- Verify the folder exists
- Verify it contains `proposal.md`, `design.md`, and `tasks.md`
- Verify `proposal.md` frontmatter `status` is NOT `done` (skip completed features)

Print a manifest:
```
## Feature Implementation Queue

Features to implement (N):
1. <path> — <name from frontmatter>
2. <path> — <name from frontmatter>

Beginning feature 1 of N...
```

### Step 2 — Process each feature

Run the Single Feature Pipeline (below) for each feature. After each feature completes, announce progress:

```
✓ Feature 1 complete (<name>) — starting feature 2 of N...
```

If a feature exits with `USER_BLOCKED`, stop the entire batch and report which features completed, which is blocked, and which are pending.

### Step 3 — Print batch summary

```markdown
## Implementation Complete

| # | Feature | Status | Tasks | Quality |
|---|---------|--------|-------|---------|
| 1 | <name> | DONE | 14/14 | CLEAN |
| 2 | <name> | USER_BLOCKED | 8/12 | — |
| 3 | <name> | PENDING | 0/10 | — |

### Completed
- <feature 1>: renamed to _completed_<feature-name>

### Blocked
- <feature 2>: <reason — what decision is needed>

### Pending
- <feature 3>: not started (blocked feature ahead in queue)
```

---

## Single Feature Pipeline

Six stages, executed sequentially. Each stage must complete before the next begins.

```
Stage 1: Plan Review     — read and understand the plan
Stage 2: Execute Tasks   — implement each task from tasks.md
Stage 3: Quality Gate    — simplify + deslop + code-review checks
Stage 4: Post-Impl       — update docs, frontmatter, README
Stage 5: CI Pre-flight   — run all locally-runnable CI checks; fix failures
Stage 6: Finalize        — mark completed, commit, push
```

---

### Stage 1: Plan Review

Read all three plan files:
- `proposal.md` — understand the problem and solution
- `design.md` — understand files to create/modify, architecture decisions, testing strategy
- `tasks.md` — the execution checklist

Read `CLAUDE.md` for project conventions.

Identify:
- **Pre-implementation tasks** — verification steps to run before coding
- **Implementation tasks** — code changes (backend, frontend, schema, etc.)
- **Testing tasks** — test files to create
- **Post-implementation tasks** — doc updates, status changes

Check for blockers:
- Do `depends-on` features exist and have `status: done`?
- Do referenced files in design.md exist?
- Are there RAID Issues that need resolution first?

If a blocker is found that requires human judgment, EXIT with `USER_BLOCKED` and explain what's needed.

---

### Stage 2: Execute Tasks

Process tasks in order from tasks.md. For each task:

#### 2a. Read the task description

Parse what needs to happen: which file, what change, what the expected outcome is.

#### 2b. Execute the task

- For code changes: read the target file, understand the context, make the change
- For new files: check design.md for the specification, create the file
- For schema changes: update the schema, check for downstream consumers
- For test files: read the code being tested, write tests covering happy path, error paths, and edge cases per the testing strategy in design.md

#### 2c. Verify the task

After each task:
- Run `<ci.lint>` on any changed source files (e.g., `ruff check` for Python, `npx eslint` for JS/TS)
- If a lint error was introduced, fix it immediately
- Mark the task as done mentally (do not edit tasks.md yet — batch update at the end)

#### 2d. Handle failures

If a task cannot be completed:
- **Missing dependency** (file doesn't exist, service not available) — check if an earlier task should have created it. If so, go back and fix. If it's external, EXIT with `USER_BLOCKED`.
- **Ambiguous requirement** (task description is unclear, multiple valid approaches) — pick the approach that matches existing codebase patterns. Document the choice in a code comment only if non-obvious.
- **Design conflict** (task contradicts design.md or existing code) — EXIT with `USER_BLOCKED` and explain the conflict.

#### 2e. Opportunistic improvement

While reading files to execute tasks, you will often encounter code that is clearly broken or improvable but outside the task scope. Do not ignore it. Apply this decision tree:

- **Fix it now** if ALL of the following are true:
  - Self-contained change (single file or a small pair of files)
  - Low risk (no schema changes, no API contract changes, no cross-cutting refactor)
  - You are confident in the correct fix (no ambiguity about the right approach)
  - Examples: failing test, dead import, obvious typo, restated comment, simple extract-to-function, broken fixture

- **Spawn a tech-debt feature and continue** if ANY of the following are true:
  - The fix spans multiple files or services in a non-trivial way
  - You see the problem clearly but the right approach is uncertain
  - Fixing it would change API contracts, DB schema, or require a cross-cutting refactor
  - Examples: architectural smell in a core module, test suite design issue affecting many files, ambiguous data model

  To spawn: create a minimal feature folder under the current epic's `features/` directory (e.g. `features/tech-debt-<short-name>/`) with a `proposal.md` describing the problem, affected files, and why you flagged it. Do NOT block or pause the current implementation — note the new feature in your Stage 6 report and continue.

**There is no third option.** Every issue you notice during implementation must result in one of the two actions above. Silent ignoring is not permitted — if you noticed it, you must act on it and report it. Both outcomes must appear in the Stage 6 report. If no issues were encountered, state that explicitly.

The same rule covers **follow-up feature work** (not just code smells) that surfaces mid-implementation: it becomes a *new* feature in the same epic — scaffold a folder as above — never a new task bolted onto the in-flight feature's `tasks.md`. Bolting scope on mid-flight is exactly what stops a feature from being one independent deliverable.

#### 2f. UI implementation discipline

Apply this only when the current task touches a frontend file — under any path in `<frontend.dirs>` (or any `*.tsx` / `*.css`). Skip entirely if `frontend` not declared in conventions.

After making the edit, before marking the task done:

1. **Open the affected route** via the MCP browser tool (`mcp__Claude_in_Chrome__navigate` + `mcp__Claude_in_Chrome__computer` for screenshots, or `mcp__Claude_Preview__preview_screenshot` if a preview server is running). If neither MCP tool is available in the current session, take the screenshot manually via the running `make dev` server and paste it into the conversation.

2. **Iterate the compare-and-fix loop until the live route matches the design intent.** A single comparison pass is **not** sufficient — UI work converges, it doesn't ship first-try.

   The mockup lives at `docs/design/mockups/<feature>/intent.png` (named in design.md's UI Verification section). If the feature itself produces the mockup (a Phase A → user sign-off → Phase B feature), use the captured PNGs from Phase A; if no mockup exists yet, fall back to the ASCII hierarchy diagrams and locked-design notes in design.md.

   Each iteration:
   - Render the current state via MCP screenshot or DOM inspection (`preview_eval`, `preview_inspect`, `preview_snapshot`).
   - Enumerate every observable difference vs. the intent: hierarchy order, spacing, density, font weight/size, colour token, badge placement, truncation behaviour, conditional rendering of optional fields. Be specific — "looks off" is not a finding.
   - Make targeted fixes to the changed file.
   - Re-render. Re-compare.

   The loop terminates only when **either**:
   - No observable differences remain (the live route matches the intent at the locked viewport(s)).
   - All remaining differences are deliberate and documented in design.md or as a Decision Log entry — a difference you can articulate and justify is fine; one you can't is a regression.

   Cap: if 5 iterations don't converge, stop and EXIT with `USER_BLOCKED` describing what won't align and why. Do not push code that visibly diverges from the intent without an explicit decision.

3. **Walk the eight-state checklist** mentally per [docs/design/eight-state-checklist.md](../../../docs/design/eight-state-checklist.md). For each state: is it reachable in the running app and verified? This is a **manual self-review** — do NOT commit a Playwright snapshot per state. The committed CI baseline is default-only per view (see [docs/design/feature-design-loop.md §5](../../../docs/design/feature-design-loop.md)). Additional states get a snapshot only when a concrete concern surfaces.

4. **Token discipline check:** confirm no hex codes, no raw `px` values outside the token system, no inline `style={...}` with magic numbers in the changed file.

If MCP browser tools are unavailable and the user can't screenshot manually for any reason, note this in the Stage 6 report under "Environment gaps" and continue — but flag the design-intent comparison as not performed for this task. Do NOT silently skip the comparison.

---

### Stage 3: Quality Gate

After all implementation and test tasks are done, run three quality checks. Fix issues from each before proceeding to the next.

#### 3a. Simplify

Review all changed files (from git diff) for:
- Code reuse — search for existing utilities that could replace new code
- Code quality — redundant state, copy-paste, leaky abstractions, unnecessary comments
- Efficiency — N+1 queries, missed concurrency, unnecessary work
- **Design token compliance** (only on changed `.tsx` and `.css` files): no hex codes, no `px` values outside the token system, no inline `style={...}` with magic numbers. Per [docs/design/principles.md](../../../docs/design/principles.md).

Fix issues directly. Skip false positives.

#### 3b. Deslop

Scan all changed files for AI-generated slop patterns per CLAUDE.md Anti-Slop Rules:
- Restated comments, hedging language, buzzword comments
- Broad `except Exception` in business logic
- Defensive None checks on typed params
- Placeholder implementations, vague TODOs
- Debug artifacts

Fix issues directly.

#### 3c. Code Review

Run the code review checks from the `/code-review` skill:
- Project pattern compliance (repository pattern, service layer, settings, adapters)
- Anti-slop verification
- Incremental cleanup of pre-existing issues in files touched OR read during implementation (per Incremental Cleanup Policy and Stage 2e). Apply the 2e decision tree: fix small/confident issues directly; spawn a tech-debt feature for large/uncertain ones.
- Test coverage verification (all test layers present, error paths covered)

Fix issues directly. If an incremental cleanup is low-risk and self-contained, apply it.

#### 3d. Run linter

Run `<ci.lint>` on all changed source files. Fix any issues.

#### 3e. Run unit tests for changed modules

Run unit tests scoped to the modules changed in this feature, using the repo's test runner from `testing.runner`:

```bash
# Example for pytest. Substitute the runner and test path from conventions.
# If the project requires test-environment env vars (DB URLs, API keys, etc.),
# they should be declared in conventions under `testing.env_vars` and exported here;
# otherwise check the project's existing CI config (.github/workflows/) for the canonical set.

<testing.runner> -m unit <test_dir>/ -x --tb=short -q
```

If tests fail due to a known environment gap (declared in `testing.env_gaps`, or e.g. `libmagic` not installed on Windows for some Python projects), note the limitation and continue. Failures caused by code logic must be fixed before continuing.

#### 3f. Test quality check

Review all test files created or modified in this feature for quality issues:

- **Value:** Every test has meaningful assertions that verify observable behavior. Remove or rewrite tests with no assertions, trivial assertions (`assert result is not None` when the contract is richer), or tests that only verify mock call counts without checking outcomes.
- **Correctness:** Assertions match actual business rules. Fix tests where expected values look arbitrary or were copied from implementation output without validation against a spec or requirement.
- **Level placement:** Each test runs at the right level. Move tests that use full DB setup to verify a pure function into unit tests. Convert tests that heavily mock internal components into integration tests if a real integration test would be more valuable and less brittle.
- **Efficiency:** Fix disproportionate setup — full fixture graphs for a single assertion, repeated expensive setup that could use `@pytest.mark.parametrize` or shared fixtures, unnecessary service instantiation for testing a static/pure method.
- **Deduplication:** Merge or remove tests that assert the same behavior as another test (same or different file). Consolidate parameterizable cases written as separate test functions into `@pytest.mark.parametrize`.

Fix directly. If a change would require significant restructuring beyond this feature's scope, spawn a tech-debt feature (per Stage 2e) and continue.

#### 3g. UI quality checks

Run only if any file under `<frontend.dirs>` changed (or any `*.tsx` / `*.css`). Skip entirely if `frontend` not declared in conventions.

The project's UI checks come in three flavours; check which exist in this repo:
1. **a11y smoke** — declared in conventions as `frontend.ui_a11y_command` (e.g., `make ui-a11y`). Skip if not declared.
2. **Visual harness** — declared as `frontend.ui_visual_command` (e.g., `make ui-visual`). The local run is for **inspection** vs. the mockup; actual regression diff happens in CI via `<frontend.visual_regression>` (e.g., Argos, Percy, Chromatic). Skip if not declared.
3. **Lint** — `<ci.lint>` on touched frontend files (e.g., `npx eslint` with `jsx-a11y` for React).

For new visual specs: the first run on the visual-regression service marks all baselines as "no prior" — accept them carefully in the service's web UI. The baseline is the contract the regression gate enforces forever after; accepting a wrong baseline locks in the wrong design.

For a11y violations: fix directly. Do not silence without a TODO referencing a remediation issue.

---

### Stage 4: Post-Implementation

#### 4a. Update tasks.md

Mark all completed tasks with `[x]`. Update the status header:
```
**Status:** DONE
**Last updated:** YYYY-MM-DD
```

#### 4b. Update proposal.md frontmatter

```yaml
status: done
start: YYYY-MM-DD  # today or when implementation started
end: YYYY-MM-DD    # today
```

#### 4c. Update epic README

Find the epic's README.md (parent of `features/` directory). Update the feature's status in the status table from `not started` to `done`.

#### 4d. Update functional spec (if applicable)

If the feature changed user-facing behavior AND `<docs.functional_spec>` is declared AND design.md mentions updating it, make the update. Skip otherwise.

---

### Stage 5: CI Pre-flight

Run all CI checks that can execute locally to catch failures before pushing. The goal is a green CI run on the first push — fix every failure found here rather than relying on GitHub Actions to surface them.

Determine which check groups apply by running `git diff --name-only HEAD` on the staged changes:
- **Backend checks** — any file under `<package>/`, plus shared backend config (`pyproject.toml`, etc.)
- **Frontend checks** — any file under `<frontend.dirs>`, plus shared frontend config (`package.json`, lint configs)

#### 5a. Architecture guards (backend)

If the conventions file declares `ci.architecture_guards: <command-or-script>`, run it. This is typically a shell script that mirrors the project's CI architecture-guard grep checks (e.g., "no direct `.query()` outside repositories", "no `bus.publish` outside relay") with project-specific allowlists.

If `ci.architecture_guards` is not declared, **skip this step** — note in the Stage 5 pre-flight summary that no architecture guards are configured for this repo. Do NOT improvise project-specific grep-based guards from the skill text; the allowlists are repo-specific and improvisation will produce false positives or miss real violations.

If the guard script reports failures, fix the violations before continuing.

#### 5b. Full unit test suite (backend)

Run the full unit test suite with the repo's runner from `testing.runner`:

```bash
# Example for pytest. Substitute the runner, test path, coverage target from conventions.
<testing.runner> -n auto -m unit <test_dir>/ --tb=short -q --cov=<package> --cov-fail-under=<repo's coverage threshold>
```

On failure: read the full traceback, fix the root cause in production or test code, then re-run. Do not skip or xfail tests to make this pass — fix the underlying issue.

Exception: if a test fails only due to a known environment gap declared in `testing.env_gaps` (or `testing.pre_existing_failures`), skip that specific test file and note it in the Stage 6 commit message.

If tests fail in modules **not** directly changed by this feature, investigate — do not dismiss. Every failure must result in one of:
- Simple pre-existing bug, confident fix → fix it, include in Stage 6 report under "Opportunistic fixes".
- Larger issue with uncertain approach → spawn a tech-debt feature (per Stage 2e), include in Stage 6 report under "Tech-debt spawned".
- Known environment gap (libmagic, missing service) → note explicitly in report under "Environment gaps" and continue.

Do not xfail or skip tests to make this pass. No silent dismissals.

#### 5c. Type-check baseline (backend, if applicable)

If the conventions file declares `ci.typecheck` (e.g., `bash scripts/check_mypy_baseline.sh` for Python+mypy projects, `npx tsc --noEmit` for TypeScript), run it.

If not declared or the tool isn't installed, skip and note in the commit message. If it runs and finds new violations introduced by this feature, fix them.

#### 5d. Pre-commit hooks (if installed)

```bash
pre-commit run --files $(git diff --name-only HEAD | tr '\n' ' ')
```

If pre-commit is not installed, skip. If it runs, fix any failures — pre-commit maps directly to the `pre-commit` CI job.

#### 5e. Lint + format (frontend)

Only run if files under `<frontend.dirs>` changed:

```bash
# Lint each frontend dir
<ci.lint> <frontend.dirs[*]>/src/

# Format check (e.g., prettier for JS/TS, biome, etc. — declared in ci.format_check if applicable)
<ci.format_check> "<frontend.dirs[*]>/src/**/*.{ts,tsx,css}"
```

For format failures, run the auto-fixer (e.g., `npx prettier --write`) on the affected files, then re-verify.

#### 5f. Frontend unit tests (frontend)

Only run if files under `<frontend.dirs>` changed. For each frontend dir that has its own test suite:

```bash
cd <frontend.dirs[N]> && npm run test
```

On failure: fix the test or the implementation. Do not skip.

#### 5g. OpenAPI / generated-types drift (frontend, if applicable)

Only run if the conventions file declares `frontend.openapi_drift_check` AND backend routes or schemas changed.

```bash
# Example: cd <frontend.dirs[main]> && npm run generate:types:ci && npx prettier --write <generated-types-file>
# Then: git diff --exit-code <generated-types-file>
```

If drift is detected, the generated file needs to be committed as part of this feature. Run the generate command, check the diff looks correct, and include the file in the commit.

Skip entirely if the project doesn't generate API types from the backend (no OpenAPI schema sync, hand-written types, etc.).

#### 5h. Pre-flight summary

Before proceeding to Stage 6, print:

```
## CI Pre-flight

| Check | Result |
|-------|--------|
| Architecture guards | PASS / FAIL (fixed) / SKIPPED (not declared in conventions) |
| Unit tests | PASS (N tests) / SKIPPED (env gap from `testing.env_gaps`) |
| Type-check baseline | PASS / SKIPPED (not declared / not installed) |
| Pre-commit | PASS / SKIPPED (not installed) |
| Frontend lint (incl. a11y plugin) | PASS / N/A (no frontend changes) |
| Frontend format check | PASS / N/A (no frontend changes) |
| Frontend tests | PASS / N/A (no frontend changes) |
| a11y smoke (`<frontend.ui_a11y_command>` if declared) | PASS / N/A |
| Local visual screenshots (`<frontend.ui_visual_command>` if declared) | RAN (N screenshots) / N/A |
| Generated-types drift | PASS / N/A (no route/schema changes or not configured) |

Checks that require GitHub Actions (will run on PR):
- Integration tests (need Postgres + Redis)
- Docker build + Trivy CVE scan
- Gitleaks secret scan
- Dependency audit (pip-audit, npm audit)
- Prometheus rules validation
- frontend-visual (Argos diff, PR→test ONLY — not PR→dev for fast-fail, not PR→main as duplicative)
```

If any check that can run locally failed and could not be fixed, EXIT with `USER_BLOCKED` — do not push broken code.

#### 5i. Visual regression note

If the project uses a visual-regression service (`<frontend.visual_regression>` declared, e.g., Argos / Percy / Chromatic), it typically runs as a CI job on PR. Check the project's CI config for which gate it fires on (PR→main, PR→dev, etc.).

For new granular specs: the service marks the first run as "no baseline yet" — accept the baselines in the service's web UI on the PR before merging.

For existing baselines: the service flags any visual diff and posts a PR comment with a link. Review in the service's UI and either accept (genuine intentional change) or reject (unintentional regression — fix and push again).

---

### Stage 6: Finalize

#### 6a. Mark as completed

Rename the feature folder in-place by prepending `_completed_` to the folder name:

```bash
mv <backlog_root>/<epic>/features/<feature-name> <backlog_root>/<epic>/features/_completed_<feature-name>
```

This keeps completed features visible in their original epic while clearly marking them as done.

#### 6b. Commit

Stage all changes and commit:
```
feat: implement <feature-name>

<2-3 line summary of what was implemented>

Tasks: N/N complete
Quality: simplify + deslop + code-review + CI pre-flight passed
```

#### 6c. Push

Push to the current branch.

#### 6d. Report

```markdown
## Feature Complete: <name>

**Tasks:** N/N done
**Files changed:** N
**Tests added:** N
**Quality checks:** CLEAN / N issues fixed
**CI pre-flight:** architecture guards PASS, N unit tests PASS, mypy PASS/SKIPPED, ESLint/Prettier PASS/N/A
**UI verification:** mockup at `<path>` / N/A, granular spec at `<path>` / N/A, safety-net entry removed: yes/no/N/A, MCP comparison: matched / iterated N times / not performed (env gap) / N/A
**Opportunistic fixes:** N pre-existing issues fixed outside task scope (or: none encountered)
**Tech-debt spawned:** `<feature-name>` — `<one-line reason>` (or: none)
**Environment gaps:** `<description>` (or: none)

### Changes
- <file>: <what changed>
- <file>: <what changed>

### Tests
- <test file>: N tests (happy path, error paths, edge cases)
```

---

## Exit Conditions

### DONE
All tasks complete, quality gate passed, CI pre-flight passed, docs updated, folder renamed to `_completed_<name>`.

### USER_BLOCKED
A decision requires human input. Report:
- What stage and task was in progress
- What the decision is
- What options exist
- Which tasks are complete so far

The user resolves the issue and re-runs `/feature-implement` on the same feature. Stage 2 will skip already-completed tasks (checks `[x]` in tasks.md).

---

## Constraints

- **Read before writing** — Always read a file before modifying it. Understand the context.
- **One task at a time** — Complete each task fully before starting the next. No partial implementations.
- **Follow the plan** — Execute what's in tasks.md. Don't add features, don't skip tasks, don't change the design. If the plan is wrong, EXIT with USER_BLOCKED.
- **Follow CLAUDE.md** — All code must comply with Code Quality Rules, Anti-Slop Rules, and Incremental Cleanup Policy.
- **Opportunistic quality fixes** — Every issue you notice during implementation must be acted on: fix it directly (if small and confident) or spawn a tech-debt feature (if large or uncertain). Silent ignoring is never acceptable. Both outcomes must appear in the Stage 6 report. The goal is gradual, steady improvement — the implementation is never blocked, but issues are never dropped.
- **No interactive commands** — Don't use `git add -i`, `git rebase -i`, or any command requiring interactive input.
- **Commit message format** — Use conventional commits (`feat:`, `fix:`, `chore:`). Include the session URL.
