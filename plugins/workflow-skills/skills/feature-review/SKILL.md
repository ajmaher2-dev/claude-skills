---
name: feature-review
description: Review one or more feature plans for completeness, accuracy against the codebase, and alignment with project conventions. Single plans are reviewed directly; multiple plans run an autonomous review→fix loop until convergence.
user_invocable: true
---

# /feature-review

Review one or more feature plans.

Every plan goes through the same review→fix loop until converged or user-blocked. Multiple plans are processed sequentially.

## Usage

```
/feature-review <path>
/feature-review backlog/<epic>/features/<feature-name>/
/feature-review <path1> <path2> <path3> ...
```

**All plans go through the [Batch Mode] review→fix loop.** No plan review is complete until it either passes cleanly twice in a row, or only USER items remain (confirmed twice in a row), or the 8-iteration cap is hit.

This skill is **repo-agnostic.** It reads `.claude/repo-conventions.yaml` at Step 0 (schema: `~/.claude/repo-conventions-schema.md`) and parameterises path checks, pattern alignment, and verification gates against the active repo's conventions.

---

## Single Plan Review

### Step 0 — Read repo conventions

Read `.claude/repo-conventions.yaml` at the repo root. Set working values for `package`, `backend.*` (test_dir, api_main, config_module, events_module, migrations.tool/dir, patterns.*), `frontend.*` (dirs, test_framework, visual_regression, visual_safety_net), `docs.*`, `ci.*`, `testing.runner`, `testing.pre_existing_failures`, `backlog_root`.

If the file is missing: auto-detect what you can (package from `pyproject.toml`/`package.json`, frontend dirs from `package.json` + framework dependency); ask the user the rest. Suggest authoring `.claude/repo-conventions.yaml` at the end of the run.

Throughout this skill, `<package>`, `<test_dir>`, `<frontend.dirs>`, etc. refer to the values read in this step. Pattern checks (repository_layer, service_layer, adapter_pattern, event_bus, workflows) **fire only if declared and not `false`** in the conventions file.

### Step 1 — Read the plan

Read all files in the specified plan folder:
- `proposal.md`
- `design.md`
- `tasks.md`

If any required file is missing, flag it immediately.

### Step 2 — Run completeness checks

Check each artifact for substance, not just headings:

**proposal.md:**
- [ ] **YAML frontmatter** is present with all required fields: `epic`, `name`, `status`, `priority`, `effort`, `owner`, `depends-on`, `tags`
- [ ] `epic` matches the actual parent folder name (or `_standalone`)
- [ ] `priority` is one of P0, P1, P2
- [ ] `effort` uses a valid format (e.g., `0.5d`, `3d`, `2w`)
- [ ] Problem statement is specific (references actual files, errors, or user impact — not vague)
- [ ] Proposed solution describes what will change
- [ ] Scope has both in-scope and out-of-scope items
- [ ] **Validation:** Acceptance criteria present — observable, verifiable outcomes (not just "it works")
- [ ] **RAID — Risks:** Risks & open questions section is present and non-empty (unless the plan is genuinely trivial). Each risk should have a likelihood assessment (low/medium/high) and a mitigation strategy or fallback.
- [ ] **RAID — Assumptions:** Assumptions are explicitly stated — things the plan takes for granted that, if wrong, would change the approach (e.g., "the existing service handles concurrency," "this endpoint is already behind auth middleware," "the data model won't change before implementation"). Plans with zero stated assumptions should be flagged — every non-trivial plan has assumptions.
- [ ] **RAID — Issues:** Known problems or blockers that must be resolved before or during implementation are called out with owners or resolution criteria (e.g., "need DBA sign-off on schema change," "upstream API contract not yet finalized"). If there are genuinely no issues, an explicit "None" is acceptable.
- [ ] **RAID — Dependencies:** Dependencies (`depends-on` in frontmatter) are listed — verify referenced plans exist in `backlog/`. External dependencies (APIs, services, teams) should also be listed.

**design.md:**
- [ ] Reuse assessment is present (checked existing services/repos/adapters before proposing new ones)
- [ ] Files to create/modify are listed with specific paths
- [ ] New files justify why existing modules can't be extended
- [ ] Database section is present (either migration details or "None")
- [ ] If migrations: rollback/downgrade strategy is documented
- [ ] If API changes: endpoints listed with method, path, schemas
- [ ] Testing strategy specifies test file paths and what to test
- [ ] Security section addresses relevant concerns (not just "N/A")
- [ ] Maintenance Implications section is present if the plan adds runtime behaviour (background jobs, data-producing components, new external dependencies). If omitted, verify the plan genuinely adds no ongoing operational burden.
- [ ] Dependencies section is present
- [ ] **UI Verification section** is present and substantive (not placeholder) if `design.md` lists changes under `<frontend.dirs>` (or any `*.tsx` / `*.css` if frontend is declared). The section must include: design intent path (pointing to a real file), routes touched with status (new vs. safety-net vs. has-granular), eight-state coverage plan, motion/a11y notes.

**tasks.md:**
- [ ] All tasks are numbered sequentially
- [ ] All tasks use checkbox format (`- [ ] N. description`)
- [ ] Tasks are specific enough to act on (file paths, function names — not "implement the feature")
- [ ] Tasks are in logical dependency order (e.g., "create model" before "create repo", "create service" before "write tests for service", "create migration" before "run migration")
- [ ] Testing tasks are present and separated by layer (unit, integration, API/contract, migration — as applicable)
- [ ] Post-implementation tasks include doc updates and status tracking
- [ ] **Verification tasks present:** A "Verification" section exists between Testing and Post-implementation with concrete gates: lint pass (`ruff check`), import smoke test, app boot test (if API changes), migration verification (if DB changes), frontend build (if frontend changes), feature test suite, full test suite with pre-existing failure exclusions, and acceptance criteria cross-check against proposal.md Validation section.
- [ ] **UI Verification task block** is present if UI changes are in scope (and `frontend` declared in conventions). Required tasks: mockup placement, fixture stubs, granular `<frontend.test_framework>` spec covering at minimum default/empty/error/long-content, safety-net removal (paired with granular spec creation when touched routes are in `<frontend.visual_safety_net>`), local UI visual harness run, local UI a11y check, lint on touched files, MCP-browser comparison step, eight-state checklist walk-through.
- [ ] Post-deployment verification tasks are present if the plan adds new runtime behaviour, background jobs, or data-producing components (monitoring check after deploy, 2-week review of operational metrics, feature flag cleanup)

### Step 3 — Run accuracy checks

Verify claims in the plan against the actual codebase:

- [ ] **Referenced files exist:** For every file path mentioned in design.md "Files to modify", verify the file exists. Flag any that don't.
- [ ] **Referenced models/tables exist:** If the plan mentions existing models or DB tables, verify they exist under the project's models directory (e.g., `<package>/models/` or `<package>/db/models/`).
- [ ] **Referenced routes exist:** If modifying existing routes, verify they exist under `<package>/api/` (or wherever the project's API code lives).
- [ ] **Referenced services/repos exist:** Verify referenced services and repositories exist (only applicable if `backend.patterns.service_layer` / `repository_layer` declared).
- [ ] **Referenced symbols exist on their claimed host:** When the plan names a specific method, function, attribute, constant, or enum value as living on a referenced class, module, or file, open that file and verify the symbol actually exists there. Do **not** stop at "the file exists and the class name is correct" — the class's public surface might not match the plan's claims. Apply this to `tasks.md` task descriptions, `design.md` "Files to modify" notes, and `proposal.md` Problem/Solution text — any backtick-quoted symbol-level claim is verifiable. When the real surface differs, realign the task/design/proposal text to the actual symbols and note where the originally-claimed ones actually live (a different module, the route layer, a repo method, or nowhere).
- [ ] **Dependencies available:** Check that referenced packages are declared in the project's manifest (`pyproject.toml`, `package.json`, `go.mod`, etc.) or are standard library.
- [ ] **Migration chain correct:** If migrations are planned AND `backend.migrations` is declared, read the latest file in `<backend.migrations.dir>` and verify the plan's `down_revision` (or equivalent) matches. Verify the planned migration number is the next sequential one.
- [ ] **No contradictions:** Check `<backlog_root>/` epic READMEs and existing feature proposals for plans that touch the same files or models. Also check `<docs.module_plans>` for legacy plan conflicts (if declared). Flag potential conflicts.
- [ ] **Event types:** If new event types are added AND `backend.patterns.event_bus` is declared, verify they don't duplicate existing ones in `<backend.events_module>`.
- [ ] **Generated/derived files accounted for:** If any task produces a file that a later task or CI step depends on (e.g., `npm install` → `package-lock.json` needed by `npm ci`; `pip compile` → `requirements.txt`; code generation → output files), verify the generated file is listed in "Files to create" and there is a task to commit it. Flag any implicit dependency between tasks where a generated artifact is consumed but never explicitly created or committed.
- [ ] **Internal consistency:** When the same command or tool invocation appears in multiple places within the plan (e.g., npm scripts, pre-commit hooks, CI jobs), verify flags and arguments are consistent. Flag discrepancies where one invocation uses stricter settings than another (e.g., `--max-warnings 0` in CI but not in the local lint script).
- [ ] **Design intent file exists:** If the design.md UI Verification section names a mockup or reference path (e.g., `docs/design/mockups/<feature>/intent.png` or `docs/design/references/<file>.png`), verify the file actually exists. A path to a non-existent mockup is the same kind of error as a non-existent migration parent — flag it.

### Step 4 — Run alignment checks

Verify the plan follows project conventions:

**Product alignment** (gate on this first — it's the most fundamental check):
- [ ] **Product alignment subsection present:** `proposal.md`'s Problem section has a "Product alignment" subsection (or equivalent) addressing which product positioning the feature serves. If the repo has product docs at `docs/product/{vision,strategy,products}.md`, the subsection cites the relevant focus area / territory / product. If the repo uses different product framing, the subsection cites that instead.
- [ ] **Alignment is substantive, not boilerplate:** the subsection actually names what the feature contributes (e.g., "implements part of the Intelligence Layer product per `docs/product/products.md` §2"), not just generic statements ("aligns with product strategy").
- [ ] **Compromises flagged explicitly:** if this feature trades off against a product commitment, the trade-off is named in the subsection — not silent. (Silent compromise is worse than open compromise; flag at review.)
- [ ] **Honesty test:** does the feature, as scoped in `proposal.md`, make the product story more or less honest about what the platform actually does? Flag if "less."

**ADR conformance** (architectural constraints that bind the design):
- [ ] **ADR conformance section present:** `design.md` has an "ADR conformance" section (or equivalent) addressing which platform ADRs (typically in `docs/architecture/NNN-*.md` or the repo's equivalent) bind this feature. If the repo has no platform ADRs, this check is skipped — note it explicitly.
- [ ] **Each relevant ADR is addressed:** for every ADR named, the section states one of: (a) satisfies the ADR's constraints (with how), (b) introduces a deviation (with severity + disposition + recommended DEV-NN entry in the ADR's table), (c) interacts with an open investigation flagged in the ADR.
- [ ] **No silent ADR violations:** scan the design's Architecture, Files-to-create, and Contract sections for patterns that would violate ADR constraints not addressed in the ADR-conformance section. Flag — silent violations are the failure this check exists to prevent.
- [ ] **Deviation entries are flagged for ADR update:** if the feature introduces a deviation, tasks.md includes a post-implementation task to add the DEV-NN entry to the ADR's deviation table.

**Contract-first** (the interface is authored before the implementation):
- [ ] **Contract section present in design.md:** design.md has a "Contract" section (or equivalent) naming the typed contract this feature introduces or modifies (HTTP route schemas, MCP tool schemas, Pydantic types at module boundaries, per-sidecar contract for external integrations). If the feature touches no cross-component contract boundary, the section says so explicitly.
- [ ] **Contract is concrete, not handwaved:** the section names actual schema modules / type definitions, not just "we'll define an interface." Future-author should be able to start writing implementation without contract design questions.
- [ ] **Contract review precedes implementation tasks:** the contract is defined in design.md (under review here) BEFORE tasks.md sequences the implementation. If contract is missing or stub-only, reject — implementation tasks can't be reviewed against a contract that doesn't exist.

**Test-first** (tests are authored before the implementation that satisfies them):
- [ ] **Test tasks reference specific scenarios from design.md:** tasks.md Testing section is not a generic "write tests" — each test task names what behavioural scenario or contract aspect it covers.
- [ ] **Tests precede implementation per unit:** the discipline is per-unit-of-work: for each implementation task, the corresponding test exists and is authored first. tasks.md may group tests in a Testing section for readability, but the chronological discipline (test-then-code per unit) is preserved at execution time. Flag if the task structure implies tests are written as a batch after implementation lands.
- [ ] **No tests-against-implementation:** tests verify the contract (per the Contract section above), not the implementation's internal structure. Flag tests that would have to be rewritten if the implementation refactored to a different shape that satisfies the same contract.

**Architecture & patterns** (each check fires only if its pattern is declared in `.claude/repo-conventions.yaml` and not `false`):
- [ ] **Repository pattern:** Data access goes through `<backend.patterns.repository_layer>/<entity>_repo.py`, not direct DB queries in routes or services *(skip if not declared)*
- [ ] **Service layer:** Business logic in `<backend.patterns.service_layer>/<entity>_service.py`, not in route handlers *(skip if not declared)*
- [ ] **Settings singleton:** Uses the project's settings singleton from `<backend.config_module>`, not direct env var reads
- [ ] **Adapter pattern:** External service calls go through `<backend.patterns.adapter_pattern>/`, not inline HTTP calls *(skip if not declared)*
- [ ] **Event bus:** New event types added to `<backend.events_module>`, published via the event bus of kind `<backend.patterns.event_bus>` *(skip if not declared)*
- [ ] **Route registration:** New routes registered in `<backend.api_main>`
- [ ] **Naming conventions:** Repo files named `<entity>_repo.py`, services `<entity>_service.py`, tests `test_<module>.py` *(repo/service naming applies only if those layers declared)*
- [ ] **Migration conventions:** Per the project's migration tool (`<backend.migrations.tool>`) — for Alembic: UUID PKs, timestamps, upgrade + downgrade, sequential numbering. *(skip section entirely if no migrations tool declared)*
- [ ] **No new patterns:** The plan does not introduce a new architectural pattern where an established one already exists (e.g., a new way to access settings, a new way to publish events)

**Code reuse & dead code:**
- [ ] **No unnecessary new files:** Every proposed new file justifies why an existing module can't be extended instead. Flag any new service/repo that overlaps with an existing one.
- [ ] **Dead code planned for removal:** If the plan deprecates or replaces existing functionality, tasks.md includes explicit tasks to remove the old code (old routes, old service methods, old imports)
- [ ] **No orphaned code:** If modifying a shared function's signature or removing a method, all callers are identified and updated in the plan

**Backward compatibility:**
- [ ] **API contract stability:** If changing API response schemas, the plan addresses whether existing consumers (frontend, PWA, external integrations) will break. If breaking, a migration/versioning strategy is documented.
- [ ] **Event schema evolution:** If changing event bus schemas, the plan addresses how subscribers handle both old and new formats during rollout (or documents that no subscribers exist yet).
- [ ] **Database migration safety:** If changing database schemas used by multiple running instances, the plan orders migrations to avoid breaking currently deployed code (e.g., add-before-remove, nullable columns first).

**Maintenance & lifecycle:**
- [ ] **Maintenance Implications assessed:** If the plan adds background jobs, data-producing components, new external dependencies, or deprecates functionality, the Maintenance Implications section in design.md is present and substantive (not placeholder).
- [ ] **Data growth addressed:** If the plan creates tables that grow unboundedly (logs, traces, events, audit records), a retention or purge strategy is specified — not deferred indefinitely.
- [ ] **Deprecation timeline:** If the plan deprecates existing functionality, a removal timeline is stated (not just "will be removed later").
- [ ] **Post-deployment follow-through:** If Maintenance Implications identifies operational concerns, there are corresponding post-deployment verification tasks in tasks.md.
- [ ] **Feature flag / shim cleanup:** If the plan adds temporary compatibility shims or feature flags for rollout, there is an explicit task to remove them after the rollout is confirmed stable.

**Testing quality:**
- [ ] **All test layers covered:** Plan includes the appropriate layers for the scope of changes:
  - Unit tests (always required)
  - Integration tests (required if multiple components interact)
  - API/contract tests (required if API endpoints change)
  - Migration tests (required if DB schema changes)
- [ ] **Test structure:** Unit tests in `<test_dir>/unit/`, integration in `<test_dir>/integration/` (per `backend.test_layers` in conventions)
- [ ] **Error paths tested:** Testing strategy includes error/failure scenarios, not just happy paths
- [ ] **Edge cases specified:** Boundary conditions are identified (empty collections, null values, max lengths, permission denied)
- [ ] **Test coverage proportional:** The number and detail of test tasks is proportional to the feature's complexity — a 5-day feature should not have one generic "write tests" task
- [ ] **Test level appropriateness:** Each planned test is assigned to the correct level for what it verifies. Flag: pure function tests planned as integration tests (should be unit — faster, no DB needed), multi-component scenarios planned as unit tests with heavy mocking (should be integration — mocking internals creates brittle tests that pass when the code is wrong), HTTP contract assertions planned as unit tests (should be API/contract tests).
- [ ] **No redundant coverage:** Planned tests don't duplicate coverage already provided by another test layer. If a unit test fully covers a logic path, an integration test for the same path should add value (e.g., verifying DB interaction, transaction behavior, or cross-service orchestration) — not just repeat the same assertion with a real DB instead of a mock. Flag plans where the same scenario appears in both unit and integration test sections with no additional coverage justification.

**Verification gates:**
- [ ] **Verification section exists:** A "Verification" section is present in tasks.md between Testing and Post-implementation
- [ ] **Lint check specific:** Lint task references the actual modules changed by this feature (not a generic `<package>/`)
- [ ] **Import smoke test specific:** Import task references the primary module/class introduced or modified by this feature
- [ ] **App boot test present (if API changes):** If design.md lists new or modified routes/middleware under `<backend.api_main>` (or its package), there is an app-boot smoke test (e.g., `python -c "from <api_module> import app"` for FastAPI)
- [ ] **Migration verification present (if DB changes):** If the plan includes migrations AND `backend.migrations` is declared, there is an explicit migration-apply verification task (e.g., `alembic upgrade head`)
- [ ] **Frontend build present (if frontend changes):** If design.md lists files under `<frontend.dirs>`, there is a `<frontend.build_command>` verification task
- [ ] **Feature test run specific:** Test run task references the actual test files for this feature, not just "run tests"
- [ ] **Pre-existing failures listed:** Full test suite task explicitly names the known pre-existing test failures to skip (from `<testing.pre_existing_failures>`) so autonomous sessions don't waste time fixing them
- [ ] **Acceptance criteria gate present:** Final verification task cross-references the proposal.md Validation section acceptance criteria

**UI conventions** (apply only if `frontend` declared in conventions AND `design.md` lists frontend file changes):
- [ ] **Visual test path convention:** Granular spec uses kebab-case matching the feature folder name, in the project's visual-tests directory (e.g., `<frontend.dirs>/tests/visual/<feature>.spec.ts` — adapt to actual layout)
- [ ] **Eight-state minimum:** Granular spec covers at minimum default, empty, error, and long-content states (per `<docs.eight_state>` if declared)
- [ ] **Safety-net pairing:** If touched routes are listed in design.md as currently in `<frontend.visual_safety_net>` (if declared), the tasks.md UI Verification block includes the safety-net-removal task paired with the granular-spec creation task
- [ ] **No new design primitives:** No new motion library, no new font, no new top-level design token introduced — must reuse existing per `<docs.design_principles>` if declared. Flag any addition without explicit justification in design.md.
- [ ] **Token discipline:** If `frontend.design_tokens: true`, no hex codes, no `px` values outside the token system, no inline `style={...}` with magic numbers in the proposed implementation
- [ ] **MCP comparison step:** tasks.md UI Verification block includes the MCP-browser comparison task

**Security:**
- [ ] **Security addressed:** Ownership validation, input sanitisation, ILIKE escaping (if search), sensitive field redaction (if PII)

**General best practices:**
- [ ] **Error handling:** Uses structured errors, Slack alerting for critical failures
- [ ] **Observability:** Structured logging (structlog), Prometheus metrics where applicable
- [ ] **Idempotency:** Operations that can be retried (Temporal activities, event handlers) are designed to be idempotent
- [ ] **Single responsibility:** Proposed functions/classes each have a clear, single purpose — no "god methods" that do everything

### Step 5 — Holistic assessment

Step back from the individual checks and assess the plan as a whole. These checks catch issues that fall between the cracks of specific checks.

**Cross-file consistency:**
- [ ] **Proposal ↔ Tasks:** Every promise in the proposal's "Proposed Solution" is delivered by at least one task in tasks.md. No orphaned tasks that deliver something the proposal never mentioned.
- [ ] **Design ↔ Tasks:** Every file in design.md's "Files to create" and "Files to modify" has a corresponding implementation task. Every migration has create, verify downgrade, and run tasks. Every new API endpoint has a corresponding task and an API/contract test task.
- [ ] **RAID ↔ Tasks:** If Issues list blockers with resolution criteria, there are pre-implementation tasks to resolve them. If Risks have mitigations requiring implementation work, those mitigations appear as tasks.
- [ ] **Effort ↔ Task count:** Rough sanity check — ~3-5 tasks per day of estimated effort is typical. Flag obvious mismatches (e.g., 12 tasks with a 1-day estimate, or 3 tasks with a 2-week estimate).

**End-to-end coherence:**
- [ ] **Solves the stated problem:** Trace from the problem statement through the design to the tasks — does completing all tasks actually solve the problem described in the proposal? Flag if the tasks deliver something subtly different from what the proposal promises.
- [ ] **No broken intermediate states:** Review the task order for gaps where the system would be in a broken state between tasks (e.g., a migration runs before the code that handles the new schema is deployed, a route is registered before its dependencies are created). Every task boundary should leave the system functional.
- [ ] **Maintenance Implications ↔ Tasks:** If Maintenance Implications identifies data growth, new background jobs, deprecation, or feature flags, verify corresponding tasks exist (retention logic, monitoring, removal timeline, cleanup task). If the section is omitted, verify no ongoing burden is introduced.
- [ ] **Validation ↔ Verification:** Every acceptance criterion in proposal.md's Validation section should be verifiable — either by a specific test in the Testing section, or by the acceptance criteria gate in the Verification section. If a criterion is not testable (e.g., "deployment is stable for 24h"), it should appear in Post-deployment verification instead. No criterion should be orphaned.
- [ ] **Nothing feels off:** After reviewing all artifacts, flag anything that seems underspecified, overly optimistic, or hand-wavy — even if it passed all specific checks. Trust your judgment here. Examples: a 1-day estimate for a task that touches 8 files, a testing strategy that only covers happy paths despite complex error handling, a "simple" migration on a high-traffic table with no mention of locking, a feature adding 3 new cron jobs with no mention of monitoring. **UI specific:** a UI feature plan with no mockup reference, no eight-state plan, or no granular Playwright spec is almost always underspecified — flag for USER input.

**Plan splitting assessment:**
- [ ] **Is this one independent piece of work?** A feature should be a single self-contained deliverable that can be reviewed and merged on its own and delivers value without waiting on sibling features. Independence is the test, not size. Flag for splitting if ANY of the following are true:
  - The plan bundles multiple independent features that don't share implementation dependencies (e.g., "add search AND redesign the settings page")
  - Changes span unrelated subsystems with no shared code paths (e.g., modifying both the email poller and the PWA frontend in one plan)
  - Incremental delivery is possible — some tasks deliver user-facing value on their own and don't need to wait for the rest
  - The migration count is high (3+) and spans multiple domain models
  - **Soft signal, not a gate:** task count climbs past the repo's `backlog.feature_size.goal_max_tasks` goal (default ~10). Don't flag on size alone — use it as a prompt to re-check independence. A coherent feature slightly over the goal is fine; a small one bundling unrelated changes is not.
  - If splitting is recommended, suggest specific split boundaries (e.g., "Split into Plan A: model + migration + repo, Plan B: API endpoints + frontend integration")

### Step 6 — Produce the report

Output the review in this format:

```markdown
## Feature Review: <feature-name>

**Path:** <path-to-plan-folder>
**Verdict:** PASS | NEEDS_WORK | FAIL

---

### Completeness

#### proposal.md
- PASS/FAIL: <item> — <detail if failing>

#### design.md
- PASS/FAIL: <item> — <detail if failing>

#### tasks.md
- PASS/FAIL: <item> — <detail if failing>

### Accuracy
- PASS/FAIL: <item> — <detail if failing, including the specific file/model/route that doesn't exist>

### Alignment
- PASS/FAIL: <item> — <detail if failing, with the correct convention to follow>

### Holistic Assessment

#### Cross-file consistency
- PASS/FAIL: Proposal ↔ Tasks — <detail if failing>
- PASS/FAIL: Design ↔ Tasks — <detail if failing>
- PASS/FAIL: RAID ↔ Tasks — <detail if failing>
- PASS/FAIL: Effort ↔ Task count — <detail if failing>

#### Backward compatibility
- PASS/FAIL: API contract stability — <detail if failing>
- PASS/FAIL: Event schema evolution — <detail if failing>
- PASS/FAIL: Database migration safety — <detail if failing>

#### Maintenance & lifecycle
- PASS/FAIL: Maintenance Implications assessed — <detail if failing>
- PASS/FAIL: Data growth addressed — <detail if failing>
- PASS/FAIL: Deprecation timeline — <detail if failing>
- PASS/FAIL: Post-deployment follow-through — <detail if failing>
- PASS/FAIL: Feature flag / shim cleanup — <detail if failing>

#### Verification gates
- PASS/FAIL: Verification section exists — <detail if failing>
- PASS/FAIL: Scope-appropriate checks — <detail if missing app boot/frontend build/migration tasks that the plan scope requires>
- PASS/FAIL: Pre-existing failures listed — <detail if failing>
- PASS/FAIL: Acceptance criteria gate — <detail if failing>

#### End-to-end coherence
- PASS/FAIL: Solves the stated problem — <detail if failing>
- PASS/FAIL: No broken intermediate states — <detail if failing>
- PASS/FAIL: Validation ↔ Verification — <detail if acceptance criteria are orphaned>
- PASS/FAIL: Maintenance Implications ↔ Tasks — <detail if failing>
- PASS/FAIL: Nothing feels off — <detail or "No concerns">
- PASS/FAIL: Scope is appropriately sized — <if FAIL: recommended split boundaries>

#### UI verification (omit if no UI changes in scope)
- PASS/FAIL: design.md UI Verification section present and substantive — <detail>
- PASS/FAIL: tasks.md UI Verification block present with required tasks — <detail>
- PASS/FAIL: Design intent file exists at named path — <detail>
- PASS/FAIL: Visual test path matches `frontend/tests/visual/<feature>.spec.ts` — <detail>
- PASS/FAIL: Eight-state minimum (default/empty/error/long-content) covered — <detail>
- PASS/FAIL: Safety-net removal paired with granular spec creation — <detail>
- PASS/FAIL: No new design primitives without justification — <detail>

---

### Summary

**Passing checks:** N/M
**Failing checks:** N/M

### Action Items (if NEEDS_WORK or FAIL)
1. <Specific action — what to fix, where>
2. <Next action>
```

### Step 7 — Mark as ready (PASS only)

If the final verdict is **PASS**, rename the feature folder by prepending `_ready_` to the folder name:

```bash
mv backlog/<epic>/features/<feature-name> backlog/<epic>/features/_ready_<feature-name>
```

This makes review-ready features immediately visible in the file tree. Skip this step if the verdict is NEEDS_WORK or FAIL.

---

## Verdict criteria

- **PASS:** All checks pass, or only minor informational notes (no action required)
- **NEEDS_WORK:** Any accuracy check fails (references non-existent files, wrong migration chain) OR any completeness check fails (missing section, placeholder-only content, missing test layer, missing RAID items) OR any alignment check has a clear violation OR holistic assessment flags scope splitting or underspecified areas
- **FAIL:** Required files are missing (no proposal.md / design.md / tasks.md), plan contradicts established architecture patterns, the proposed design would create fundamental problems (circular dependencies, data integrity issues, security holes), OR holistic assessment reveals the plan doesn't actually solve the stated problem

---

## Batch Mode

Review N plans sequentially. Each plan runs its own review→fix loop until convergence. Fully autonomous — no stops between plans.

### Step 1 — Parse and validate input

Extract all plan folder paths from the user's input. For each path:
- Verify the folder exists
- Verify it contains `proposal.md`, `design.md`, and `tasks.md`

If any path is invalid or missing required files, report it immediately and exclude it from processing. Do not stop — continue with the remaining valid paths.

Print a brief processing manifest before starting:

```
## Starting Batch Review

Plans to process (N):
1. <path1>
2. <path2>
...

Beginning plan 1 of N...
```

---

### Step 2 — Process each plan sequentially

For each valid plan path, launch a **sub-agent** using the Agent tool (`subagent_type: "general-purpose"`). Wait for the sub-agent to complete before launching the next one.

Pass the following prompt to the sub-agent:

---

**Sub-agent prompt:**

```
You are reviewing and iteratively fixing the implementation plan at: <PATH>

Your job is to run a review→fix loop until one of three exit conditions is met:
  - CONVERGED: the plan passes the full review checklist twice in a row (zero failures)
  - USER_BLOCKED: the only remaining failures require human input, confirmed twice in a row (ensures auto-fixes didn't introduce new AUTO items)
  - CAP_REACHED: you have completed 8 iterations without converging

## Setup

Before starting, read the full review checklist from the Single Plan Review section:
  .claude/skills/feature-review/SKILL.md

This gives you Steps 1–6 (read plan, completeness, accuracy, alignment, holistic, report format). Follow all checks exactly.

Also read all three plan files:
  <PATH>/proposal.md
  <PATH>/design.md
  <PATH>/tasks.md

Track these counters:
  iteration_count = 0
  consecutive_clean_count = 0   # "clean" = PASS or only USER items remaining

---

## Loop

Repeat the following until an exit condition is met:

### A. Review

Run all checks from Steps 2–5 of the Single Plan Review section against the current state of the plan files. Read actual codebase files to verify accuracy checks — do not guess. Produce a full review report in the Step 6 format.

### B. Evaluate verdict

**If PASS (zero failures):**
  - Increment consecutive_clean_count
  - If consecutive_clean_count >= 2 → **IMMEDIATELY rename the folder** (see below), then EXIT: CONVERGED
  - Otherwise → go to A (re-review to confirm)

  **Rename on CONVERGED (mandatory):** Before returning, run this bash command:
  ```bash
  mv <PATH> <PARENT>/_ready_<folder-name>
  ```
  For example: `backlog/tech_debt/auth-rbac-hardening` → `backlog/tech_debt/_ready_auth-rbac-hardening`
  This is NOT optional. If the rename fails, report the error. Do NOT skip this step.

**If NEEDS_WORK or FAIL:**
  - Classify each failing check as AUTO or USER:

    AUTO — fix using information available in the codebase or derivable from
    project conventions. Examples:
      - Wrong or missing frontmatter field that can be derived (e.g. `epic`
        from folder name, `status: not-started`, `owner: AJ`)
      - File paths that don't exist but have a clear correct path in the codebase
      - Wrong `down_revision` — look up the actual latest migration and fix it
      - Wrong migration number — find the next sequential number
      - Missing section that can be populated from codebase analysis
        (e.g. missing test file paths, missing route registration note)
      - Formatting issues (checkbox format, task numbering, heading structure)
      - Missing Verification section — generate from plan's affected modules/test files
      - Generic verification tasks (e.g. `<ci.lint> <package>/`) — make specific from design.md file list
      - Missing app boot/frontend build/migration verification task when design.md scope requires it
      - Missing acceptance criteria gate — add it referencing proposal.md Validation section
      - Missing pre-existing failure list in full test suite task — copy from `<testing.pre_existing_failures>` in `.claude/repo-conventions.yaml`
      - Tasks out of dependency order — reorder them
      - Effort estimate clearly mismatched with task count — adjust estimate
      - Missing "None" placeholders for sections with no applicable content
      - Referenced service/repo/model that has a different name in the actual
        codebase — find the correct name and fix the reference
      - Referenced method/function/attribute doesn't exist on the claimed class
        or module, but the real public surface is clearly the intended target —
        open the file, find the actual symbols, and realign the task text (plus
        any matching notes in design.md and proposal.md) to them. Note where the
        originally-claimed symbol actually lives (different service, route layer,
        repo method, or nowhere). Classify as USER instead if the surface
        mismatch suggests the plan targets the wrong class entirely rather than
        just wrong method names on the right class.
      - Duplicate event type — find a non-conflicting name
      - Missing UI Verification section in design.md (when frontend files are touched)
        — auto-add the section as a structured placeholder for the user to fill
        in (do not fabricate a design intent path or eight-state plan)
      - Missing UI Verification task block in tasks.md (when frontend files are
        touched) — auto-add the canonical task list (mockup placement, fixture
        stubs, granular spec, safety-net removal, ui-visual, ui-a11y, eslint
        jsx-a11y, MCP comparison, eight-state walk-through) with the spec path
        derived from the feature folder name
      - Missing safety-net-removal task when the design.md UI Verification
        section names a route currently in `_safety-net.spec.ts` and a granular
        spec is planned — auto-add the paired removal task
      - Visual test path that doesn't match the project's visual-test convention
        (e.g., `<frontend.dirs>/tests/visual/<feature>.spec.ts`) — auto-rename to the convention

    USER — requires human judgment or domain knowledge not in the codebase.
    Examples:
      - Scope splitting decision (requires product/stakeholder input)
      - Vague problem statement (requires author to clarify intent)
      - Missing risks/assumptions where the plan is too new to assess
      - Design trade-off where multiple valid approaches exist and the plan
        has not committed to one
      - Missing business context that only the plan author knows
      - Security concern requiring threat model input
      - "Nothing feels off" flag where the holistic concern is subjective

  - If ALL remaining failures are USER items:
    - Increment consecutive_clean_count
    - If consecutive_clean_count >= 2 → EXIT: USER_BLOCKED
    - Otherwise → go to A (re-review to confirm no new AUTO items)
  - If any AUTO items exist:
    - Reset consecutive_clean_count = 0
    - Fix all AUTO items by editing the plan files. Make minimal targeted edits —
      do not rewrite sections that don't need changing.
  - Increment iteration_count. If iteration_count >= 8 → EXIT: CAP_REACHED
  - Go to A

---

## What to return

Return a structured result containing:

1. **Exit status:** CONVERGED | USER_BLOCKED | CAP_REACHED
2. **Final verdict:** PASS | NEEDS_WORK | FAIL
3. **Iterations used:** N
4. **Renamed:** yes/no — if CONVERGED, the old and new path (e.g., `backlog/tech_debt/foo` → `backlog/tech_debt/_ready_foo`). If the rename failed, report the error.
5. **Iteration log:** For each iteration, a one-line summary of what was fixed
   (or "No fixes — re-review for confirmation" on the second-pass confirmation)
6. **Remaining items (if USER_BLOCKED or CAP_REACHED):**
   - For each remaining failure: classification (USER/AUTO), check name,
     and a brief description of what needs to be done
7. **Final review report** in Step 6 format (the full report from the last iteration)

Example return format:

---
### Plan: <path>
**Exit status:** CONVERGED
**Final verdict:** PASS
**Iterations:** 3
**Renamed:** yes — `backlog/tech_debt/foo` → `backlog/tech_debt/_ready_foo`

**Iteration log:**
- Iteration 1: Fixed `down_revision` (was placeholder, set to actual hash),
  fixed task numbering (tasks 12-15 were non-sequential), added missing
  `owner: AJ` to frontmatter
- Iteration 2: Added missing test file paths in design.md testing strategy,
  reordered tasks (migration run task was before migration create task)
- Iteration 3: No fixes — re-review confirmed PASS

**Final review report:**
[full Step 6 report]
---
```

---

After the sub-agent completes, record its result (exit status, verdict, iterations, remaining items, whether folder was renamed).

Announce progress between plans:

```
✓ Plan 1 complete (CONVERGED, 3 iterations, renamed to _ready_<name>) — starting plan 2 of N...
```

---

### Step 3 — Print combined summary

After all plans have been processed, output:

```markdown
## Batch Feature Review — Complete

| # | Plan | Verdict | Status | Iterations | Remaining Items |
|---|------|---------|--------|------------|-----------------|
| 1 | <path> | PASS | CONVERGED | 3 | — |
| 2 | <path> | NEEDS_WORK | USER_BLOCKED | 5 | 2 items |
| 3 | <path> | NEEDS_WORK | CAP_REACHED | 8 | 4 items |

---

### Auto-fixes applied

#### <path1>
- Iteration 1: <summary>
- Iteration 2: <summary>
- Iteration 3: No fixes — confirmation pass

#### <path2>
- Iteration 1: <summary>
...

---

### Your input required

List all USER items from any plan that exited as USER_BLOCKED or CAP_REACHED.
Group by plan.

#### <path2> (USER_BLOCKED after 5 iterations)
1. **<check name>** — <what needs to be decided and where in the plan>
2. **<check name>** — <what needs to be decided>

#### <path3> (CAP_REACHED after 8 iterations)
1. **<check name> [USER]** — <description>
2. **<check name> [AUTO]** — <description — note these are AUTO items that ran
   out of iterations, not user-input items, but still need attention>

---

### Ready for implementation

List all plans that CONVERGED and were renamed:

#### Renamed
- `<old-path>` → `<new-path>` (renamed to `_ready_<feature-name>`)

---

### Next steps
1. Address the "Your input required" items above, then re-run
   `/feature-review` on the affected plans
2. Plans that CONVERGED have been renamed to `_ready_<feature-name>` and are
   ready for `/feature-implement`
```

---

## Constraints

- **Cap:** Max 8 iterations per plan. Stop and report remaining items.
- **Sequential:** Each plan runs in its own sub-agent. Do not launch plans in parallel — wait for each to finish before starting the next.
- **No plan cap:** There is no limit on how many plans can be passed. The outer loop only holds short summaries, not full review context.
- **Conservative auto-fixes:** Only fix things verifiable against the codebase. When in doubt, classify as USER.
- **Do not invoke `/feature-review` recursively:** Sub-agents read the checklist directly from the skill file and run checks inline.
- **Read before editing:** Sub-agents must read each plan file before editing it.
