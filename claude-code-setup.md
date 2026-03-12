# Claude Code Setup

> **Scope**: CLAUDE.md authorship, PostToolUse hooks for auto-formatting, and permission controls
> **Prerequisites**: None — do this first
> **Deliverables**: `CLAUDE.md` written, `.claude/settings.json` with hooks, `.claude/settings.local.json` with permissions
> **Estimated effort**: M

Everything you need to configure Claude Code for maximum effectiveness. Path-scoped rules, agents, and skills are covered in [Claude Code Rules](claude-code-rules.md).

---

## CLAUDE.md — The Project Brain

`CLAUDE.md` is the single most important file for Claude Code. It's loaded as the system prompt for every conversation, so it directly determines code quality.

### Recommended Sections

Fill in the product sections from your [Product Context](product-context.md) work. If you haven't answered those questions yet, do that first.

```markdown
# Project Name

## What [Project] Does
**[One-line pitch — what it is in ≤10 words]**

### User Problem
- [Pain point 1 — what users can't do or what's painful]
- [Pain point 2]
- [Why existing alternatives don't solve it]

### Solution
[How the project addresses these problems — concrete examples, not abstract]

### Who It's For
[Primary persona — specific behaviors and goals, not demographics]

## Core Principles (YOU MUST FOLLOW)
- **Python 3.14+ Required** - Modern syntax, type safety
- **DRY Where It Counts** - No duplicated business logic; structural patterns may repeat for readability
- **Immutable Domain** - Pure transformations, no side effects
- **Batch-First** - Collections over single items
- [Your project-specific principles]

## Architecture
**Dependency Flow**: Interface → Application → Domain ← Infrastructure

[Layer descriptions with directory mappings]

## Essential Commands
[Dev, test, lint, format, type-check commands]

## Required Coding Patterns
### Python 3.14+ Syntax (REQUIRED)
[DO / DON'T examples — see Python Tooling guide]

## Testing
### Self-Check (after every implementation)
1. Did I write tests? If not, write them now
2. Right level? Domain=unit, UseCase=unit+mocks, Repository=integration
3. Beyond happy path? Error cases, edge cases, validation
4. Using existing factories from tests/fixtures/?
5. Tests pass? `uv run pytest tests/path/to/test_file.py -x`

### When to Run What
- **During implementation**: targeted test file only (`-x`, `-k`, `--lf`)
- **Before committing**: `uv run pytest` (full fast suite)
- **NEVER** run the full suite after every small edit

## Documentation Map
[Links to deeper docs]
```

### Writing Style Tips

- **Use imperative language**: "YOU MUST FOLLOW" — Claude treats CLAUDE.md as authoritative instructions
- **Keep it under 300 lines** — lines beyond that risk context truncation; link to deeper docs
- **Include a Self-Check pattern** — a checklist Claude runs after every implementation to catch its own gaps
- **Be specific about commands** — include the exact `uv run` prefix, flag combinations, etc.

---

## .claude/ Directory Configuration

### settings.json — PostToolUse Hooks

Hooks run automatically after Claude uses the Edit or Write tools. This ensures every file Claude touches is instantly formatted and linted.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path // empty' | xargs -I{} uv run ruff check {} --fix --quiet 2>/dev/null; exit 0"
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path // empty' | grep -E '\\\\.(tsx?|jsx?)$' | xargs -I{} pnpm --prefix web exec biome check --write {} 2>/dev/null; exit 0"
          }
        ]
      }
    ]
  }
}
```

**How it works**:
- `jq` extracts the file path from Claude's tool input JSON
- First hook: runs `ruff check --fix` on any Python file edit
- Second hook: runs `biome check --write` on TypeScript/JavaScript edits (filtered by `grep`)
- `exit 0` ensures hook failures never block Claude's workflow
- `2>/dev/null` suppresses noise from files outside the tool's scope

### settings.local.json — Permissions Matrix

This file is **gitignored** — it's per-developer. It controls what Claude can do without asking.

```json
{
  "permissions": {
    "allow": [
      "Bash(uv run pytest:*)",
      "Bash(uv run ruff check:*)",
      "Bash(uv run ruff format:*)",
      "Bash(uv run basedpyright:*)",
      "Bash(pnpm:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(ls:*)",
      "Bash(find:*)",
      "Bash(grep:*)",
      "WebSearch"
    ],
    "deny": []
  }
}
```

**Principle**: allow read-only operations and dev tooling by default; require confirmation for destructive operations (git push, file deletion, etc.).
