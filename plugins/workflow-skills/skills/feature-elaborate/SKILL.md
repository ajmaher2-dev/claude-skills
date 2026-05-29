---
name: feature-elaborate
description: Author design.md and tasks.md for a feature folder that already has a converged proposal.md, without overwriting the proposal. Use when /feature-create's "all three from scratch" flow doesn't fit because the proposal already exists.
user_invocable: true
---

# /feature-elaborate

Author the missing `design.md` and `tasks.md` for a feature folder whose `proposal.md` already exists and is the source of truth.

This is the complement to `/feature-create`, which always writes all three files from scratch and refuses if the target folder is non-empty. `/feature-elaborate` is for the scenario where a proposal has been refined through one or more `/feature-review` iterations and you now need design + tasks authored against it without losing the converged content.

## Usage

```
/feature-elaborate <path-to-feature-folder>
/feature-elaborate <path-to-feature-folder> --force   # required to overwrite an existing design.md or tasks.md
```

Path is the feature folder, e.g. `backlog/<epic>/features/<feature-name>/` or `client-agent-platform/backlog/<epic>/features/<feature-name>/`.

## What you do

### Step 1 — Verify shape

1. Confirm the path exists and is a directory.
2. `proposal.md` MUST exist. If it doesn't, abort and recommend `/feature-create` instead.
3. `design.md` and `tasks.md` MUST NOT exist. If either does:
   - Without `--force`: abort with a clear message naming which file is in the way and what content it has (file size + first heading line). Do not overwrite.
   - With `--force`: warn loudly that you are about to overwrite, name the file(s), pause for explicit user confirmation in the chat before proceeding. `--force` is not a license to overwrite silently.
4. Confirm the folder layout fits the convention (`backlog/<epic>/features/<name>/` or `backlog/_standalone/<name>/`). If it doesn't, flag it but continue.

### Step 2 — Read the proposal carefully

Read `proposal.md` end-to-end. Record (in your working notes, not in any file):

- **Frontmatter:** `epic`, `name`, `priority`, `effort`, `depends-on`, `tags`. These constrain the design (e.g., effort estimate must be consistent with task count).
- **Problem and Proposed Solution:** verbatim. The design must deliver every promise in Proposed Solution and no more.
- **Scope (in / out):** the design's "Files to create / modify" must stay inside In Scope. Anything Out of Scope must NOT appear in design.md or tasks.md.
- **RAID:** Risks may need mitigation tasks; Issues may need pre-implementation tasks; Dependencies must match the frontmatter `depends-on`; Open Questions that are still open must be reflected as either decision-required tasks or surfaced to the user (Step 7).
- **Validation acceptance criteria:** every item must map to either a test in design.md Testing Strategy or a verification gate in tasks.md.
- **Decision Log:** decisions already made constrain the design. Do not re-litigate them; do not contradict them.
- **Resolved questions (struck-through with rationale):** treat the resolution as binding. Quote the resolution rationale in design.md when it materially shapes a design decision so the trace is clear.

### Step 3 — Read codebase context

Verify the proposal against reality before authoring. Don't guess; check.

1. **Epic README** (`backlog/<epic>/README.md`) — understand the epic's phases, related features, and status table. Note where this feature sits in the sequence and what it depends on.
2. **Sibling features** in the same epic — scan their `proposal.md` files for overlap, especially anything that names the same modules/files this proposal touches. Flag overlap to the user.
3. **Files mentioned in the proposal** — open every file the proposal names (modules, classes, configs, scripts) and verify:
   - The file exists at the claimed path.
   - Symbols named in backticks (`ClassName`, `function_name`, `CONST`) actually exist on that file's public surface.
   - Docstrings or comments in those files don't contradict the proposal. **This is the F3 case:** F1's `core/config/tenant_loader.py` docstring says F3 will deliver four schemas (`PersonaConfig`, `SpecialistsConfig`, `RoutingConfig`, `TeamsConfig`), but F3's proposal lists three YAMLs. That is a real proposal-level gap and must be surfaced (Step 7), not silently absorbed.
4. **Master plan / project roadmap** if the proposal references one — confirm decisions logged there are consistent with the proposal.
5. **Dependency features** named in `depends-on` — confirm they are merged or in flight, and identify the symbols/files this feature can rely on.
6. **Latest migration** in the migrations folder if the proposal implies DB work — record the latest revision hash for `down_revision` and the next sequential migration number.

Record discrepancies. They feed Step 7.

### Step 4 — Author design.md

Write `design.md` using **the same template as `/feature-create` Step 3 (the `design.md` template block in `~/.claude/skills/feature-create/SKILL.md`)**, parameterized by the proposal AND the active repo's `.claude/repo-conventions.yaml` (substitute paths and pattern conditionals as `/feature-create` does):

- **Architecture section** — describe how the feature fits into the existing layers, citing real modules from Step 3.
- **Reuse assessment** — list real existing services/repos/adapters/utilities that overlap. If the proposal's Proposed Solution names new files, justify why each can't be a modification of an existing file.
- **Files to create / Files to modify** — every file path must be real (for modifications) or land inside the directory structure the proposal sanctions (for creations). No invented paths.
- **Database / API / Event sections** — include only if the proposal's Proposed Solution actually requires them. Omit cleanly otherwise.
- **Testing Strategy** — derive BDD scenarios directly from the proposal's Validation acceptance criteria. Every criterion → at least one Given/When/Then. Then expand to unit / integration / API / migration layers as relevant.
- **UI Verification section** — include if and only if the proposal's Scope mentions UI. If included, the design intent reference must come from the proposal Scope; do not invent one.
- **Security, Observability, Maintenance Implications** — fill in based on what the proposal's Proposed Solution actually introduces. If the proposal explicitly defers an area (e.g., out-of-scope multi-tenant runtime selection), state the deferral in Maintenance Implications and link the deferral language back to the proposal Out of Scope item.
- **Dependencies section** — must match the proposal's frontmatter `depends-on` exactly.

Two hard rules for this step:

1. **Do not expand scope.** If the proposal says "three YAMLs," design.md ships three YAMLs. The fourth YAML in F1's docstring is a Step 7 question for the user — not a unilateral expansion. The design may grow only after the proposal is amended.
2. **Do not contradict resolved questions.** If the proposal's Open Questions section has a struck-through resolution, the design honours the resolution. Cite the resolution date and rationale in the relevant design section.

### Step 5 — Author tasks.md

Write `tasks.md` using **the same template as `/feature-create` Step 3 (the `tasks.md` template block)**, parameterized by the proposal and your design.md from Step 4:

- **Pre-implementation** — include a task for any Issue with a resolution criterion in the proposal RAID. If Open Questions remain unresolved (Step 7 would surface them), include a "decide question X with stakeholder" task.
- **Implementation** — one task per "File to create" / "File to modify" bullet in design.md, plus any cross-cutting wiring tasks (route registration, event publishing, settings additions).
- **Database / API / UI Verification / Post-deployment** — include each section conditionally on the design's scope, exactly as `/feature-create` does. The conditional rules from `/feature-create` Step 6 (self-validation) apply: if design has migrations, Verification includes `alembic upgrade head`; if design has frontend changes, Verification includes `npm run build`; etc.
- **Testing** — one task per test layer (unit happy / unit error / integration / API / migration), each spelling out the test file path.
- **Verification** — the standard gate block from `/feature-create` Step 3, parameterized to the changed modules.
- **Acceptance criteria task** — explicitly cross-checks every Validation bullet from the proposal. If any criterion can only be verified post-deploy, move the corresponding check into Post-deployment verification.
- **Post-implementation** — proposal status update, epic README update, FUNCTIONAL_SPECIFICATION / TECHNICAL_SPEC updates if relevant, smoke-test check.

Number tasks sequentially across all sections.

If the task count climbs past the repo's `backlog.feature_size.goal_max_tasks` goal (default ~10), pause before writing the file and re-ask the independence question: is this still **one independent, mergeable deliverable**, or has the proposal grown into several? Size alone isn't a blocker — a coherent feature slightly over the goal is fine — but a bloated task list bundling unrelated work is a signal to surface a split to the user (Step 7), not to author it silently. You cannot edit `proposal.md`, so the split is the user's call.

### Step 6 — Self-validate coherence

Before writing the files, run the cross-checks from `/feature-create` Step 6 (Proposal ↔ Tasks, Design ↔ Tasks, RAID ↔ Tasks, Effort ↔ Task count, Maintenance ↔ Tasks, Design scope ↔ Verification, UI scope ↔ UI Verification, Validation ↔ Verification). Any inconsistency means either the design or the tasks need to change before the files land — do NOT modify the proposal to paper over the gap. The proposal is the source of truth; design + tasks must conform to it.

### Step 7 — Surface proposal-level gaps to the user

If Step 3 turned up genuine contradictions between the proposal and the codebase or sibling features, **stop and ask the user before writing any files**. Do not silently expand the proposal's scope, do not silently shrink it, do not pick a side. Frame each gap as a discrete question:

> "The proposal says X. The codebase / a sibling feature says Y. Which is correct? Options: (a) update the proposal, (b) treat the codebase claim as wrong and design against the proposal as written, (c) defer."

Do not edit `proposal.md` yourself. The user decides whether the proposal needs an amendment, and if so, runs that amendment as its own change (often via `/feature-review` or a manual edit they then validate).

Wait for the user's answer before proceeding. The answers feed back into Step 4 / Step 5.

If Step 3 turned up no proposal-level gaps, skip this step and proceed.

### Step 8 — Write the files

Write `design.md` and `tasks.md` to the feature folder. Do **not** touch `proposal.md` under any circumstance — even if Step 7 surfaced a gap and the user said "fix it in the proposal," the user owns that edit.

### Step 9 — Final output

After writing, output a summary:

```
Authored backlog/<epic>/<feature-name>/
  - proposal.md  (unchanged)
  - design.md    (authored — <N> files-to-create, <M> files-to-modify, migration: yes/no, UI: yes/no)
  - tasks.md     (authored — <T> tasks across <S> sections)

Proposal-level gaps surfaced to user: <list, or "None">
Resolved questions honoured in design: <list of struck-through proposal questions, or "None">
Sibling feature overlap detected: <list, or "None">

Next step: Run /feature-review backlog/<epic>/<feature-name>/ to validate the design + tasks against the proposal and the codebase. Iterate until convergence or only USER items remain.
```

## Hard rules (non-negotiable)

1. **Never modify `proposal.md`.** If the design surfaces a real proposal-level gap, surface it to the user in Step 7 — do not edit.
2. **Never overwrite `design.md` or `tasks.md` without `--force` AND explicit chat confirmation.** Regenerating these blows away prior work; treat the overwrite as a destructive action.
3. **Never expand scope beyond the proposal.** The proposal says three YAMLs → design ships three YAMLs. Documentation-elsewhere claims of "four" are gaps to surface, not licenses to grow.
4. **Never contradict resolved questions.** Struck-through Open Questions with rationale are binding decisions; cite them, don't relitigate.
5. **Never invent file paths.** Every backtick-quoted symbol or path in design.md must be one you read in Step 3 (or one your Step 4 plan sanctions creating). No speculation.

## Why this skill exists

The `/feature-create` skill writes all three files in one shot and is gated on the target folder being empty. That gate is correct for the from-scratch flow but blocks a recurring legitimate scenario: a proposal that has converged through several `/feature-review` iterations now needs its design + tasks authored. F3 `tenant-persona-config` was the trigger case — its proposal converged on 2026-05-04 after three rounds of refinement (PR #15 in client-agent-platform), and re-running `/feature-create` would have stomped on the converged content. `/feature-elaborate` fills exactly that gap.
