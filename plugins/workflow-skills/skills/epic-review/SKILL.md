---
name: epic-review
description: Review an epic for strategic alignment (coherence, feature split, sequencing, arch docs) AND template completeness. Strategic findings come first; completeness gaps can be auto-generated. Run periodically as features grow and change.
user_invocable: true
---

# /epic-review

Review an epic at two levels in one pass:

1. **Strategic alignment** — do the features still ladder to the epic goal? Are they sized and sequenced right? Have arch docs and ADRs kept up with the epic's evolution? *(Advisory only — never auto-fixed.)*
2. **Template completeness** — do epic README, feature folders, and frontmatter conform to the project templates? *(Auto-generatable with `--generate`.)*

The strategic pass is the more valuable one — a complete-but-incoherent epic is worse than an incomplete-but-coherent one. The report puts strategic findings first.

## Usage

```
/epic-review <epic-name>
/epic-review backlog/<epic-name>/
/epic-review backlog/<epic-name>/ --generate          # auto-fix completeness gaps only
/epic-review backlog/<epic-name>/ --strategic-only    # skip the completeness pass
/epic-review backlog/<epic-name>/ --completeness-only # legacy behaviour: skip strategic pass
```

`--generate` only fixes mechanical completeness gaps (missing template sections, stale status table rows). Strategic findings are advisory — no auto-fix, ever.

## When to run

`/epic-review` can be invoked at any pipeline step once an epic exists. Cadence guidance to prevent both noise and drift:

- **After every 2–3 features land** — catches misalignment before it compounds.
- **Before starting a new feature in the epic** — surfaces upstream drift so the new feature is shaped against current reality, not the epic's original framing.
- **When you find yourself wanting to expand a feature's scope** — that itch is often the epic asking to be re-reviewed (or split). Run `/epic-review` before approving the scope expansion.
- **When a feature's `/feature-review` keeps surfacing items that imply the proposal is wrong** — the proposal might be downstream of an epic-level drift.
- **Before declaring an epic complete** — final coherence check: did we actually deliver the goal, or did the goal quietly become something else?

Skip if you just ran it and nothing has changed since.

## Philosophy

- **Substance over format.** Heading case, section ordering, and table-vs-bulleted-list variations all pass if the substance is intact. The skill catches genuine gaps and contradictions, not stylistic preferences.
- **Strategic findings are advisory.** Never auto-fixed. They surface for human judgment because re-shaping an epic is a decision, not a mechanical edit.
- **Completeness findings are mechanical.** Safe to auto-generate with `--generate`.

---

## What you do

### Step 1 — Locate the epic

Resolve the epic folder:
- If given a bare name (e.g., `email-platform`), look in `backlog/<name>/` first, then walk `backlog/phase-*/<name>/` if not found at top level.
- If given a path, use it directly.
- If the folder does not exist, list available epics from `backlog/` (top level + each `phase-*/`) and ask the user to confirm.

Read everything once, up front:
1. `backlog/<epic>/README.md` — epic overview (or note it is missing).
2. List all subdirectories under `backlog/<epic>/features/`.
3. For each feature folder, read whichever of `proposal.md`, `design.md`, `tasks.md` exist.
4. List and read any files under `backlog/<epic>/decisions/` (ADRs).
5. `docs/templates/epic-readme.md`, `docs/templates/proposal.md`, `docs/templates/design.md`, `docs/templates/tasks.md` — completeness templates.
6. The epic's parent phase README (e.g., `backlog/phase-1/README.md`) if the epic lives under a phase — used for sequencing checks.

---

### Step 2 — Strategic alignment review (PRIMARY)

This is the most valuable part of the skill. The strategic findings appear first in the report.

#### 2a — Coherence with the epic goal

For each feature:
- [ ] **Feature problem connects to the epic goal.** Open the feature's `proposal.md` Problem section and the epic README's Goal section. Does the feature's problem clearly serve the epic's goal? If the feature's problem mentions a domain, layer, or outcome the epic doesn't, flag it as a coherence drift.
- [ ] **Feature scope stays inside the epic's implicit scope.** If a feature's "In Scope" lists items that go beyond what a reader of the epic README would expect this epic to cover, flag it. The fix is usually one of: (a) tighten the feature's scope, (b) update the epic README to acknowledge the broader scope, or (c) split the feature out into its own epic.
- [ ] **No silent goal drift in the epic README.** If the README's Goal section reads like it was written for an earlier framing and the features are now solving a different problem, flag it. The README should reflect what the epic actually delivers.

#### 2b — Feature split sanity

- [ ] **Effort distribution is sane.** Flag features with `effort: > 5d` for split consideration; flag features with `effort: ≤ 0.5d` as candidates for either merge with a sibling, moving to `tech_debt/`, or absorption into a larger feature. (Advisory — small features may legitimately be standalone if they have an owner-facing reason to ship independently.)
- [ ] **No file-level overlap between features.** Compare `design.md` "Files to create / modify" lists across features in the epic. If two features both modify the same module heavily and both are not-yet-implemented, flag for either merge or sequencing-with-conflict-plan.
- [ ] **No problem-level duplication.** If two features' Problem sections describe the same underlying problem from different angles, flag — this often indicates one is the real feature and the other is implementation detail.
- [ ] **No coverage gaps.** Walk the epic README's Goal and check that each promise has at least one feature delivering it. If a piece of the goal has no feature, flag — either add a feature, narrow the epic's goal, or note the gap explicitly in the README.

#### 2c — Sequencing and dependencies

- [ ] **`depends-on` chains resolve.** Every feature's `depends-on` frontmatter must reference a feature that exists. Flag orphan dependencies (depends-on a feature that was renamed, deleted, or never existed).
- [ ] **No cross-epic dependencies treated as intra-epic.** If a feature depends on something outside this epic, the dependency belongs in the epic README's Dependencies section, not just in feature frontmatter.
- [ ] **Status mix is healthy.** Flag if more than 2 features are simultaneously `in-progress` (focus drift) or if any feature has been `blocked` for more than two weeks without movement (stale blocker — needs explicit re-decision).
- [ ] **No circular dependencies.** Two features depending on each other is always wrong. Flag.

#### 2d — Arch doc and ADR alignment

- [ ] **ADRs in `decisions/` don't contradict each other.** If two ADRs make conflicting claims (e.g., one says "use Redis," another says "use in-memory queue"), flag — at most one is current; the other should be marked superseded.
- [ ] **ADRs don't contradict feature designs.** If an ADR commits to pattern X and a feature design proposes pattern Y for the same concern, flag.
- [ ] **Decision Log entries match ADRs.** If a feature's proposal Decision Log references an architectural choice, that choice should also appear in `decisions/` if it's epic-scope (not feature-local). Conversely, ADRs should be referenced from the proposals they affect.
- [ ] **Epic README Notes section reflects current sequencing.** If the README says "feature A blocks feature B" and the actual `depends-on` chain or status doesn't match, flag.

#### 2e — Scope drift detection

- [ ] **Each feature is one independent deliverable.** A feature in this epic should be mergeable on its own and deliver value without waiting on a sibling. Flag features that bundle unrelated work, can't be reviewed/merged independently, or are a slice that's meaningless without another feature — mis-scoped regardless of task count.
- [ ] **Decision Log churn.** Features with 3+ rows in their Decision Log (especially ones marking "scope expanded" or "approach changed") signal that the feature has been reshaped multiple times. Flag for: is the right-sized feature emerging, or is this evidence the epic is mis-scoped?
- [ ] **Resolved-then-reopened questions.** Features with struck-through Open Questions that have been re-edited or re-opened — flag, the resolution may not have stuck and the underlying question may be epic-level.
- [ ] **Effort estimate inflation.** If a feature's `effort` has grown across edits (compare current value to git history if accessible), flag. Growing effort estimates on a single feature usually mean either incomplete original analysis or epic-level scope creep.

#### 2f — Product alignment

The most fundamental strategic check: does the epic itself serve stated product positioning?

- [ ] **Product alignment section present in epic README.** If the repo has product docs at `docs/product/{vision,strategy,products}.md`, the epic README should have a "Product alignment" section addressing which focus area / territory / product the epic serves. If the repo uses different product framing, the section should cite that instead. Flag missing section.
- [ ] **Alignment is substantive, not boilerplate.** The section names what the epic contributes (e.g., "delivers the Intelligence Layer product per `docs/product/products.md` §2"), not generic statements ("aligns with product strategy"). Flag boilerplate.
- [ ] **Compromises flagged explicitly.** If the epic trades off against a product commitment, the trade-off is named in the section — never silent. Silent compromise is worse than open compromise; flag at review.
- [ ] **Feature-level alignment propagates.** Sample 2-3 feature proposals from the epic; each should have its own Product alignment subsection consistent with the epic's. If features and epic alignment statements drift apart, flag (the epic may be reshaping mid-flight and the README is stale).
- [ ] **Honesty test:** does the epic, as scoped in the README + features, make the product story more or less honest about what the platform actually does? Flag if "less."

#### 2g — Platform ADR conformance

Beyond intra-epic ADR consistency (covered in 2d), check that the epic engages with platform-wide ADRs.

- [ ] **Platform ADRs are linked from the epic.** If the repo has platform-wide ADRs (typically in `docs/architecture/NNN-*.md` or the repo's equivalent), the epic README's Decisions section (or `decisions/README.md`) should explicitly link the platform ADRs that bind this epic, not just list epic-scoped ADRs. Flag missing platform-ADR references.
- [ ] **Each feature's design addresses relevant platform ADRs.** Sample 2-3 feature design.md files from the epic; each should have an "ADR conformance" section (or equivalent) that names which platform ADRs bind the feature and how the feature satisfies them (or deviates with recorded disposition). Flag features that touch ADR-bound concerns without addressing conformance.
- [ ] **No silent platform-ADR violations across the epic.** Scan the epic's feature designs collectively for patterns that would violate a platform ADR constraint without being recorded as a deviation. Examples: features importing a vendor SDK directly when ADR 002 (or repo's equivalent) mandates a router; features assuming multi-tenant runtime when an ADR mandates single-tenant; etc. Flag silent violations.
- [ ] **Open investigations are tracked.** If a platform ADR has open-investigation deviations (e.g., a "non-deferrable" DEV-NN row), epic features that interact with that concern should reference the investigation. Flag features that proceed as if the investigation is settled when it's not.

---

### Step 3 — Template completeness review (SECONDARY)

This is the carry-over from the original `/epic-audit` behaviour. Mechanical, auto-generatable. Findings appear after the strategic findings in the report.

#### 3a — Epic README

Check `backlog/<epic>/README.md` against `docs/templates/epic-readme.md`:

- [ ] **File exists**
- [ ] **Status field** — one of: NOT STARTED | IN PROGRESS | COMPLETE
- [ ] **Owner field** — present and non-empty
- [ ] **Goal section** — present and substantive (not placeholder text)
- [ ] **Features table** — present with columns: Feature | Status | Priority | Effort | Description
- [ ] **Features table is current** — every subfolder in `features/` has a row; no rows reference features that no longer exist
- [ ] **No stale status** — features marked one status in the README but a different status in their frontmatter

#### 3b — Each feature folder

For each folder in `backlog/<epic>/features/`:

**File presence:**
- [ ] `proposal.md` exists
- [ ] `design.md` exists
- [ ] `tasks.md` exists

If any file is missing, note it and skip the content checks for that file.

**`proposal.md` content checks. All heading checks below are case-insensitive.**

Frontmatter:
- [ ] YAML frontmatter is present
- [ ] `epic` field matches the parent epic folder name
- [ ] `name` is present and non-empty
- [ ] `status` is one of: `not-started | in-progress | ready | blocked | done`
- [ ] `priority` is one of: `P0 | P1 | P2`
- [ ] `effort` is present (e.g., `0.5d`, `3d`, `2w`)
- [ ] `owner` is present

Required sections (present and not placeholder/empty):
- [ ] `## Problem` — specific, references files/errors/user impact (not just "we need X")
- [ ] `## Proposed Solution` — describes what will actually change
- [ ] `## Scope` — has both in-scope and out-of-scope items
- [ ] `## RAID` — present in **one of two acceptable formats**:
  - **Subsection format**: `### Risks`, `### Assumptions`, `### Issues`, `### Dependencies` (and optionally `### Open Questions`) as separate subsections, each non-empty or explicitly "None"
  - **Table format**: a single markdown table covering risk/assumption/issue/dependency content
- [ ] `## Validation` — acceptance criteria present (observable outcomes, not just "it works")
- [ ] `## Decision Log` — table present (can be empty if no decisions yet)

**`design.md` content checks. All heading checks are case-insensitive.**

Required sections:
- [ ] `## Architecture` — present and non-empty
- [ ] Reuse assessment — mentions what existing code was considered before proposing new files
- [ ] Files-to-create / Files-to-modify list — at least one entry with paths to create or modify
- [ ] **Substantive design detail** — either under `## Detailed Design` or distributed across `## Architecture` and component-specific subsections. Substance over heading name; reject only if shallow.
- [ ] `## Testing Strategy` (or `## Testing strategy`) — present. **BDD Gherkin scenarios are required for behavioural features** (new endpoints, workflows, business logic). Not required for pure visual/refactor/cleanup features — prose plus bulleted scenarios is fine for those.
- [ ] `## Database` — present (migration details or explicit "None")
- [ ] `## Security` (or `## Security Considerations`) — present (specific concerns or explicit "No new surface area")
- [ ] `## Maintenance Implications` (accept `## Maintenance` as synonym) — present if the feature adds runtime behaviour; explicitly states "None" otherwise

**`tasks.md` content checks. All heading checks are case-insensitive.**

Required structure:
- [ ] Tasks use checkbox format (`- [ ] N. description`)
- [ ] Tasks are numbered sequentially within each section (numbering can restart per section or run continuously — both fine)
- [ ] At least one implementation section with **specific** tasks (real file paths or specific responsibilities, not vague "implement feature")
- [ ] **Section structure** — accept either:
  - **Phase-numbered** (`## Phase 1 — <Title>`, `## Phase 2 — <Title>`), OR
  - **Domain-named** (`## Implementation`, `## Backend`, `## Frontend`, `## Database`, `## i18n`, etc.)
- [ ] **Testing/verification section present** with test tasks — accept any of: `## Testing`, `## Verification`, `## Tests`. Both is fine.
- [ ] `## Post-implementation` section present with at least: status update on the proposal frontmatter, AND epic README update task

#### 3c — Optional but valuable docs

Check whether the epic would benefit from docs it doesn't currently have:

**ADRs (`backlog/<epic>/decisions/`):**
- Suggest creating ADRs if:
  - The epic README or any proposal mentions a technology choice or architectural decision
  - Any proposal Decision Log table has entries that are epic-scope (not feature-local)
  - The epic is large (5+ features) and has no ADRs

**DESIGN_VISION.md:**
- Suggest only if the epic involves user-facing design (UI, UX flows, end-user workflows) and no design vision exists

Do not suggest docs that would add no value for the epic's scope.

---

### Step 4 — Produce the review report

Output in this format. **Strategic findings appear first** — they are the more important findings, and putting them first means they're not buried below mechanical completeness check-marks.

```markdown
## Epic Review: <epic-name>

**Path:** backlog/<epic-name>/
**Features:** N
**ADRs:** M
**Date:** <today>
**Overall:** ALIGNED ✓ | NEEDS REVIEW | NEEDS WORK

---

## Strategic findings

### Coherence with epic goal
- <finding or "All features ladder cleanly to the epic goal.">

### Feature split sanity
- <finding or "Feature sizing and split look right.">

### Sequencing and dependencies
- <finding or "Dependencies resolve cleanly; status mix healthy.">

### Arch doc and ADR alignment
- <finding or "ADRs are consistent and aligned with feature designs.">

### Scope drift signals
- <finding or "No drift signals detected.">

**Strategic verdict:** ALIGNED | NEEDS DISCUSSION (N items for human judgment)

---

## Completeness findings

### Epic README
- PASS/FAIL: <check> — <detail>

### Features

| Feature | proposal | design | tasks | Notes |
|---|---|---|---|---|
| <name> | PASS/GAPS/MISSING | PASS/GAPS/MISSING | PASS/GAPS/MISSING | <one-line> |

**Specific gaps:**
- `<feature>/proposal.md`: <gap>
- `<feature>/design.md`: <gap>
- ...

### Optional docs
- <suggestion or "None recommended">

**Completeness verdict:** COMPLETE | N gaps (auto-fixable with --generate)

---

## Next steps

**Strategic items (human judgment required, never auto-fixed):**
1. <action — e.g., "Decide whether to split feature X (effort 8d) into two features">
2. <action — e.g., "Update epic README Goal section — current goal text predates the addition of features Y and Z">
3. ...

**Completeness items (auto-fixable):**
- N gaps. Run `/epic-review <epic-name> --generate` to fix mechanical completeness gaps.
- (Strategic items above will NOT be auto-fixed; they need your call first.)
```

If the run was `--strategic-only` or `--completeness-only`, omit the skipped section.

---

### Step 5 — Generate fixes (only if `--generate` AND only for completeness)

For each completeness gap, generate the missing file or section using the corresponding template from `docs/templates/`, populated with as much real content as can be inferred from:
- The epic README
- Other existing files in the same feature folder
- The feature folder name and any naming conventions

Rules:
- **Strategic findings are NEVER auto-fixed.** `--generate` ignores them. They remain in the report as human-judgment items.
- Never overwrite a file that already exists — only create files that are missing entirely.
- For incomplete files (present but missing sections), list the specific sections to add and ask whether to edit the file.
- Populate frontmatter fields that can be inferred (epic name, feature name, owner from README).
- Leave placeholder comments (`<!-- ... -->`) where content requires human input.
- After generating, update the epic README features table if it is out of date.

After generating, output a list of files created/modified, and re-print the strategic findings (so they don't get lost in the generation output).

---

## Hard rules

1. **Strategic findings are advisory.** Never auto-fixed. Re-shaping an epic is a decision, not a mechanical edit.
2. **Strategic findings come first in the report.** If mechanical findings come first, the part that matters gets buried.
3. **`--generate` is scoped to completeness only.** Even if the user passes `--generate` with strategic findings present, only the completeness gaps are fixed; strategic items are reprinted at the end so they're not lost.
4. **Substance over format.** Don't flag a feature for using a single RAID table when subsections were the template. Both are accepted.
5. **No persistent review reports** (for now). Each run produces a transient report. If drift detection across reviews becomes valuable, add `reviews/YYYY-MM-DD.md` artifacts then.
