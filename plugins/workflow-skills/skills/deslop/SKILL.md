---
name: deslop
description: Scan code for AI-generated slop patterns — restated comments, hedging, broad exceptions, empty schemas, stubs, cross-language contamination, speculative abstractions. Fast, targeted, no plans generated.
user_invocable: true
---

# /deslop

Fast, targeted scan for AI-generated code patterns that degrade quality. Complements `/codebase-review` (which is comprehensive but heavy) and `/simplify` (which is general-purpose).

## Usage

```
/deslop                          # scan changed files (git diff)
/deslop <path>                   # scan specific file or directory
/deslop --all                    # scan the whole repo's source dirs (read from conventions)
/deslop --fix                    # scan + auto-fix safe patterns (HIGH certainty only)
```

This skill is **repo-agnostic.** It reads `.claude/repo-conventions.yaml` (schema: `~/.claude/repo-conventions-schema.md`) to determine which directories to scan when `--all` is used.

## What you do

### Step 0 — Read repo conventions

Read `.claude/repo-conventions.yaml`. Cache `package`, `backend.test_dir`, `frontend.dirs`. These define the source roots scanned when `--all` is used. If the file is missing, auto-detect (`pyproject.toml` for the package, `package.json` + framework dependency for frontend dirs).

### Step 1 — Determine scope

Based on the invocation:

- **No args (default):** Get changed files via `git diff --name-only HEAD` and `git diff --name-only --cached`. Scan only those files. If no changes, tell the user and stop.
- **`<path>` provided:** If it's a file, scan that file. If it's a directory, scan all source files in it recursively. Source extensions: `.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`, `.rs` (adapt to the project's languages).
- **`--all`:** Scan `<package>/` (backend) and each entry in `<frontend.dirs>/src/` (frontend, if declared). Exclude common build/cache dirs: `<backend.migrations.dir>` (if declared), `node_modules/`, `__pycache__/`, `.venv/`, `venv/`, `dist/`, `build/`, `.next/`, `target/`.

### Step 2 — Read CLAUDE.md Anti-Slop Rules

Read `CLAUDE.md` to load the current Anti-Slop Rules. These are the authoritative rules. If the rules in this skill conflict with CLAUDE.md, CLAUDE.md wins.

### Step 3 — Scan for patterns

For each file in scope, use Grep and Read tools to check for the patterns below. Do NOT use Bash grep/rg — use the Grep tool.

**IMPORTANT:** Read the actual code context around each match. A pattern match is not automatically a finding — apply judgment. For example, `except Exception` in middleware is acceptable per CLAUDE.md rules.

#### Pattern catalog

Patterns are grouped by certainty level. HIGH certainty patterns can be auto-fixed with `--fix`. MEDIUM patterns are flagged for human review.

##### Comments & Documentation (MEDIUM)

| ID | Pattern | How to detect | Auto-fix? |
|----|---------|---------------|-----------|
| C1 | Restated comment | Comment on line N restates what line N+1 does. Read both lines and compare semantics. | No — requires judgment |
| C2 | Hedging language | Grep for: `should work`, `probably`, `for now`, `hopefully`, `might need`, `may need to`, `may need to change` in comments (`#` or `//` lines) | No |
| C3 | Buzzword comment | Grep for: `robust`, `scalable`, `enterprise-grade`, `production-ready`, `leverages`, `best practices`, `efficient`, `optimized` in comments/docstrings | No |
| C4 | AI preamble | Grep for: `Here's a`, `This is a`, `Let me`, `I'll` at start of comments/docstrings | No |
| C5 | Signature-restating docstring | Docstring that just restates the function name with minor rewording. Read function def + docstring, compare. | No |
| C6 | Section banner comment | Grep for: `# ===`, `# ---`, `// ===`, `// ---` (exclude migration files) | Yes — remove line |

##### Error Handling (MEDIUM-HIGH)

| ID | Pattern | How to detect | Auto-fix? |
|----|---------|---------------|-----------|
| E1 | Broad except in business logic | Grep for `except Exception` in files NOT in `middleware/`, `agents/`, `workflows/activities/`. Read context to confirm it's not a top-level handler. | No |
| E2 | Empty except block | Grep for `except.*:\s*$` followed by `pass` on next line. Read context — acceptable in JSON parsing fallbacks per CLAUDE.md. | No |
| E3 | Defensive None check on typed param | Function declares typed parameter, then immediately checks `if param is None`. Read function signature + body. | No |

##### Structure (MEDIUM)

| ID | Pattern | How to detect | Auto-fix? |
|----|---------|---------------|-----------|
| S1 | Empty model/schema body | Grep for `class.*BaseModel` or `class.*Base\)` followed by docstring + `pass` with no fields. | No |
| S2 | Stub signal/event handler | Grep for `@workflow.signal` or `@workflow.query` where the handler body is only a log statement + TODO. | No |
| S3 | Single-method utility class | Read class definitions — if only one method besides `__init__`, flag it. Skip protocol/interface implementations. | No |
| S4 | Forwarding wrapper | Function body is a single `return other_func(same, args)` with no transformation. | No |

##### Language Discipline (HIGH)

| ID | Pattern | How to detect | Auto-fix? |
|----|---------|---------------|-----------|
| L1 | JS/Java in Python | Grep `.py` files for: `.push(`, `.length`, `===`, `!==`, `.forEach(`, `.equals(`, `.isEmpty(`, `null` (outside strings), `const `, `let `, `var ` in comments | Yes — replace with Python equivalent |
| L2 | Debug artifacts | Grep for: `print(` (not in test files), `pprint(`, `breakpoint()`, `pdb.set_trace()`, `console.log(`, `debugger;` in non-test production files | Yes — remove line |

##### Completeness (MEDIUM-HIGH)

| ID | Pattern | How to detect | Auto-fix? |
|----|---------|---------------|-----------|
| T1 | Placeholder implementation | Grep for `raise NotImplementedError` (not in abstract methods), `return {}` or `return []` as sole function body | No |
| T2 | Vague TODO | Grep for `TODO` or `FIXME`. Read the comment — flag if it lacks a ticket reference, concrete condition, or actionable description. `# TODO: fix this` is vague. `# TODO(GH-234): migrate to tool_use` is fine. | No |
| T3 | Speculative code | Harder to detect via grep. When reading files for other patterns, note: unused parameters, branches with `# future:` comments, abstractions used only once. | No |

### Step 4 — Compile findings

Group findings by file, sorted by severity (HIGH first, then MEDIUM).

### Step 5 — Auto-fix (if `--fix` flag)

Only fix HIGH-certainty patterns (L1, L2, C6). For each fix:
1. Read the file
2. Show the fix to be applied
3. Apply via Edit tool
4. Note what was fixed

Do NOT auto-fix MEDIUM patterns — they require human judgment.

### Step 6 — Report

Output findings in this format:

```markdown
## Deslop Report — YYYY-MM-DD

**Scope:** <what was scanned — changed files / path / full codebase>
**Files scanned:** N
**Findings:** N (N HIGH, N MEDIUM)
**Auto-fixed:** N (only if --fix was used)

### Findings by File

#### `<file_path>`

| # | Pattern | Line | Certainty | Description |
|---|---------|------|-----------|-------------|
| 1 | C1 | 42 | MEDIUM | Comment "# Build the prompt" restates `system_prompt = load_prompt(...)` |
| 2 | E1 | 128 | MEDIUM | Broad `except Exception` in service method — catch specific types |

<details>
<summary>Code context</summary>

```python
# line 42
# Build the prompt          <-- slop: restates next line
system_prompt = load_prompt("email/reply_system")
```

</details>

### Summary by Pattern

| Pattern | Count | Certainty | Auto-fixable? |
|---------|-------|-----------|---------------|
| C1: Restated comment | 3 | MEDIUM | No |
| E1: Broad except | 12 | MEDIUM | No |
| L2: Debug artifact | 1 | HIGH | Yes |
| **Total** | **16** | | |

### Recommended Actions

1. **Address HIGH findings first** — these have clear fixes
2. **Review MEDIUM findings** — use judgment on each; some may be intentional
3. **Run `/simplify`** on files with structural findings (S1-S4) for deeper refactoring
```

## Execution Constraints

1. **Read actual code** — Every finding must show the real code. Do not flag based on grep matches alone without reading context.
2. **Respect exceptions** — CLAUDE.md defines acceptable cases for each rule (e.g., `except Exception` in middleware). Honor them.
3. **No plans** — This skill produces a report only. It does not create plans or modify backlog. If findings are systemic, recommend running `/codebase-review`.
4. **No code changes unless `--fix`** — Without the fix flag, this is read-only.
5. **Be precise** — False positives erode trust. When in doubt, skip the finding rather than flag a false positive.
6. **Fast execution** — Use Grep tool for bulk pattern matching, then Read for context verification. Don't read every file line-by-line.
