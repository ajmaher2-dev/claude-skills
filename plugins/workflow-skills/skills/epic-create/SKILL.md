---
name: epic-create
description: Scaffold a new epic folder under backlog/ with README, features/ subfolder, and (optionally) decisions/ subfolder. Right-sizes scope before scaffolding.
user_invocable: true
---

# /epic-create

Scaffold a new epic folder inside the `backlog/` structure.

This is the create-counterpart to `/epic-review`. The pair mirrors `/feature-create` ↔ `/feature-review` for symmetry across the planning surface.

## Usage

```
/epic-create <epic-name>
/epic-create <epic-name> --phase phase-N
/epic-create <epic-name> --phase phase-N "<one-line goal>"
```

`--phase` is one of `phase-1 | phase-2 | phase-3 | phase-4 | deprioritised`. If omitted, the skill asks before scaffolding (epics rarely belong outside a phase).

The optional trailing string is a one-line goal description used to seed the README's Goal section. If omitted, the section is left as a placeholder comment for the user to fill in.

## What you do

### Step 1 — Gather context

Before scaffolding anything, read:

1. `backlog/README.md` — confirm the phase structure, conventions (kebab-case, etc.), and frontmatter schema.
2. `backlog/<phase>/README.md` if `--phase` was given — understand what's already in the phase, the phase's exit criteria, and where the new epic fits in sequence.
3. List existing epic folders under all `backlog/phase-*/` directories — used in Step 2 for overlap detection.
4. `docs/templates/epic-readme.md` — the canonical README shape this skill writes.

### Step 2 — Right-sizing assessment

Epics get created weekly in this project, so right-sizing is the most valuable thing this skill does. Before scaffolding, check:

1. **Overlap with existing epics.** Scan epic READMEs and feature proposals for names, modules, or domains that intersect what the user described. If overlap is real, ask: should this be a new epic, or a feature inside an existing epic?
2. **Scope size.** Ask the user (one question, conversational): "Roughly how many features do you expect this epic to ship, and over what timeframe?"
   - **1–2 features over <1 week:** push back. This is probably a single feature or a small cluster — `/feature-create` directly may be the better fit.
   - **3–6 features over 1–4 weeks:** typical epic size. Scaffold normally.
   - **7+ features OR effort >6 weeks:** push back. Suggest splitting along an axis the user names (capability, layer, customer-segment, phase). Two related epics with a stated dependency relationship are usually clearer than one giant epic.
3. **Phase fit.** If the epic name and goal don't obviously fit the chosen phase (or the user didn't specify a phase), ask before scaffolding. Wrong-phase placement is harder to fix later than getting the answer up front.

If any of the three checks turn up a concern, surface it and wait for the user's call. Don't auto-decide.

When estimating feature count, treat a "feature" as **one independent deliverable** (one PR, one review cycle, mergeable on its own) — not a task bundle. That keeps the epic's split aligned with how `/feature-create` and `/feature-review` right-size each feature.

### Step 3 — Determine target location

- If `--phase phase-N` was provided → `backlog/<phase>/<epic-name>/`
- If `--phase deprioritised` was provided → `backlog/deprioritised/<epic-name>/`
- If no phase specified → ask the user (default suggestion: the most recently active phase from `backlog/README.md`'s phase status table)

Verify the target folder doesn't already exist. If it does, abort and report the existing folder's contents — never overwrite an epic.

### Step 4 — Scaffold the epic

Create the folder structure:

```
backlog/<phase>/<epic-name>/
├── README.md
├── decisions/           # empty; placeholder file with one-line README
└── features/            # empty
```

#### `README.md`

Populate from `docs/templates/epic-readme.md`, with as many fields filled in as possible:

- **Title:** human-readable form of the epic name (e.g., `tenant-persona-config` → `Tenant Persona Config`)
- **Status:** `NOT STARTED`
- **Owner:** `AJ Maher` (from existing epic READMEs as the convention)
- **Goal:** if the user supplied a one-line goal in the invocation, populate it. Otherwise leave the placeholder comment in place.
- **Features table:** empty table with the template's column headers. Do NOT invent feature rows — features come later via `/feature-create --epic <epic-name>`.
- **Dependencies / Notes:** leave the placeholder comments; user fills these in as the epic takes shape.

#### `decisions/`

Create the folder. Add a `decisions/README.md` with one short paragraph:

```markdown
# Decisions

Architecture decision records (ADRs) for this epic. One file per decision, named `NNN-short-title.md`. Use `docs/templates/adr.md` as the template.

ADRs land here when:
- A technology choice is made (library, framework, data store)
- A structural decision is made that crosses multiple features
- A trade-off is made that future readers might second-guess

Single-feature decisions belong in that feature's `proposal.md` Decision Log, not here.
```

#### `features/`

Create the folder, empty. No `.gitkeep` file — git tracks the folder once features land in it. If the project's git config requires empty-folder placeholders, add a one-line `features/README.md` instead.

### Step 5 — Update the phase README

If the epic was scaffolded under a phase (`phase-1`, `phase-2`, etc.), update that phase's `README.md` to add a row for the new epic in its epic table. Keep the row minimal: epic name, status `NOT STARTED`, link to the folder, one-line description from the user's goal (or "TBD" if not provided).

If the phase README has no epic table or has a different structure, leave it alone and note in Step 6 that the user should add the epic manually.

### Step 6 — Final output

```
Created backlog/<phase>/<epic-name>/
  - README.md           — populated from template (status: NOT STARTED, owner: AJ Maher)
  - decisions/README.md — ADR landing page
  - features/           — empty, awaiting /feature-create

Phase: <phase-name>
Goal: <one-line goal or "not provided — fill in README.md">

Right-sizing notes (if any): <e.g., "user described 8 features; suggested splitting along capability axis. User confirmed single epic with explicit phase-2 follow-on noted in README Notes.">

Phase README updated: <yes / no — manual edit needed>

Next steps:
1. Fill in the README's Goal and Notes sections if not done in this run
2. Run /feature-create <feature-name> --epic <epic-name> for the first feature(s)
3. Run /epic-review <epic-name> after a few features land to check completeness
```

## Hard rules

1. **Never scaffold an epic without phase confirmation.** Wrong-phase placement is harder to fix than asking up front.
2. **Never overwrite an existing epic folder.** If the target exists, abort and report.
3. **Never invent features in the README's features table.** That's `/feature-create`'s job. Empty table only.
4. **Push back on under-sized epics.** A 1-feature "epic" is a feature; suggest `/feature-create` instead.
5. **Push back on over-sized epics.** A 7+ feature, multi-month epic should be split before scaffolding, not after.

## Why this skill exists

Epics are scaffolded roughly weekly in this project, and the boilerplate (README from template, `decisions/` folder, phase README update) is identical every time. Doing it by hand is fast but error-prone — phase placement gets wrong, owner field gets forgotten, the `decisions/` folder gets skipped and ADRs end up scattered. More importantly, the right-sizing check in Step 2 is the most valuable part of epic creation and the easiest to skip when you're scaffolding manually. A skill makes the right-sizing question unavoidable.

`/epic-create` does NOT create features — that's `/feature-create --epic <epic-name>`. The two are deliberately separate so a feature can be created days or weeks after the epic, when the design has firmed up. Bundling them would force premature feature definition.
