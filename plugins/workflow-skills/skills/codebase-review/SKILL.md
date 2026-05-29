---
name: codebase-review
description: Run a comprehensive codebase review across architecture, security, code quality, testing, operations, and long-term maintenance. Produces findings, health scores, trend analysis against prior reviews, and implementation plans.
user_invocable: true
---

# /codebase-review

Run a full, autonomous codebase review across six quality dimensions. Produces a timestamped review report with scored findings, dispositions, trend analysis, and implementation plans.

This skill is **repo-agnostic.** It reads `.claude/repo-conventions.yaml` (schema: `~/.claude/repo-conventions-schema.md`) at Step 0 and parameterises the discovery inventory, dimension reviews, and test path templates against the active repo's conventions. Pattern checks (repository_layer, service_layer, adapter_pattern, event_bus, workflows) **fire only if declared and not `false`**.

## Usage

```
/codebase-review
```

## Execution Overview

This skill runs six phases end-to-end without stopping for user input. If ambiguity is encountered, make your best judgement, apply it, and document it in the synthesis report.

```
Phase 1: Discovery         — Auto-map codebase structure
Phase 2: Dimension Reviews  — 6 parallel agents reviewing the full codebase
Phase 3: Synthesis          — Merge, de-duplicate, prioritize, score
Phase 4: Plan Creation      — Generate plans from PLAN-disposition findings
Phase 5: Plan Validation    — Review each plan 3 times, improving per pass
Phase 6: Final Report       — Write all outputs to docs/reviews/YYYY-MM-DD/
```

---

## Phase 0: Read repo conventions

Read `.claude/repo-conventions.yaml` at the repo root. Cache values for `package`, `backend.*` (test_dir, api_main, config_module, events_module, migrations.dir, patterns.*), `frontend.*` (dirs, framework), `docs.*`, `backlog_root`. These define which areas the dimension agents scan and which patterns they check for.

If conventions file missing: auto-detect (package from `pyproject.toml`/`package.json`; frontend dirs from `package.json` + framework dependency); ask the user for what's still unclear; suggest authoring the conventions file at the end of the run.

Throughout this skill, `<package>`, `<test_dir>`, `<frontend.dirs>`, etc. refer to values from the conventions file.

## Phase 1: Discovery

**Goal:** Build a live map of the codebase so dimension reviewers know what exists. Do NOT hardcode file lists — scan dynamically using the conventions-derived paths.

### Check for prior reviews

Before scanning, check whether prior reviews exist in `docs/reviews/`. If a prior review folder exists (e.g., `docs/reviews/2025-12-01/`), read its `synthesis.md` to extract:
- Prior health scores per dimension
- Prior finding counts by priority and disposition
- Plan topics generated and their current status (check `backlog/` for `status: done` in frontmatter, or `docs/plans/completed/` for archived plans)

Store this as `prior_review` context. It will be passed to dimension agents for trend comparison and used in Phase 3 for delta analysis. If no prior review exists, note "First review — no baseline" and proceed.

### What to scan

Use the Glob and Grep tools (NOT bash find/grep) to build a structured inventory:

| Area | How to scan |
|------|-------------|
| **Backend modules** | `<package>/` — list all top-level packages, count source files per package |
| **API layer** | `<backend.api_main>` and the rest of the API package (e.g., `<package>/api/routes/`, `schemas/`, `services/`, `repositories/`) — adapt to the project's actual API layout |
| **Database** | Project's models directory (e.g., `<package>/models/` or `<package>/db/models/`); `<backend.migrations.dir>` if declared — count migrations, find latest |
| **Workflows** | `<backend.patterns.workflows_dir>` if `workflows` declared — list workflow and activity files |
| **Adapters** | `<backend.patterns.adapter_pattern>` if declared — list external integrations |
| **LLM** | Project's LLM module if it has one (e.g., `<package>/llm/`) — list providers, tracing |
| **Events** | `<backend.events_module>` if declared — list event types |
| **Config** | `<backend.config_module>` and surrounding config package — settings, feature flags |
| **Legacy** | Any legacy code directories the project has (e.g., a `core/` folder pre-migration). Skip if no legacy area exists. |
| **Frontend** | For each `<frontend.dirs[*]>/src/` — list top-level directories, count component files |
| **Tests** | `<test_dir>/<test_layers[*]>/` for backend; for frontend, follow the project's frontend test convention (e.g., `<frontend.dirs[*]>/src/**/*.test.*`) — count test files per directory |
| **Infrastructure** | `docker-compose*.yml`, `.github/workflows/`, language-specific manifests (`pyproject.toml`, `package.json`, `go.mod`, etc.), migration tool config (`alembic.ini` if Alembic) |
| **Prompts** | Project's prompts directory if it has one (e.g., `prompts/`) — list prompt template files |
| **Backlog** | `<backlog_root>/` — list epics, count features per epic, check feature statuses in frontmatter |
| **Docs** | `docs/` — list top-level structure |

### Output

Write the inventory to `docs/reviews/YYYY-MM-DD/discovery.md`:

```markdown
# Codebase Discovery — YYYY-MM-DD

## Prior Review
- **Last review:** <YYYY-MM-DD or "None — first review">
- **Overall score then:** <N/10 or "N/A">
- **Plans generated then:** <N, with M completed, K still active>
- **Open findings carried forward:** <N>

## Summary
- Backend packages: N
- API routes: N files
- Database models: N classes across N files
- Migrations: N (latest: NNN_description)
- Workflows: N files
- Test files: N unit, N integration, N acceptance, N frontend
- Frontend components: ~N files
- PWA components: ~N files

## Detailed Inventory
<structured listing per area>
```

---

## Phase 2: Dimension Reviews

Launch **6 parallel agents** (using the Agent tool with `subagent_type: "general-purpose"`), one per dimension. Each agent receives:
1. The discovery inventory from Phase 1
2. Its dimension-specific review instructions (below)
3. Access to read any file in the codebase
4. If a prior review exists (see Phase 1 trend tracking), the prior dimension report for comparison

Each agent writes its findings to a dimension report file. The agent must return its full findings content so you can write the file.

**IMPORTANT:** Tell each agent to **read actual source files** to verify its findings. Do not accept findings based on file names or assumptions alone. Each agent should read at least the key files in its dimension and sample files from other areas.

**IMPORTANT:** Tell each agent the project context. Synthesise this from the conventions file and any technical-spec doc declared in `<docs.technical_spec>`. Include: language(s), key frameworks (web, ORM, async runtime), DB, frontend stack, multi-tenancy approach if applicable. Example for a Python/FastAPI project: "FastAPI + SQLAlchemy + PostgreSQL + Temporal + React SaaS platform. Multi-tenancy via database-level isolation. See `<docs.technical_spec>` for full architecture." Adapt the description to the actual stack from conventions.

### Dimension 1: Architecture & Structure

**Agent instructions:**

Review the entire codebase for architectural health. Read key files across all packages.

**Check for:**

1. **Pattern consistency** — Does all code follow the established patterns? (Check only patterns declared in `.claude/repo-conventions.yaml` `backend.patterns`):
   - Repository pattern: data access through `<backend.patterns.repository_layer>/*_repo.py`, not direct DB queries in routes/services *(skip if not declared)*
   - Service layer: business logic in `<backend.patterns.service_layer>/*_service.py`, not in route handlers *(skip if not declared)*
   - Adapter pattern: external calls through `<backend.patterns.adapter_pattern>/`, not inline *(skip if not declared)*
   - Settings: project's settings singleton from `<backend.config_module>`, not direct env var reads
   - Events: published via the project's event bus with types from `<backend.events_module>` *(skip if `event_bus` not declared)*

2. **Module organization** — Are files in the right place?
   - Business logic in routes (should be in services, if service layer pattern is declared)
   - Data access in services (should be in repositories, if repository pattern is declared)
   - Legacy code in pre-migration locations that should have moved to the canonical package
   - Misnamed files not following conventions (`*_repo.py`, `*_service.py` if those patterns declared)

3. **Dependency direction** — Do dependencies flow correctly?
   - Routes → Services → Repositories → Models (never reversed)
   - Domain modules should not import directly from each other without going through services
   - Circular import risks

4. **Code reuse & duplication** — Is code properly shared?
   - Duplicate logic across modules (same query pattern, same validation, same transformation)
   - Copy-pasted code blocks that should be abstracted
   - Parallel implementations (legacy code path + canonical-package code path)
   - Utility functions duplicated across files

5. **Dead code** — What can be removed?
   - Unused functions, methods, classes (defined but never called)
   - Unused imports
   - Commented-out code blocks
   - Orphaned files not imported anywhere
   - Deprecated compatibility shims no longer needed

6. **Module isolation** — Can modules be toggled independently?
   - Hard coupling between domain modules
   - Route registration that can't be conditionally excluded
   - FK constraints that break if a module's data is absent

**Output format:** Use the standard dimension report format (see Phase 2 Output Format below).

### Dimension 2: Security

**Agent instructions:**

Review the entire codebase for security vulnerabilities and gaps. Focus on OWASP Top 10 and application-specific risks.

**Check for:**

1. **Authentication & Authorization**
   - Are all API endpoints protected with auth dependencies?
   - Are there endpoints missing ownership validation (user can access other users' data)?
   - Is role-based access control consistently enforced?
   - Are JWT tokens properly validated (expiry, signature, audience)?

2. **Input validation**
   - Are all user inputs validated via Pydantic schemas before processing?
   - Are there raw request body accesses bypassing schema validation?
   - Are file uploads validated (type, size, content)?
   - Are search queries using ILIKE properly escaped to prevent SQL injection?

3. **Data access scoping**
   - Do repository queries filter by the authenticated user's scope?
   - Can a user enumerate or access resources belonging to other users?
   - Are bulk operations (list, export, delete) properly scoped?

4. **Secrets management**
   - Are secrets stored in settings with `SecretStr` type?
   - Are there hardcoded credentials, API keys, or tokens in source code?
   - Are secrets accidentally logged or included in error responses?

5. **Injection vulnerabilities**
   - SQL injection: raw SQL queries with string interpolation
   - Command injection: `subprocess` or `os.system` with user input
   - XSS: unescaped user content in frontend templates
   - Path traversal: file operations with user-supplied paths

6. **Sensitive data handling**
   - PII redaction in logs and error responses
   - Sensitive fields excluded from API responses
   - Proper error messages (no stack traces or internal details to clients)

7. **Frontend security**
   - CSRF protection
   - Content Security Policy headers
   - Secure cookie flags (HttpOnly, Secure, SameSite)
   - Auth token storage (localStorage vs httpOnly cookies)

**Output format:** Use the standard dimension report format.

### Dimension 3: Code Quality

**Agent instructions:**

Review the entire codebase for clean code practices, maintainability, and technical debt.

**Check for:**

1. **Clean code principles**
   - Single responsibility: functions/classes doing too many things
   - Function length: methods over ~50 lines that should be decomposed
   - File size: files over ~500 lines that should be split
   - Nesting depth: deeply nested conditionals (>3 levels)
   - Naming: unclear variable/function/class names

2. **Error handling**
   - Bare `except:` or `except Exception:` swallowing errors silently
   - Missing error handling on external calls (API, DB, file I/O)
   - Inconsistent error response formats across endpoints
   - Error messages that leak implementation details

3. **Type safety**
   - Missing type annotations on public functions/methods
   - `Any` types where specific types are known
   - Dict/list used where dataclasses/Pydantic models should be
   - Type mismatches between layers (schema vs model vs API response)

4. **Code smells**
   - God classes/functions that do too much
   - Feature envy: methods that use more data from another class than their own
   - Primitive obsession: using raw strings/ints where domain types should exist
   - Magic numbers/strings without named constants
   - Mutable default arguments

5. **Logging & observability**
   - Consistent structured logging (structlog) vs print/basic logging
   - Appropriate log levels (debug vs info vs warning vs error)
   - Key operations that lack logging entirely
   - Log messages that are too verbose or too sparse

6. **Consistency**
   - Mixed patterns for the same concern across modules
   - Inconsistent naming conventions within the same layer
   - Different error handling strategies across similar code paths

7. **AI-generated code patterns (slop detection)** — Check for patterns that indicate low-quality AI-generated code. Reference the Anti-Slop Rules in `CLAUDE.md` for the full list. Key patterns to scan for:
   - **Restated comments** — comments that just rephrase the next line of code (`# Get the user` above `user = get_user()`)
   - **Hedging language** — "should work", "probably", "for now", "hopefully", "might need to" in comments
   - **Buzzword comments** — "robust", "scalable", "enterprise-grade", "production-ready", "leverages", "best practices" in comments/docstrings
   - **Docstrings that restate the signature** — `def get_user(): """Get the user."""`
   - **Broad except Exception in business logic** — only acceptable in top-level handlers (middleware, agent loops, Temporal wrappers)
   - **Defensive None/isinstance checks on typed parameters** — checking `if x is None` when the type annotation guarantees it's not None
   - **Empty model/schema bodies** — Pydantic models with just `pass` and a docstring
   - **Stub handlers** — signal/event handlers that only log with a TODO comment
   - **Cross-language contamination** — JavaScript/Java idioms in Python (`.push()`, `.length`, `===`, `.forEach()`, `.equals()`, `null` in comments)
   - **Placeholder implementations** — functions that `raise NotImplementedError` or return empty dicts/lists as stubs
   - **Vague TODOs** — `# TODO: fix this` without a ticket reference or concrete action
   - **Debug artifacts** — `print()`, `breakpoint()`, `pdb.set_trace()`, `console.log()` in production code
   - **Speculative abstractions** — single-method utility classes, wrapper functions that just forward args, parameters/branches for hypothetical future requirements

**Output format:** Use the standard dimension report format.

### Dimension 4: Testing

**Agent instructions:**

Review the test suite for coverage, quality, and completeness across all test levels.

**Check for:**

1. **Test level completeness** — For each backend module, verify the levels declared in `backend.test_layers` exist:
   - Unit tests (`<test_dir>/unit/test_*.py`) — isolated, mocked dependencies
   - Integration tests (`<test_dir>/integration/test_*.py`) — real DB, mocked external services
   - API/contract tests — endpoint request/response validation (if API exists)
   - Migration tests — upgrade/downgrade verification (only if `backend.migrations` declared)
   - Frontend tests — component and integration tests (only if `frontend` declared, per the project's frontend test convention)

2. **Coverage gaps** — Identify modules/services/repos that have:
   - No tests at all
   - Only happy-path tests (no error cases)
   - No boundary/edge case tests (empty lists, null values, max lengths)
   - No tests for authorization/ownership validation
   - No tests for concurrent access or race conditions (where relevant)

3. **Test quality**
   - Tests that test implementation rather than behavior (brittle mocks)
   - Tests with no assertions (passing but verifying nothing)
   - Tests with too many assertions (testing multiple behaviors in one test)
   - Copy-pasted test code that should use parameterize/fixtures
   - Test names that don't describe the scenario (`test_1`, `test_thing`)

4. **Regression gaps**
   - Features/bug fixes without corresponding regression tests
   - State machine transitions without exhaustive transition tests
   - API endpoints without error response tests
   - Event handlers without failure mode tests

5. **Test infrastructure**
   - Factory completeness: do test factories cover all models?
   - Fixture quality: are fixtures reusable and well-scoped?
   - Test isolation: can tests run independently and in any order?
   - Test performance: are there slow tests that should be optimized?

6. **Frontend test coverage**
   - Components without any tests
   - User interactions without test coverage
   - API integration tests (mocked API responses)
   - Accessibility testing gaps

**Output format:** Use the standard dimension report format.

### Dimension 5: Operations & Infrastructure

**Agent instructions:**

Review the operational readiness, infrastructure configuration, CI/CD, and deployment setup.

**Check for:**

1. **Docker & deployment**
   - Docker image security (base image currency, non-root user, no secrets in image)
   - Docker Compose service configuration (health checks, restart policies, resource limits)
   - Environment variable management (all required vars documented, defaults sensible)
   - Multi-stage builds for smaller production images

2. **CI/CD pipeline**
   - Test automation in CI (all test levels run?)
   - Linting and type checking in CI
   - Security scanning (dependency vulnerabilities, SAST)
   - Build artifact management
   - Deployment automation and rollback strategy

3. **Database operations**
   - Migration safety (can migrations run without downtime?)
   - Backup and restore procedures
   - Connection pooling configuration
   - Query performance (N+1 queries, missing indexes, slow queries)

4. **Monitoring & alerting**
   - Health check endpoints (liveness, readiness)
   - Prometheus metrics coverage
   - Log aggregation setup
   - Error alerting (Slack integration, alert fatigue risk)
   - Uptime monitoring

5. **Migration quality**
   - All migrations have both `upgrade()` and `downgrade()`
   - Data migrations separated from schema migrations
   - Concurrent index creation for large tables
   - No breaking changes in migration chain

6. **Dependency management**
   - Outdated dependencies with known vulnerabilities
   - Pinned vs unpinned dependency versions
   - Unused dependencies that can be removed
   - License compliance

**Output format:** Use the standard dimension report format.

### Dimension 6: Maintenance & Lifecycle

**Agent instructions:**

Review the codebase for long-term maintainability, sustainability, and operational lifecycle health. This dimension looks beyond current correctness to assess whether the application will remain healthy over months and years.

**Check for:**

1. **Dependency freshness & upgrade readiness**
   - How far behind are major framework versions (FastAPI, SQLAlchemy, React, Python, Node)?
   - Are there dependencies with known EOL dates or approaching end-of-support?
   - Is the dependency tree pinned in a way that allows controlled upgrades (not overly broad or overly tight)?
   - Are there deprecated API usages that will break on the next major version of a dependency?
   - Is there a pattern for how upgrades have been handled historically (changelogs, migration guides referenced)?

2. **Deprecation tracking**
   - Are there deprecated functions, endpoints, or modules with no removal timeline?
   - Are `# deprecated` or `DeprecationWarning` markers used, and do they include a target removal version or date?
   - Are there compatibility shims or backward-compat code that has outlived its usefulness?
   - Do API endpoints have versioning or sunset headers?
   - Are there database columns/tables marked for removal but still present?

3. **Data lifecycle & growth**
   - Are there tables that grow unboundedly without retention policies (logs, traces, events, audit records)?
   - Is there an archival or purge strategy for historical data?
   - Are large tables indexed appropriately for their growth trajectory?
   - Are there materialized views, caches, or aggregation tables that need periodic refresh?
   - Is storage cost or DB size a growing concern based on data model design?

4. **API versioning & backward compatibility**
   - Are API contracts documented and versioned?
   - Are there breaking changes to API schemas without version bumps?
   - Are event bus schemas versioned or do consumers handle schema evolution?
   - Are webhook/callback contracts stable or evolving without notice to consumers?
   - Is there a strategy for deprecating and removing old API versions?

5. **Operational burden assessment**
   - How many cron jobs, scheduled tasks, or background workers exist and are they documented?
   - Are there manual operational procedures that should be automated?
   - What is the on-call surface area — how many services/components can page?
   - Are health checks comprehensive enough to detect degradation before outage?
   - Are runbooks or operational playbooks present for failure scenarios?

6. **Configuration & environment drift**
   - Are all environment variables documented with their purpose and defaults?
   - Are there environment-specific configurations that could cause test/prod divergence?
   - Is the settings model growing unwieldy (too many flags, unclear groupings)?
   - Are feature flags cleaned up after rollout, or do they accumulate?

7. **Knowledge & onboarding sustainability**
   - Are complex business logic areas documented or self-documenting?
   - Are there bus-factor risks (code areas only one person understands)?
   - Is the architecture documented well enough for a new developer to become productive?
   - Are naming conventions and patterns consistent enough to be discoverable without tribal knowledge?

**Output format:** Use the standard dimension report format.

### Phase 2 Output Format

Each dimension agent must return findings in this exact format:

```markdown
# <Dimension Name> Review — YYYY-MM-DD

## Health: N/10

## Summary
<3-5 sentences: overall assessment, key strengths, key concerns>

## Findings

| # | Finding | File(s) | Priority | Disposition | Effort | Plan Ref |
|---|---------|---------|----------|-------------|--------|----------|
| 1 | <description> | <file:line> | P0/P1/P2/P3 | PLAN/QUICK_FIX/ACCEPT/MONITOR | Xh/Xd | <topic or —> |

## Finding Details

### F1: <Finding title>
**Priority:** P0/P1/P2/P3
**Disposition:** PLAN / QUICK_FIX / ACCEPT / MONITOR
**Files:** <file paths with line numbers>
**Evidence:** <what was found — quote or describe the actual code>
**Recommendation:** <specific fix — what to change and where>
**Rationale for disposition:** <why this disposition was chosen>
```

**Priority definitions:**
- **P0** (redesign) — fundamentally wrong, needs architectural rethinking
- **P1** (critical) — will cause production incidents, data loss, or security vulnerabilities
- **P2** (significant) — actively hinders development velocity or creates ongoing tech debt
- **P3** (minor) — cleanup, readability, consistency improvements

**Disposition definitions:**
- **PLAN** — needs structured implementation (generates a feature plan in `backlog/tech_debt/<topic>/`)
- **QUICK_FIX** — <30 min, obvious fix, can be done ad-hoc
- **ACCEPT** — known trade-off, not worth fixing now (document rationale)
- **MONITOR** — not broken yet but trending wrong, flag for next review

**Health score guidance:**
- 9-10: Excellent — no P0/P1, few P2, production-ready
- 7-8: Good — no P0, minor P1s addressed, some P2 tech debt
- 5-6: Fair — has P1 issues or significant P2 accumulation
- 3-4: Concerning — P0 or multiple P1 issues, significant gaps
- 1-2: Critical — fundamental architectural problems, security holes

---

## Phase 3: Synthesis

After all 6 dimension agents complete, merge their findings into a single synthesis report.

### Step 1: Collect all findings
Read all 6 dimension reports. Build a unified findings list.

### Step 2: De-duplicate
- Identify findings that appear in multiple dimensions (e.g., "missing input validation" in both Security and Code Quality)
- Merge duplicates into a single finding, keeping the highest priority and noting all affected dimensions
- If two findings describe the same root cause from different angles, consolidate into the root cause

### Step 3: Resolve conflicts
- If two dimensions recommend different fixes for the same problem, choose the approach that aligns with established project patterns
- Document the resolution rationale

### Step 4: Group into plan topics
- Cluster related PLAN-disposition findings into coherent implementation topics
- Each topic should be a cohesive unit of work (not too granular, not too broad)
- A good topic groups 3-8 related findings that would be fixed together
- Name topics descriptively: e.g., `api-input-validation-hardening`, `test-coverage-email-module`, `dead-code-cleanup`
- **Scope check:** Each topic should be **one independent deliverable** — independently mergeable, not a slice that's meaningless without another. Split by subsystem when findings span unrelated areas with no shared code path. Task count past the repo's `backlog.feature_size.goal_max_tasks` goal (default ~10) is a soft signal to re-check that — not a hard limit.
- **Quick-fix bundle:** After grouping PLAN-disposition findings, also create a single `review-quick-fixes` plan topic that bundles all QUICK_FIX-disposition findings into one deliverable feature. This ensures quick fixes get tracked as a proper backlog item with proposal, design, and tasks — not just a list in the synthesis report.

### Step 5: Trend analysis (if prior review exists)

If `prior_review` context was captured in Phase 1, compute deltas:

- **Score deltas** per dimension (e.g., Security: 6→8, +2)
- **Finding resolution rate** — how many findings from the prior review have been addressed (check if their referenced files/issues have been fixed, or if related plans moved to `docs/plans/completed/`)
- **New findings** — findings in this review that didn't exist in the prior review
- **Recurring findings** — findings that appeared in the prior review and are still present (stale issues)
- **Regressions** — areas where the score dropped or new P0/P1 findings appeared in a previously healthy dimension

This analysis goes into the synthesis report's Trend section.

### Step 6: Write synthesis report

Write to `docs/reviews/YYYY-MM-DD/synthesis.md`:

```markdown
# Codebase Review Synthesis — YYYY-MM-DD

## Health Scorecard

| Dimension | Score | Prior | Delta | Trend |
|-----------|-------|-------|-------|-------|
| Architecture & Structure | N/10 | N/10 or — | +/-N or — | improved/stable/regressed/new |
| Security | N/10 | N/10 or — | +/-N or — | ... |
| Code Quality | N/10 | N/10 or — | +/-N or — | ... |
| Testing | N/10 | N/10 or — | +/-N or — | ... |
| Operations & Infrastructure | N/10 | N/10 or — | +/-N or — | ... |
| Maintenance & Lifecycle | N/10 | N/10 or — | +/-N or — | ... |
| **Overall** | **N/10** | **N/10 or —** | **+/-N or —** | ... |

## Executive Summary
<5-10 sentences: overall codebase health, top concerns, top strengths, recommended focus areas>

## Review Trend (omit section if first review)

### Resolution Progress
- **Prior findings:** N total
- **Resolved since last review:** N (N%)
- **Still open (recurring):** N — <list the most critical stale findings>
- **New this review:** N

### Regressions
<List any dimensions where score dropped, or new P0/P1 findings appeared in previously healthy areas. If none: "No regressions detected.">

### Recommended Review Cadence
<Based on the current health and rate of change, recommend a next review date. Factors: number of open P0/P1 findings, velocity of changes, number of active plans. Typical cadence: monthly for active development, quarterly for stable codebases.>

## Findings by Priority

### P0 — Redesign Required
| # | Finding | Dimensions | File(s) | Disposition | Plan Topic |
...

### P1 — Critical
...

### P2 — Significant
...

### P3 — Minor
...

## Statistics
- Total findings: N
- By priority: P0: N, P1: N, P2: N, P3: N
- By disposition: PLAN: N, QUICK_FIX: N, ACCEPT: N, MONITOR: N
- Plans to generate: N

## Plan Generation Manifest
| # | Plan Topic | Findings Covered | Priority Range | Estimated Effort |
|---|------------|-----------------|----------------|------------------|
| 1 | <topic> | F1, F5, F12 | P1-P2 | Xd |
...

## Accepted Risks
| # | Finding | Rationale |
...

## Items to Monitor
| # | Finding | What to watch | Next review action |
...

## Quick Fixes
| # | Finding | What to do | Effort |
...
```

---

## Phase 4: Plan Creation

For each plan topic in the Plan Generation Manifest, create a plan in `backlog/tech_debt/<topic>/` with three files following the project's established plan format.

### Plan format (same conventions as /feature-create)

Create three files per plan:

#### `proposal.md`

```markdown
---
epic: tech_debt
name: <Topic Human Name>
status: not-started
priority: <P0 | P1 | P2 — based on highest-priority finding in the topic>
effort: <Xd>
start:
end:
depends-on: <other review-generated plans that must complete first, or []>
owner: AJ
tags: [codebase-review, YYYY-MM-DD]
---

# <Topic Name> — Proposal

**Created:** YYYY-MM-DD
**Source:** Codebase review YYYY-MM-DD (findings: F1, F5, F12)

---

## Problem

<What is wrong, referencing specific files, functions, and evidence from the review findings. This should be concrete, not vague.>

## Proposed Solution

<High-level approach. What will change?>

## Scope

### In scope
- <item>

### Out of scope
- <item>

## Findings Addressed

| Finding | Priority | Dimension | Description |
|---------|----------|-----------|-------------|
| F1 | P1 | Security | <description> |
| F5 | P2 | Code Quality | <description> |

## RAID

### Risks
- **Risk:** <description> — **Likelihood:** <low/medium/high> — **Mitigation:** <what to do if it happens>

### Assumptions
- <assumption — things this plan takes for granted that, if wrong, would change the approach>

### Issues
- <known blocker with owner or resolution criteria, or "None">

### Open Questions
- <what is not yet decided>

## Decision Log

| Decision | Choice | Date | Rationale |
|---|---|---|---|
| | | | |
```

#### `design.md`

```markdown
# <Topic Name> — Technical Design

---

## Architecture

<How does this fit into existing architecture?>

### Reuse assessment
- **Existing services/repos that cover related functionality:** <list, or "None — new domain">
- **Existing adapters that can be reused:** <list, or "None needed">
- **Existing utilities/helpers that apply:** <list, or "None">
- **Code to deprecate/remove:** <if this replaces existing functionality, list what becomes dead code>

### Files to create
- <path> — <purpose> — <why an existing file can't be extended instead>

### Files to modify
- <path> — <what changes>

### Database changes (omit section if no migrations)
- Migration number: <next after current latest, read dynamically>
- New tables/columns: <describe>
- `down_revision`: <current latest migration revision hash>
- Rollback strategy: <what `downgrade()` does>

### API changes (omit section if no API changes)
<New or modified endpoints. Include method, path, request/response schemas.>
<If new routes: note registration in `api/main.py`>

### Event bus changes (omit section if no new events)
<New event types to add to `events/schemas.py`.>

## Dependencies

- **Python packages:** <any new dependencies to add to pyproject.toml>
- **External services:** <any new service dependencies>
- **Internal dependencies:** <which existing modules/services this depends on>

## Testing Strategy

Use a BDD-style approach: describe tests as behaviours ("Given X, When Y, Then Z") before specifying implementation.

### Behavioural scenarios (BDD)
- **Given** <precondition>, **When** <action>, **Then** <expected outcome>
- **Given** <error precondition>, **When** <action>, **Then** <expected error handling>

### Unit tests — `<test_dir>/unit/test_<module>.py`
- Happy path: <scenarios>
- Error paths: <scenarios>
- Boundary conditions: <scenarios>

### Integration tests — `<test_dir>/integration/test_<module>.py`
- <what to test>

### API/contract tests (omit section if no API changes)
- <endpoints to test>

### Migration tests (omit section if no DB changes)
- <migration test scenarios>

### Known constraints
<Substitute project-specific testing constraints from CLAUDE.md and the conventions file. For EmailPOC: conftest uses `_reset_singletons` autouse fixture; mock settings/DB/external services in unit tests; `import magic` crashes on Windows; pre-existing test failures listed in `<testing.pre_existing_failures>`. For other repos: read CLAUDE.md and the project's conftest to discover analogous constraints.>

## Security Considerations (omit section if purely tooling/config with no runtime impact)
- <address relevant concerns: input validation, ownership scoping, ILIKE escaping, sensitive field redaction>

## Observability
- Structured logging: <what gets logged, at what level>
- Metrics: <any Prometheus metrics to add>
- Error alerting: <Slack alerting for failures>

## Maintenance Implications (omit section if plan adds no ongoing operational burden)
- **New background jobs / scheduled tasks:** <any new recurring work — frequency and failure handling>
- **Data growth:** <unbounded data? retention/archival strategy?>
- **New external dependencies:** <what happens if they go down or change API?>
- **Deprecation impact:** <timeline and migration path for deprecated functionality>
- **Upgrade sensitivity:** <tight coupling to specific library versions?>
- **Operational surface:** <new things that can fail and page on-call?>
```

#### `tasks.md`

```markdown
# <Topic Name> — Tasks

**Status:** NOT_STARTED
**Last updated:** YYYY-MM-DD
**Source review:** docs/reviews/YYYY-MM-DD/

---

## Pre-implementation
- [ ] 1. Verify no conflicts with existing plans (checked during plan creation)
- [ ] 2. Review design with stakeholder (if applicable)

## Implementation
- [ ] 3. <Specific task with file path and function name>
- [ ] 4. <Next task>
- [ ] 5. <Continue numbering sequentially>

## Database (include only if migrations are needed)
- [ ] N. Create migration <NNN>_<description>.py
- [ ] N+1. Verify `downgrade()` works cleanly
- [ ] N+2. Run migration on test environment

## Testing
- [ ] N. Write unit tests — happy path scenarios
- [ ] N+1. Write unit tests — error paths and boundary conditions
- [ ] N+2. Write integration tests
- [ ] N+3. Write API/contract tests (if API changes)
- [ ] N+4. Write migration tests (if DB changes)
- [ ] N+5. Run full test suite — verify no regressions (exclude known pre-existing failures)

## Post-implementation
- [ ] N. Update `docs/general/FUNCTIONAL_SPECIFICATION.md` with new feature (only if user-facing functionality changed)
- [ ] N+1. Update `docs/general/TECHNICAL_SPEC.md` if architecture changed (omit for tooling/config-only plans)
- [ ] N+2. Update proposal.md frontmatter: `status: done`, set `end:` date
- [ ] N+3. Move to epic if grouping emerges, or archive to `docs/plans/completed/<topic>/` during periodic cleanup

## Post-deployment verification (include if plan adds new runtime behaviour)
- [ ] N. After deployment: verify metrics/logs are nominal for 24h
- [ ] N+1. After 2 weeks: review data growth and operational metrics — confirm Maintenance Implications assumptions hold
- [ ] N+2. Remove any temporary feature flags or compatibility shims
```

---

## Phase 5: Plan Validation (3 iterations)

For each generated plan, run **3 review passes**. Each pass checks and improves the plan. Do NOT invoke `/feature-review` as a separate skill — run the checks inline.

### Review checks (applied each pass)

**Completeness:**
- [ ] YAML frontmatter is present with all required fields: `epic`, `name`, `status`, `priority`, `effort`, `owner`, `depends-on`, `tags`
- [ ] Problem statement is specific (references actual files, not vague)
- [ ] Solution describes what will change
- [ ] Scope has in-scope and out-of-scope
- [ ] Effort estimate is present and reasonable
- [ ] Findings addressed table maps back to synthesis findings
- [ ] Files to create/modify list specific paths
- [ ] New files justify why existing modules can't be extended
- [ ] Testing strategy covers error paths and edge cases, not just happy path
- [ ] Testing tasks separated by layer (unit, integration, API)
- [ ] Tasks are specific enough to act on (file paths, function names)
- [ ] Tasks are in dependency order
- [ ] **RAID — Risks:** Present with likelihood and mitigation per risk
- [ ] **RAID — Assumptions:** Explicitly stated (flag if zero assumptions on a non-trivial plan)
- [ ] **RAID — Issues:** Known blockers called out with resolution criteria, or explicit "None"
- [ ] **RAID — Dependencies:** Internal (depends on / blocks) and external (APIs, services, teams) listed

**Accuracy:**
- [ ] All referenced file paths exist in the codebase
- [ ] All referenced models/services/repos exist
- [ ] Migration number is correct (next sequential after latest)
- [ ] No contradictions with other generated plans

**Alignment:**
- [ ] Repository pattern for data access
- [ ] Service layer for business logic
- [ ] Settings via `get_settings()` singleton
- [ ] Adapter pattern for external calls
- [ ] Naming conventions followed
- [ ] Security addressed (input validation, ownership scoping, ILIKE escaping)
- [ ] No new architectural patterns introduced

**Maintenance & backward compatibility:**
- [ ] Maintenance Implications section present if plan adds runtime behaviour (background jobs, data growth, new external dependencies)
- [ ] Data growth strategy specified for unbounded tables (not deferred indefinitely)
- [ ] Deprecation timeline stated if plan deprecates existing functionality
- [ ] Backward compatibility addressed if changing API schemas, event schemas, or DB schemas used by running instances
- [ ] Post-deployment verification tasks present if plan adds new runtime behaviour
- [ ] Feature flag / shim cleanup task present if temporary rollout mechanisms are added

**Coherence (cross-file consistency):**
- [ ] Every promise in proposal's "Proposed Solution" is delivered by at least one task
- [ ] Every file in design.md's "Files to create/modify" has a corresponding task
- [ ] Every migration in design has create, verify downgrade, and run tasks
- [ ] If RAID Issues list blockers, there are pre-implementation tasks to resolve them
- [ ] If RAID Risks have mitigations requiring work, they appear as tasks
- [ ] Effort estimate is sane relative to task count (~3-5 tasks per day)
- [ ] Maintenance Implications concerns have corresponding tasks (retention logic, monitoring, cleanup)

**Holistic assessment:**
- [ ] Completing all tasks actually solves the problem described in the proposal
- [ ] No broken intermediate states — task order won't leave the system broken between steps
- [ ] Nothing feels underspecified, overly optimistic, or hand-wavy
- [ ] Scope is one independent deliverable — flag if findings span unrelated subsystems that should be separate plans, or if task count past the repo's `feature_size` goal (default ~10) signals it isn't one coherent unit (soft check, not a hard limit)

### Iteration process

**Pass 1:** Run all checks. Fix any failures directly in the plan files. Log what was fixed.

**Pass 2:** Re-run all checks on the updated plans. Fix any remaining issues. Verify Pass 1 fixes didn't introduce new problems.

**Pass 3:** Final verification pass. All checks must pass. If any still fail, document them as known limitations in the plan's `proposal.md` under "RAID > Open Questions".

After 3 passes, append a validation summary to each plan's `proposal.md`:

```markdown
## Validation

**Reviewed:** 3 passes (codebase-review YYYY-MM-DD)
**Result:** PASS / PASS_WITH_NOTES
**Notes:** <any caveats from pass 3>
```

---

## Phase 6: Final Report

### Write all output files

Ensure the following files exist in `docs/reviews/YYYY-MM-DD/`:

```
docs/reviews/YYYY-MM-DD/
├── discovery.md              # Phase 1 output (includes prior review reference)
├── architecture.md           # Dimension 1 findings
├── security.md               # Dimension 2 findings
├── code-quality.md           # Dimension 3 findings
├── testing.md                # Dimension 4 findings
├── operations.md             # Dimension 5 findings
├── maintenance.md            # Dimension 6 findings
└── synthesis.md              # Merged findings, scores, trends, plan manifest
```

And feature plans in `backlog/tech_debt/<topic>/` (one per plan topic):

```
backlog/tech_debt/<topic>/
├── proposal.md    (with YAML frontmatter)
├── design.md
└── tasks.md
```

### Print final summary

After all files are written, output a summary to the user:

```markdown
## Codebase Review Complete — YYYY-MM-DD

### Health Scorecard
| Dimension | Score | Delta |
|-----------|-------|-------|
| Architecture & Structure | N/10 | +/-N or — |
| Security | N/10 | +/-N or — |
| Code Quality | N/10 | +/-N or — |
| Testing | N/10 | +/-N or — |
| Operations & Infrastructure | N/10 | +/-N or — |
| Maintenance & Lifecycle | N/10 | +/-N or — |
| **Overall** | **N/10** | **+/-N or —** |

### Trend (omit if first review)
- Prior review: YYYY-MM-DD
- Findings resolved since then: N/M (N%)
- Recurring (stale) findings: N
- Regressions: <list or "None">

### Findings
- Total: N findings
- By priority: P0: N | P1: N | P2: N | P3: N
- By disposition: PLAN: N | QUICK_FIX: N | ACCEPT: N | MONITOR: N

### Plans Generated
| # | Plan | Location | Effort | Validation |
|---|------|----------|--------|------------|
| 1 | <topic> | `backlog/tech_debt/<topic>/` | Xd | PASS |
...

### Quick Fixes (can be done now)
1. <description> — <file> — <effort>
...

### Next Steps
1. Review generated plans in `backlog/tech_debt/` — move into epics if grouping emerges
2. Prioritize and schedule plan implementation (update frontmatter `start`/`end` dates)
3. Address quick fixes ad-hoc
4. **Next review:** <recommended date based on cadence analysis>
```

---

## Execution Constraints

1. **Autonomous completion** — Run all 6 phases without stopping for user input. Make judgement calls on ambiguity and document them.
2. **Read actual code** — Every finding must reference actual files and code. Do not generate findings from assumptions or file names alone.
3. **No code changes** — This skill produces reports and plans only. It does not modify application source code, tests, or configuration.
4. **Use parallel agents** — Phase 2 must use parallel agents for the 6 dimensions. Phase 4 can create plans sequentially since they depend on synthesis output.
5. **Track progress** — Use TodoWrite to track which phases are complete.
6. **Date-stamped output** — Use today's date for the review folder name.
7. **Independent of existing plans** — Do not update or modify existing plans in `docs/general/module_implementation/`. Generated plans go into `backlog/tech_debt/`.
8. **Existing review prompts as reference only** — The module review prompts in `docs/general/module_review_prompts/` can be read for context by dimension agents, but this skill does not depend on them.
