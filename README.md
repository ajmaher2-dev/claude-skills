# aj-skills

A Claude Code plugin marketplace providing the **`workflow-skills`** plugin: a
suite of planning and review skills.

## `workflow-skills`

Bundled skills (invoked namespaced as `workflow-skills:<name>`):

| Skill | Purpose |
|-------|---------|
| `feature-create` | Scaffold a feature plan (proposal + design + tasks) |
| `feature-elaborate` | Author design + tasks for an existing proposal |
| `feature-implement` | Implement a feature end-to-end |
| `feature-review` | Review a feature plan for completeness and alignment |
| `epic-create` | Scaffold an epic folder |
| `epic-review` | Review an epic for coherence and completeness |
| `codebase-review` | Comprehensive codebase review |
| `code-review` | Post-implementation review of changed files |
| `deslop` | Scan code for AI-generated slop patterns |

The skills are **repo-agnostic**: they read `.claude/repo-conventions.yaml` (if
present) to parameterise paths, patterns, and conventions to the active repo.

## Use it

In a consuming repo, commit `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "aj-skills": { "source": { "source": "github", "repo": "ajmaher2-dev/claude-skills" } }
  },
  "enabledPlugins": { "workflow-skills@aj-skills": true }
}
```

Or add it ad hoc:

```
claude plugin marketplace add ajmaher2-dev/claude-skills
claude plugin install workflow-skills@aj-skills
```

## Cloud sessions

In hosted Claude Code cloud sessions the plugin auto-installs from committed
settings, reconciling on the first turn boundary after a cold start (so the very
first turn of a fresh session may not have it yet). A **single-turn /
non-interactive run** that never gets a second turn should install via a
pre-session setup script (`claude plugin install workflow-skills@aj-skills`) in
the cloud environment settings.
