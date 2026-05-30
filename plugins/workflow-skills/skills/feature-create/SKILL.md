---
name: feature-create
description: Create a new feature plan with consistent structure, project alignment, and best practices. Reads .claude/repo-conventions.yaml at Step 0 to parameterise paths and patterns to the active repo.
user_invocable: true
---

# /feature-create

Create a new feature plan inside the project's `backlog/` structure.

This skill is **repo-agnostic.** It reads `.claude/repo-conventions.yaml` at Step 0 (schema: `~/.claude/repo-conventions-schema.md`) and parameterises everything against that file. Examples in this skill use EmailPOC conventions for concreteness; substitute your repo's values from its conventions file.

## Usage

```
/feature-create <feature-name> [description or path to ideas/<topic>.md]
/feature-create <feature-name> --epic <epic-name>
```

If `--epic` is provided, creates in `<backlog_root>/<epic>/features/<feature-name>/`.
If omitted, creates in `<backlog_root>/_standalone/<feature-name>/`.

## What you do

### Step 0 — Read repo conventions

Read `.claude/repo-conventions.yaml` at the repo root. Set working values for:
- `package` — backend package name (e.g., `kristallklar`)
- `backend.test_dir`, `backend.api_main`, `backend.config_module`, `backend.events_module`
- `backend.migrations.tool`, `backend.migrations.dir` (if declared)
- `backend.patterns.{repository_layer, service_layer, adapter_pattern, event_bus, workflows}` — note which are declared/`false`
- `frontend.*` if present (omit frontend guidance entirely if absent)
- `docs.*` paths (skip references to docs that aren't declared)
- `ci.*` commands
- `testing.runner`, `testing.pre_existing_failures`
- `backlog_root` (default: `backlog`)

**If `.claude/repo-conventions.yaml` is missing:**
1. Auto-detect what you can: read `pyproject.toml` for the package name; look for `alembic.ini` or `manage.py` for migrations; look for `package.json` with React/Vue/Svelte for frontend.
2. Ask the user the minimum needed for this run (typically: backend package name).
3. At the end of the run, suggest authoring `.claude/repo-conventions.yaml` to avoid re-asking next time.

Throughout the steps below, `<package>`, `<test_dir>`, etc. refer to the values read in this step.

### Step 1 — Gather context

Read the following to understand the current state. Skip any path that isn't declared in conventions or doesn't exist.

1. `<backlog_root>/README.md` — backlog structure and conventions
2. If `--epic` provided, read `<backlog_root>/<epic>/README.md` — epic scope, phases, existing features
3. Scan `<backlog_root>/` for existing features that overlap — check epic READMEs and feature proposals
4. `<docs.technical_spec>` if declared — current architecture
5. `<docs.functional_spec>` if declared — current feature set
6. If `<backend.migrations.dir>` is declared: read the latest migration file there to find current highest migration number and `down_revision`
7. If an idea file was referenced by the user, read it from `<docs.ideas_root>/` (or wherever the user named)
8. `<docs.module_plans>` if declared — overlapping legacy plans (historical reference)

Flag any overlap or contradiction with existing plans before proceeding.

**UI design intent (mandatory if the feature touches UI AND `frontend` section is declared in conventions):**

If the user's brief mentions UI, screen, page, dialog, modal, table, form, button, or any visible behaviour, ask the user — before generating any files — for the design intent reference. Acceptable forms:

- A path to a mockup PNG that you will move into `docs/design/mockups/<feature-name>/intent.png` (or wherever the project's mockups live)
- A path to an existing screenshot in `docs/design/references/`
- "Match existing screen X" — name the route and explain which aspects to match (density, layout, tone)

If the user can't supply any of these, suggest they screenshot an existing screen of theirs that has the right feel. The point is concrete intent — "build a customer detail page" without a reference is underspecified, and you should say so explicitly.

Record the chosen reference path; you will surface it in the proposal's Scope section, the design.md UI Verification section, and the tasks.md UI Verification block.

### Step 1b — Detect inline plan from conversation

Before writing any files, check whether the current conversation already contains an agreed-upon plan for this feature. Look for:

- A plan mode output (the Claude Code plan file surfaced in this session)
- Back-and-forth discussion that ended with a clear problem, solution, and scope being agreed

**If found:** Set a flag: `inline_plan_detected = true`. Record the agreed problem statement, solution, and scope verbatim — you will use these in Step 3. Do NOT reinterpret, expand, or summarise them differently.

**If not found:** Proceed as before — all three files will be generated from scratch.

### Step 2 — Determine target location

- If `--epic <epic-name>` was provided → `<backlog_root>/<epic>/features/<feature-name>/`
- If no epic specified → `<backlog_root>/_standalone/<feature-name>/`

**Folder existence check:**

- If the target folder **does not exist** → proceed normally.
- If the target folder **exists and contains `proposal.md`** → **STOP.** The proposal is the source of truth and must not be overwritten. Tell the user:
  > "A `proposal.md` already exists at `<path>`. This skill writes all three files from scratch and would overwrite the converged proposal. Did you mean to run `/feature-elaborate <path>` instead? It authors `design.md` and `tasks.md` against the existing proposal without touching it."
  Wait for the user's call before doing anything. Do not proceed with `/feature-create` unless the user explicitly confirms they want to overwrite.
- If the target folder **exists but `proposal.md` is missing** (e.g., a partial/stale folder) → flag what's there, ask the user whether to delete the partial folder and start fresh, or move the partial files aside.

If creating under an epic, verify the epic folder exists. If it doesn't, ask the user whether to create the epic (suggest `/epic-create`) or use `_standalone`.

### Step 3 — Create the plan folder

**If `inline_plan_detected = true`:**

Write `proposal.md` by converting the inline plan to the proposal format:
- Copy the agreed problem, solution, and scope verbatim — do not add scope, change framing, or reinterpret.
- Fill in the RAID sections (Risks, Assumptions, Issues, Open Questions, Dependencies) by synthesising from what was discussed. If a section was not discussed, use reasonable defaults based on the design, but keep them brief.
- Fill in Validation acceptance criteria from any success conditions discussed.
- Fill in the YAML frontmatter (status, priority, effort, etc.) using information from the discussion or sensible defaults.
- Add a Decision Log row for any choices that were explicitly decided during the inline discussion.

Write `design.md` and `tasks.md` as the new artifacts, generated from the proposal content above. Follow the templates below as normal for these two files.

**If `inline_plan_detected = false`:**

Generate all three files from scratch using the templates below.

---

Create the target folder with three files:

#### `proposal.md`

```markdown
---
epic: <epic-name or _standalone>
name: <Human Readable Feature Name>
status: not-started
priority: <P0 | P1 | P2>
effort: <Xd>
start:
end:
depends-on: []
owner: <owner-from-existing-features-or-AJ>
tags: []
---

# <Feature Name> — Proposal

**Created:** <today's date>

---

## Problem

<What problem does this solve? Why does it matter? Be specific — reference actual files, error messages, or user impact.>

### Product alignment

<REQUIRED. Which product positioning does this feature serve? Reference the relevant product/strategy docs in this repo — e.g., `docs/product/{vision,strategy,products}.md` if present, or whatever product framing this repo uses. State which focus area / territory / product the feature contributes to. If this feature compromises a product commitment (a known trade-off against stated positioning), flag it explicitly here — never silent. The honesty test: does this feature make the product story more or less honest about what the platform actually does?>

## Proposed Solution

<High-level approach. What will change? What will be built?>

## Scope

### In scope
- <item>

### Out of scope
- <item>

## RAID

### Risks
- **Risk:** <description> — **Likelihood:** <low/medium/high> — **Mitigation:** <what to do if it happens>

### Assumptions
- <assumption — e.g., "the existing auth middleware already protects these routes">

### Issues
- <issue — e.g., "need DBA sign-off on schema change before migration task", or "None">

### Open Questions
- <open question>

### Dependencies
- <dependency, or "None">

## Validation

- [ ] <criterion 1>
- [ ] <criterion 2>

## Decision Log

| Decision | Choice | Rationale | Date |
|---|---|---|---|
| | | | |

## Related Plans
<List only plans with a direct dependency or benefit relationship — not all plans in the same domain. Or "None".>
```

#### `design.md`

The template below references conventions. **Substitute paths from the conventions file** when writing the actual design.md. Omit sections that don't apply (e.g., omit Database section if the repo has no migrations tool, omit UI Verification if no frontend section).

```markdown
# <Feature Name> — Technical Design

---

## Architecture

<How does this fit into the existing architecture? Which layers are affected?>

### Reuse assessment
- **Existing services/repos that cover related functionality:** <list, or "None — new domain">
- **Existing adapters that can be reused:** <list, or "None needed"> [omit bullet if `adapter_pattern: false`]
- **Existing utilities/helpers that apply:** <list, or "None">
- **Code to deprecate/remove:** <if this replaces existing functionality, list what becomes dead code and should be removed>

### Files to create
- <path> — <purpose> — <why an existing file can't be extended instead>

### Files to modify
- <path> — <what changes>

### Database changes (omit section if `backend.migrations` is not declared OR no DB changes)
- Migration number: <next after current latest, read from `<backend.migrations.dir>`>
- New tables/columns: <describe>
- `down_revision`: <current latest migration revision hash>
- Rollback strategy: <what `downgrade()` does>

### API changes (omit section if no API changes)
<New or modified endpoints. Include method, path, request/response schemas.>
<If new routes: note registration in `<backend.api_main>`>

### Event bus changes (omit section if `backend.patterns.event_bus: false` OR no new events)
<New event types to add to `<backend.events_module>`.>

## Contract

<Names the typed contract this feature introduces or modifies. The contract is the spec — implementation conforms to it; tests verify against it. Contract is authored HERE in design.md before implementation tasks are written in tasks.md.>

<Where each interface's contract lives (pick the relevant ones):
- HTTP routes: OpenAPI schema additions/changes in the API's schema module
- MCP tools: tool schemas in the relevant contract types module
- Module boundaries between packages: Pydantic / typed-dict types in the relevant contract module
- External integrations: per-sidecar / per-adapter contract owned by that integration's package>

<If this feature touches no cross-component contract boundary, say so explicitly. Reviewers reject implementations without a reviewed contract.>

### Contract decisions — ADR-007 (no implementation-time decisions)

Per ADR-007 ("No implementation-time decisions"), **no contract decision this feature depends on may be deferred to implementation time.** A contract decision is any choice of a type, schema, API shape, persistence layout, or test-infra dependency that another component (or this feature's own tests) will bind to. There is no "decide while building" — every such decision must be routed *here, in analysis,* to exactly one of:

- **Resolve** — settle it now, in this design. State the concrete type / schema / shape / layout. The importing code can be written against it directly.
- **Decouple** — the decision belongs to a sibling feature that *owns* that contract. Name the sibling and add it to the proposal's `depends-on`. Do **not** stub the sibling's contract here.
- **Spike** — the decision needs a time-boxed investigation before it can be resolved. Add an explicit spike task and gate the dependent work on its outcome.
- **Block** — the contract this feature needs does not exist yet and cannot be resolved, decoupled, or spiked now. Mark the proposal `status: blocked` and name the missing dependency. Do not scaffold around the gap.

Never land a thin/stub version of a sibling-owned contract in this feature's PR and "reconcile later" — that is the exact failure ADR-007 forbids. The fix for a deferred decision is always to **route** it (resolve / decouple / spike / block), never to reword the deferral.

For every dependency named in the **Dependencies** section below, record which of the four routes applies. A dependency-driven contract decision must never stay implicit.

## ADR conformance

<Which platform ADRs (typically in `docs/architecture/NNN-*.md` or this repo's equivalent) bind this feature? Name them and address conformance for each:
- Does this feature satisfy the ADR's constraints? (most common — state how)
- Does it introduce a deviation? Record as a DEV-NN entry in the ADR's deviation table, with severity and disposition.
- Does it interact with an open investigation flagged in an ADR (e.g., a "non-deferrable" DEV-NN row)?>

<If this feature touches no ADR-bound concern, say so explicitly. Reviewers reject features that silently violate ADR constraints.>

## Dependencies

For every dependency listed here, append its **ADR-007 route** in brackets so the contract decision it implies is never left implicit: `[resolve | decouple → <sibling feature> | spike → <task> | block → <missing dependency>]`. A dependency whose contract is not yet settled must be decoupled, spiked, or blocked — never carried as an implicit "we'll figure it out in code." (See the Contract section's ADR-007 note.)

- **Language packages:** <any new dependencies to add to pyproject.toml / package.json>
- **External services:** <any new service dependencies>
- **Internal dependencies:** <which existing modules/services this depends on — each tagged with its ADR-007 route, e.g. `auth.token contract [decouple → auth-rbac-hardening]`>

## Testing Strategy

Use a BDD-style approach: describe tests as behaviours ("Given X, When Y, Then Z") before specifying implementation.

### Behavioural scenarios (BDD)
- **Given** <precondition>, **When** <action>, **Then** <expected outcome>
- **Given** <error precondition>, **When** <action>, **Then** <expected error handling>

### Unit tests — `<test_dir>/unit/test_<module>.py` (omit section if no application code changes)
- Happy path: <list specific scenarios derived from behaviours above>
- Error paths: <invalid input, missing data, permission denied, service unavailable>
- Boundary conditions: <empty lists, max values, null values, concurrent access>

### Integration tests — `<test_dir>/integration/test_<module>.py` (omit if no multi-component interactions)
- <list what to test end-to-end within the backend>

### API / contract tests (omit if no API changes)
- <list endpoints to test>

### Migration tests (omit if no DB changes)
- <migration test scenarios>

### Known constraints
<Carry over from the conventions file's testing section, e.g. pre-existing failures to skip. Add any project-specific constraints discovered in Step 1.>

## UI Verification (omit entire section if `frontend` is not declared in conventions OR no UI changes)

<Required for any feature that touches frontend dirs declared in `frontend.dirs`. See `<docs.design_loop>` if declared for the full workflow.>

### Design intent
- **Reference path:** `docs/design/mockups/<feature-name>/intent.png` (or wherever the project's mockups live)
- **What to match:** <density / layout / tone / colour / all of these>
- **What is allowed to differ:** <e.g., "colour palette comes from existing tokens, not the mockup">

### Routes touched
- `<route path>` — current status: <new route | already in safety net | already has granular spec>

### Eight-state coverage plan (only include if `<docs.eight_state>` is declared)
<For each state in the project's eight-state checklist, note: reachable in running app via seed data, reachable via easy seed tweak, or requires API stub in the granular spec.>

| State | Reachable how |
|---|---|
| Empty | <e.g., "stub returns []"> |
| Loading | <e.g., "delayed stub response"> |
| Error | <e.g., "stub returns 500"> |
| Long content | <e.g., "stub returns 200-char title fixture"> |
| Mobile (320px) | <e.g., "real app, viewport resize in spec"> |
| Keyboard focus | <e.g., "real app, manual verification"> |
| Dark mode | <if applicable> |
| Disabled / read-only | <if applicable> |

### Motion and a11y notes (motion bullet only if `frontend.motion_library` is declared)
- **Motion:** <reuse existing config from `frontend.motion_library`, or note the specific deviation and why>
- **a11y:** <new ARIA attributes needed, focus management considerations, semantic HTML elements used>

## Security Considerations (omit if plan is purely tooling/config with no runtime impact)

- Input validation: <how inputs are validated>
- Ownership scoping: <how data access is restricted to the owning user/account>
- Search-query escaping: <if search queries are involved — e.g., ILIKE escaping for PostgreSQL>
- Sensitive field redaction: <if PII or credentials are handled>

## Observability

- Structured logging: <what gets logged, at what level>
- Metrics: <any metrics to add>
- Error alerting: <error/alert channels>

## Maintenance Implications (omit if plan adds no ongoing operational burden)

- **New background jobs / scheduled tasks:** <jobs added — include frequency and what happens if they fail>
- **Data growth:** <unbounded data created? retention/archival strategy?>
- **New external dependencies:** <new APIs, services, third-party integrations>
- **Deprecation impact:** <does this deprecate existing functionality?>
- **Upgrade sensitivity:** <tightly coupled to a specific library version or API contract?>
- **Operational surface:** <new things that can fail and page on-call?>
```

#### `tasks.md`

The template below references conventions. Substitute paths and commands from the conventions file. Omit sections that don't apply.

```markdown
# <Feature Name> — Tasks

**Status:** NOT_STARTED
**Last updated:** <today's date>

---

## Pre-implementation
- [ ] 1. Verify no conflicts with existing plans (checked during plan creation)
- [ ] 2. Review design with stakeholder (if applicable)

## Implementation
- [ ] 3. <First implementation task — be specific: file, function, what to do>
- [ ] 4. <Next task>
- [ ] 5. <Continue numbering sequentially>

## Database (include only if `backend.migrations` declared and migrations are needed)
- [ ] N. Create migration <NNN>_<description>.py in `<backend.migrations.dir>/`
- [ ] N+1. Verify `downgrade()` works cleanly
- [ ] N+2. Run migration on test environment

## Testing

**Test-first discipline: each test below is authored BEFORE the implementation task it verifies. The Testing section is grouped here for readability, not to imply chronological ordering — in execution, tests for unit X are written and run-red before the implementation of X. Reviewers reject implementations whose tests were authored after the code.**

- [ ] N. Write unit tests — happy path scenarios in `<test_dir>/unit/test_<feature>.py`
- [ ] N+1. Write unit tests — error paths and boundary conditions
- [ ] N+2. Write integration tests in `<test_dir>/integration/test_<feature>.py` (if multi-component)
- [ ] N+3. Write API/contract tests (if API changes)
- [ ] N+4. Write migration tests (if DB changes)

## UI Verification (omit entire section if `frontend` is not declared OR no UI changes)
- [ ] N. Author or move mockup to `docs/design/mockups/<feature-name>/intent.png`
- [ ] N+1. Write fixture stub for the default state in the project's visual-test fixtures location. **Do not add per-state fixtures preemptively.**
- [ ] N+2. Write granular `<frontend.test_framework>` spec covering **only the default state** (one snapshot per view as the starting baseline)
- [ ] N+3. If any touched route is currently in `<frontend.visual_safety_net>` (if declared), remove the safety-net entry — granular replaces, not supplements
- [ ] N+4. Run the project's UI visual harness locally (e.g., `make ui-visual` if defined) — takes screenshots for inspection vs. the mockup. Actual regression diff happens in `<frontend.visual_regression>` on PR.
- [ ] N+5. Run the project's UI a11y check locally (e.g., `make ui-a11y` if defined) — fix violations or open a remediation issue
- [ ] N+6. Run linting on touched frontend files (e.g., `npx eslint` with `jsx-a11y` for React)
- [ ] N+7. MCP-browser comparison: open the affected route, screenshot, compare to the mockup, iterate until they match
- [ ] N+8. Walk the project's eight-state checklist (if declared at `<docs.eight_state>`) **manually during development**

## Verification

These are the gates an autonomous implementation session uses to confirm the feature is clean before marking done. Each task should fail fast — run cheap checks first.

- [ ] N. Run linting on changed modules — `<ci.lint> <package>/<affected_paths>` passes clean
- [ ] N+1. Import smoke test — `python -c "from <package>.<primary_module> import <MainExport>"` succeeds (Python only)
- [ ] N+2. App boot smoke test — `python -c "from <backend.api_main module path> import app"` succeeds (include only if API routes or middleware changed)
- [ ] N+3. Verify migration applies cleanly — for `<backend.migrations.tool>`, e.g. `alembic upgrade head` succeeds (include only if DB changes)
- [ ] N+4. Frontend build — `<frontend.build_command>` completes without errors (include only if frontend files changed)
- [ ] N+5. Run feature test suite — `<testing.runner> <test_dir>/unit/test_<feature>.py <test_dir>/integration/test_<feature>.py -v` — all pass, no regressions in related modules
- [ ] N+6. Run full test suite — verify no regressions. Known pre-existing failures to SKIP (do not attempt to fix): list from `<testing.pre_existing_failures>` (or "None declared")
- [ ] N+7. Acceptance criteria — walk through each criterion in the proposal.md Validation section and confirm it is satisfied

## Post-implementation
- [ ] N. Update `<docs.functional_spec>` with new feature (only if user-facing functionality changed AND doc declared)
- [ ] N+1. Update `<docs.technical_spec>` if architecture changed (omit if doc not declared OR for tooling/config-only plans)
- [ ] N+2. Update proposal.md frontmatter: `status: done`, set `end:` date
- [ ] N+3. Update epic README.md status table (if feature is under an epic)
- [ ] N+4. **Smoke test check:** does this feature change a critical path (auth flow, health endpoint, high-traffic API route)? If yes, add or update an assertion in `<ci.smoke_test>` (if declared) and its test file. Otherwise note "no critical path change."

## Post-deployment verification (include if feature adds new runtime behaviour, background jobs, or data-producing components)
- [ ] N. After deployment to test/staging: verify metrics/logs are nominal for 24h
- [ ] N+1. After deployment to production: verify no error rate increase and new components are healthy
- [ ] N+2. After 2 weeks: review data growth and operational metrics — confirm assumptions in Maintenance Implications section hold
- [ ] N+3. Remove or resolve any temporary feature flags or compatibility shims added during rollout
```

### Step 4 — Evaluate scope and consider splitting

After drafting the plan content, assess whether this is **one independent piece of work** — a single self-contained deliverable that can be reviewed and merged on its own and delivers value without waiting on sibling features. Independence is the test, not size. Split if:

- The plan bundles independent features with no shared implementation dependencies
- Changes span unrelated subsystems with no shared code path
- Incremental delivery is possible — some tasks deliver value on their own without the rest
- 3+ migrations span different domain models

Task count is a **soft signal, not a gate**: if the plan climbs past the repo's `backlog.feature_size.goal_max_tasks` goal (default ~10), treat that as a prompt to re-ask the independence question — not an automatic split. A coherent feature slightly over the goal is fine; a small feature bundling two unrelated changes is not.

If splitting is warranted, go back to Step 2 and create separate plan folders instead. Note their dependency relationship in each proposal's Dependencies field.

### Step 5 — Apply project conventions and best practices

When generating the plan content, ensure the guidance below is applied. **Each "if pattern declared" guard checks the conventions file** — apply the rule only if the relevant pattern is declared and not `false`.

**Architecture patterns (apply only those declared in `backend.patterns`):**
- **Settings:** singleton from `<backend.config_module>`
- **Database:** repository pattern at `<backend.patterns.repository_layer>` with async sessions — *only if `repository_layer` is declared*
- **Services:** service layer at `<backend.patterns.service_layer>` — *only if declared*
- **External integrations:** adapter pattern at `<backend.patterns.adapter_pattern>` — *only if declared*
- **Events:** event bus of kind `<backend.patterns.event_bus>`, types in `<backend.events_module>` — *only if event_bus declared and not false*
- **Workflows:** `<backend.patterns.workflows>` workflows in `<backend.patterns.workflows_dir>` — *only if workflows declared and not false*

**Code reuse (always applies):**
- Search existing repos, services, and adapters for overlapping functionality
- Prefer extending an existing class/module over creating a parallel one
- If new functionality is similar to existing code, extract shared logic
- If this feature replaces existing functionality, explicitly plan removal of the old code in tasks.md

**Dead code prevention (always applies):**
- If deprecating a function/class/module, add a removal task to tasks.md
- If replacing an endpoint, add a task to remove the old route
- If superseding a service method, plan the migration of all callers before removal

**ADR-007 — no implementation-time decisions (always applies):**
- For **every** dependency the feature names (proposal `depends-on`, design.md Dependencies, any "this relies on X" claim), route the contract decision it implies to exactly one of **resolve / decouple / spike / block** — do not leave it for "implementation time."
- Do not scaffold a stub that stands in for a contract a sibling feature owns. If the contract isn't settled, decouple it to that sibling (and add it to `depends-on`), spike it, or mark the proposal `status: blocked` and name the missing dependency.
- Never emit deferral language — "decide at implementation time", "TBD in code", "figure out while building", "land a thin version and reconcile later" — for a type, schema, API shape, persistence layout, or test-infra dependency. The Contract section's ADR-007 note is the place to record the chosen route; mirror each dependency's route in the Dependencies section.

**API conventions (apply if API exists):**
- Routes registered in `<backend.api_main>`
- Request/response schemas (Pydantic for FastAPI, equivalent for other frameworks)
- Dependency injection for auth, DB sessions
- Ownership validation on all data-access endpoints

**Migration conventions (apply only if `backend.migrations` declared):**
- File naming: per the migration tool's convention (Alembic: `<NNN>_<description>.py` with sequential numbering)
- UUID primary keys (project convention — confirm in conventions if different)
- `created_at` / `updated_at` timestamp columns
- Both `upgrade()` and `downgrade()` implemented
- Correct `down_revision` chain

**Naming conventions (apply if `repository_layer` / `service_layer` declared):**
- Repository files: `<entity>_repo.py`
- Service files: `<entity>_service.py`
- Test files: `test_<module>.py`
- Kebab-case for plan folder names

**Task granularity:**
- Don't split trivially related edits to the same file into separate tasks
- Do split tasks that produce independently verifiable outcomes
- If a task involves `npm install`, `pip install`, or similar, include a task for committing any generated lockfiles

**Testing best practices:**
- Use BDD-style thinking: describe behaviours (Given/When/Then) before writing test code
- Cover all test layers: unit → integration → API/contract → migration (where applicable)
- Test error paths, not just happy paths
- Test boundary conditions
- Test the contract, not the implementation
- Keep test names descriptive: `test_<method>_<scenario>_<expected_result>`
- Separate test tasks by layer

**Security (address in every plan):**
- Input validation on all endpoints
- Ownership scoping (users only see their own data)
- Search-query escaping (e.g., ILIKE for PostgreSQL)
- Sensitive field redaction in logs/responses

**Maintenance & lifecycle:**
- Document new background jobs/scheduled tasks (frequency, failure mode, monitoring) in Maintenance Implications
- Add retention/purge strategy for unbounded-growth tables
- Set deprecation timelines explicitly
- Document graceful degradation for new external dependencies
- Plan post-deployment cleanup of feature flags and compatibility shims

**Backward compatibility:**
- API response schema changes: assess existing consumers (frontend, mobile, external integrations) for breakage
- Event bus schema changes: ensure subscribers handle both old and new formats during rollout
- DB schema changes used by multiple services: order migrations to avoid breaking running instances
- Public function/method removals: identify all external callers

**General coding best practices:**
- Single responsibility — each function/class does one thing
- Fail fast — validate at the boundary, not deep in the call stack
- Explicit over implicit — no magic strings, no implicit type conversions
- Idempotency for retried operations (event handlers, workflow activities, etc.)

**UI conventions (apply only if `frontend` declared in conventions):**
- Design intent reference is named in the proposal Scope and design.md UI Verification sections
- Eight-state coverage plan is filled in design.md (if `<docs.eight_state>` declared)
- Granular spec covers **only the default state** at first; additional states added incrementally as concrete concerns surface
- Safety-net removal task paired with granular spec creation when applicable
- No new motion library, font, or design token introduced — must reuse existing per `<docs.design_principles>` if declared
- Design tokens only (if `frontend.design_tokens: true`): no hex codes, no `px` outside the token system, no inline styles with magic numbers
- MCP-browser comparison step appears in tasks.md UI Verification block

### Step 6 — Self-validate coherence

Before finalising, cross-check the three files against each other and fix any inconsistencies:

**Proposal ↔ Tasks:**
- Every promise in the proposal's Proposed Solution is delivered by at least one task
- No orphaned tasks that deliver something the proposal never mentioned

**Design ↔ Tasks:**
- Every file in design.md's "Files to create / modify" has a corresponding implementation task
- Every migration in the design has create, verify-downgrade, and run tasks
- Every new API endpoint has a task and an API/contract test task

**RAID ↔ Tasks:**
- Issues with resolution criteria → pre-implementation tasks (or preconditions)
- Risks with implementation-work mitigations → mitigations appear as tasks

**Effort ↔ Task count:**
- ~3-5 tasks per day of estimated effort is typical. Flag obvious mismatches.

**Maintenance Implications ↔ Tasks:**
- Data growth → retention/purge task
- New background jobs → monitoring/health check task
- Deprecation → post-deployment removal task

**Design scope ↔ Verification:**
- API route changes → app boot smoke test
- Frontend file changes → frontend build (`<frontend.build_command>`)
- Migrations → migration apply check (e.g., `alembic upgrade head`)
- None → conditional tasks omitted (not left as placeholders)

**UI scope ↔ UI Verification (apply only if `frontend` declared):**
- Frontend file changes → both design.md UI Verification section AND tasks.md UI Verification block must be present and non-placeholder
- Touched route in safety net → safety-net-removal task paired with granular-spec creation task
- Granular-spec task covers default state only at first

**Dependencies ↔ ADR-007 routing:**
- Every dependency named (proposal `depends-on`, design.md Dependencies) carries an explicit ADR-007 route — resolve / decouple / spike / block. No unrouted dependency.
- No section of any of the three files contains an implementation-time deferral of a contract decision (type, schema, API shape, persistence layout, test-infra dependency). Scan for "decide at implementation time", "TBD in code", "figure out while building", "land a thin version and reconcile later", and any stub standing in for a sibling-owned contract. If found, route the decision — do not reword it.
- If a contract this feature needs genuinely does not exist yet, set the proposal `status: blocked` and name the dependency, rather than scaffolding around the gap.

**Validation ↔ Verification:**
- Every acceptance criterion in proposal.md is verifiable — covered by a test or by the acceptance criteria gate
- Runtime-observation criteria (e.g., "no error rate increase for 24h") belong in Post-deployment verification, not Verification

If any inconsistency is found, fix it before output. Do not leave known inconsistencies for `/feature-review` to catch.

### Step 7 — Update epic README (if under an epic)

If the feature was created under an epic, update the epic's `README.md` status table to include the new feature row.

### Step 8 — Final output

```
Created <backlog_root>/<epic-or-standalone>/<feature-name>/
  - proposal.md — converted from inline plan, or generated fresh (with YAML frontmatter)
  - design.md — architecture, files, migrations (if applicable), testing
  - tasks.md — <N> implementation tasks

Conventions applied: <package=X, patterns={...}, frontend=present/absent, etc.>
Epic: <epic-name or "_standalone">
Related plans: <list or "None">
Potential overlaps: <list or "None detected">

Next step: Run /feature-review to validate before implementation.
```

If `.claude/repo-conventions.yaml` was missing and you had to ask the user for values, suggest authoring the file at the end:

> "I had to ask for `<package>` (and N other values) because `.claude/repo-conventions.yaml` is missing. Want me to draft one based on what I learned this run? It'd save re-asking next time. Schema reference: `~/.claude/repo-conventions-schema.md`."
