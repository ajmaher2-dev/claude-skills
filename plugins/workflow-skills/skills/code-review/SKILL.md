---
name: code-review
description: Post-implementation code review — checks quality, patterns, tests, docs, and incremental cleanup opportunities on changed files. Lightweight alternative to full codebase-review.
user_invocable: true
---

# /code-review

Post-implementation code review for changed files. Checks code quality, project patterns, test completeness, documentation, and identifies incremental cleanup opportunities in touched files.

This skill is **repo-agnostic.** It reads `.claude/repo-conventions.yaml` (schema: `~/.claude/repo-conventions-schema.md`) before running. Pattern checks (repository_layer, service_layer, adapter_pattern, event_bus, etc.) **fire only if declared and not `false`** in the conventions file.

## Usage

```
/code-review                                    # review git diff (staged + unstaged)
/code-review <path1> <path2> ...                # review specific files
/code-review --feature <feature-path>           # review against a feature's tasks.md
```

## Execution

Runs 4 phases sequentially. Fixes issues directly — does not just report them.

---

## Phase 1: Determine Scope

**No args:** Get changed files via `git diff --name-only HEAD` and `git diff --name-only --cached`. If no changes, fall back to the last commit: `git diff --name-only HEAD~1..HEAD`. If still nothing, tell the user and stop.

**Paths provided:** Review those specific files.

**`--feature <path>`:** Read `<path>/tasks.md` to get the task list. Derive the files from the tasks (file paths mentioned in task descriptions). Also get changed files from git diff for cross-reference.

Separate files into:
- **Implementation files** — `.py`, `.ts`, `.tsx`, `.yaml` (not tests, not docs)
- **Test files** — files in `tests/` or `__tests__/` directories
- **Doc/config files** — `.md`, `.json`, other

Print the scope:
```
## Code Review Scope
- Implementation files: N
- Test files: N
- Doc/config files: N
```

---

## Phase 2: Code Quality Review

For each implementation file, read the full file (or the changed sections if the file is large) and check:

### 2a. Project Pattern Compliance

Read `CLAUDE.md` first AND `.claude/repo-conventions.yaml`. Then verify (each check fires only if its pattern is declared and not `false` in conventions):

- [ ] **Repository pattern** — Data access through `<backend.patterns.repository_layer>/<entity>_repo.py`, not direct DB queries in routes/services *(skip if not declared)*
- [ ] **Service layer** — Business logic in `<backend.patterns.service_layer>/<entity>_service.py`, not in route handlers *(skip if not declared)*
- [ ] **Settings singleton** — Uses the project's settings singleton from `<backend.config_module>`, not direct env reads *(always check; substitute the actual import)*
- [ ] **Adapter pattern** — External calls through `<backend.patterns.adapter_pattern>/`, not inline HTTP/SDK calls *(skip if not declared)*
- [ ] **Event bus** — Events published via the project's event-bus mechanism (outbox/relay/etc.) with types from `<backend.events_module>` *(skip if `event_bus` not declared)*
- [ ] **Logging** — Per the project's logging convention (read CLAUDE.md or equivalent — e.g., for kristallklar: BaseAgent subclasses use `self._logger`, standalone modules use `structlog.get_logger()`)
- [ ] **Error handling** — Typed exceptions at service boundaries; `except Exception` only in top-level handlers (middleware, agent event loops, async-runtime wrappers like Temporal)
- [ ] **Event payload extraction** — Uses the project's payload-helper convention if declared (e.g., `require_payload_field()` in kristallklar)

### 2b. Code Quality Checks

- [ ] **Broad exception handling** — `except Exception` in business logic (not top-level handlers)
- [ ] **N+1 queries** — DB queries inside loops that could be batched
- [ ] **Untyped parameters** — Public function params missing type annotations
- [ ] **Stringly-typed values** — Raw strings where constants/enums exist in the codebase
- [ ] **Duplicate logic** — Same pattern repeated that should be extracted
- [ ] **Unused imports** — Run `ruff check` on changed files and report
- [ ] **Dead code** — Functions/methods defined but never called
- [ ] **Missing input validation** — API endpoints accepting unvalidated input

### 2c. Anti-Slop Check (from CLAUDE.md)

- [ ] Restated comments (comment says what the next line does)
- [ ] Hedging language in comments ("should work", "probably", "for now")
- [ ] Buzzword comments ("robust", "scalable", "enterprise-grade")
- [ ] Docstrings that restate the signature
- [ ] Defensive None checks on typed parameters
- [ ] Placeholder implementations (`raise NotImplementedError`, empty returns)
- [ ] Vague TODOs without ticket/condition reference
- [ ] Debug artifacts (`print()`, `breakpoint()`, `console.log()`)

### 2d. Incremental Cleanup (per CLAUDE.md Incremental Cleanup Policy)

For each touched file, scan the ENTIRE file (not just changed lines) for:
- Pre-existing broad `except Exception` in non-top-level code
- Pre-existing N+1 patterns
- Pre-existing duplicate DB queries
- Pre-existing unused imports
- Pre-existing dead code in the same file

**Only flag if the fix is:**
- Self-contained within the file
- Low-risk (no DB schema changes, no API contract changes)
- Obvious (clear what the fix should be)

Do NOT flag:
- Issues requiring cross-file refactors
- Issues requiring DB migrations
- Issues that would change API response shapes
- Issues in files you didn't touch

---

## Phase 3: Test & Documentation Completeness

### 3a. Test Coverage Check

For each implementation file, verify corresponding tests exist:

- [ ] **Unit test exists** — `<test_dir>/unit/test_<module>.py` or per the project's test convention
- [ ] **Happy path tested** — At least one test for the success case
- [ ] **Error paths tested** — Tests for failure/exception cases
- [ ] **Edge cases tested** — Boundary conditions (empty input, null values, max lengths)
- [ ] **Frontend tests exist** — If file is under `<frontend.dirs>`, check the project's frontend test convention (e.g., `__tests__/`, sibling `.test.tsx`, etc.)

For API route changes:
- [ ] **API contract tests** — Request/response schema validation

For new endpoints:
- [ ] **Auth coverage** — Test that unauthenticated requests are rejected

### 3b. Test Quality Assessment

For each test file in scope, assess whether the tests are well-structured, correctly placed, and non-redundant:

- [ ] **Value check** — Every test has meaningful assertions that verify observable behavior, not implementation details. Flag tests with no assertions, trivial assertions (`assertTrue(True)`, `assert result is not None` when the real contract is richer), or tests that only verify mock call counts without checking outcomes.
- [ ] **Correctness check** — Assertions match actual business rules. Flag tests where the expected value looks wrong, arbitrary, or copied from implementation output without validation (e.g., asserting a magic number that came from running the code once rather than from a spec).
- [ ] **Level placement** — Each test runs at the appropriate level:
  - Pure logic (no I/O, no DB, no external calls) → unit test
  - Multi-component interaction with real DB → integration test
  - HTTP request/response contract validation → API test
  - Flag tests that use full DB setup to test a pure function (should be unit), or tests that heavily mock internal components when a real integration test would be more valuable and less brittle.
- [ ] **Efficiency** — Flag tests with disproportionate setup relative to what they verify: full fixture graphs created for a single-field assertion, repeated expensive DB setup that could use shared fixtures or `@pytest.mark.parametrize`, unnecessary service instantiation for testing a static/pure method.
- [ ] **Deduplication** — Flag tests that assert the same behavior as another test in the same or a different test file. Includes: identical scenarios with different names, overlapping integration and unit tests covering the exact same logic path with no additional value at the second level, parameterizable cases written as separate test functions.

### 3c. Documentation Check

- [ ] **Functional spec** — If user-facing behavior changed AND `<docs.functional_spec>` is declared, is it updated?
- [ ] **Technical spec** — If architecture changed AND `<docs.technical_spec>` is declared, is it updated?

### 3d. Feature Task Completeness (only with `--feature` flag)

Read `<feature-path>/tasks.md`. For each task:
- Check if the task's deliverable exists in the codebase (file created, function added, test written)
- Mark as DONE or NOT DONE
- Report completion percentage

Also check post-implementation tasks:
- [ ] Proposal frontmatter updated (`status: done`, `start`/`end` dates)
- [ ] Epic README status table updated
- [ ] Feature moved to `backlog/_completed/` (or flagged as pending)

---

## Phase 4: Fix & Report

### Step 1: Fix issues

Fix all issues found in Phases 2-3 directly. Group fixes by type:

1. **Code quality fixes** — Apply immediately (narrow exceptions, add types, remove dead code)
2. **Incremental cleanup** — Apply if low-risk per the policy
3. **Test gaps** — Create missing test files/cases
4. **Test quality fixes** — Move misleveled tests, merge duplicates, remove valueless tests, fix incorrect assertions
5. **Doc updates** — Update frontmatter, READMEs

If a fix is ambiguous or carries risk, skip it and include in the report.

### Step 2: Run linter

After all fixes, run `ruff check` on all changed `.py` files. Fix any new issues introduced.

### Step 3: Produce report

```markdown
## Code Review Report

### Scope
- Files reviewed: N
- Implementation: N | Tests: N | Docs: N

### Fixes Applied
| # | File | Issue | Fix |
|---|------|-------|-----|
| 1 | <file:line> | <issue> | <what was fixed> |

### Skipped (needs human input)
| # | File | Issue | Why skipped |
|---|------|-------|-------------|
| 1 | <file:line> | <issue> | <risk or ambiguity> |

### Incremental Cleanup Applied
| # | File | Pre-existing Issue | Fix |
|---|------|-------------------|-----|
| 1 | <file:line> | <issue> | <what was cleaned up> |

### Test Coverage
| Module | Unit | Integration | Error Paths | Edge Cases |
|--------|------|-------------|-------------|------------|
| <name> | ✓/✗ | ✓/✗/N/A | ✓/✗ | ✓/✗ |

### Test Quality
| # | File | Issue | Action |
|---|------|-------|--------|
| 1 | <file:line> | <value/correctness/level/efficiency/dedup issue> | <fixed/skipped — detail> |

### Feature Task Completion (if --feature)
| # | Task | Status |
|---|------|--------|
| 1 | <description> | DONE/NOT DONE |

**Completion:** N/M tasks done (N%)

### Verdict
- **CLEAN** — No issues found or all issues fixed
- **NEEDS_WORK** — Issues found that need human input (listed in Skipped)
```

### Step 4: Commit fixes

If any code changes were made, stage and commit with message:
```
fix: code review — <summary of fixes>
```

Do NOT push — let the user review the commit first.

---

## Constraints

- **Read before editing** — Always read the full file (or relevant section) before making changes.
- **Conservative fixes** — Only fix what you're confident about. When in doubt, skip and report.
- **No scope creep** — Don't refactor working code that isn't flagged by the checks. The incremental cleanup policy applies only to pre-existing issues in touched files.
- **Lint after fixing** — Always run `ruff check` after making changes to catch regressions.
- **Don't break tests** — If a fix would require test updates, include the test updates in the same fix.
