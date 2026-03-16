# Claude Code Setup

> **Scope**: CLAUDE.md authorship, hooks for auto-formatting, permission controls
> **Prerequisites**: None — do this first
> **Deliverables**: `CLAUDE.md` written, `.claude/settings.json` with hooks, `.claude/settings.local.json` with permissions
> **Estimated effort**: M

Everything you need to configure Claude Code for maximum effectiveness. Path-scoped rules, agents, and skills are covered in [Claude Code Rules](claude-code-rules.md).

**Official docs**: [code.claude.com/docs/en/](https://code.claude.com/docs/en/)

---

## CLAUDE.md — The Project Brain

`CLAUDE.md` is the single most important file for Claude Code. It's loaded as a user message after the system prompt for every conversation, so it directly shapes code quality and adherence to project conventions.

### Recommended Sections

Fill in the product sections from your [Product Context](product-context.md) work. If you haven't answered those questions yet, do that first.

```markdown
# Project Name

## What [Project] Does
**[One-line pitch — what it is in <=10 words]**

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
**Dependency Flow**: Interface -> Application -> Domain <- Infrastructure

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

### Keeping CLAUDE.md Effective

- **Use imperative language**: "YOU MUST FOLLOW" — Claude treats CLAUDE.md as authoritative instructions
- **Keep it under 200 lines** — longer files consume more context and reduce adherence. Link to deeper docs instead
- **Use `@path` imports** to reference detailed docs without bloating CLAUDE.md: `@docs/architecture/layers-and-patterns.md` inlines that file's content. Supports relative, absolute, and `@~/` home-dir paths, up to 5 hops of recursive imports
- **Include a Self-Check pattern** — a checklist Claude runs after every implementation to catch its own gaps
- **Be specific about commands** — include the exact `uv run` prefix, flag combinations, etc.

### Where Content Comes From

Each guide in this project produces concrete deliverables — CLAUDE.md sections, `.claude/rules/` files, tool configs. Use the guides as source material, then bake the patterns into your project's own config. Claude Code reads your `CLAUDE.md` and `.claude/` directory every session; it never reads `docs/dev-setup-guide/`.

---

## .claude/ Directory Configuration

### settings.json — Hooks

Hooks run automatically in response to Claude Code events. The most common use is auto-formatting after file edits.

**Four handler types** are available:
- `command` — run a shell command (most common)
- `http` — POST to a URL
- `prompt` — send to a Claude model for LLM-based evaluation
- `agent` — spawn a subagent with tool access

**Major hook events** (22 total — see official docs for the full list):
- `PostToolUse` — after Claude uses a tool (most common for formatters)
- `PreToolUse` — before tool execution (validation, guards)
- `UserPromptSubmit` — before processing user input
- `Notification` — when Claude sends a notification
- `Stop` — when Claude finishes a response
- `SessionStart` / `SessionEnd` — conversation lifecycle

**Example** — auto-format on every file edit:

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

### settings.local.json — Permissions Matrix

This file is **gitignored** — it's per-developer. It controls what Claude can do without asking.

```json
{
  "permissions": {
    "allow": [
      "Bash(uv run pytest *)",
      "Bash(uv run ruff check *)",
      "Bash(uv run ruff format *)",
      "Bash(uv run basedpyright *)",
      "Bash(pnpm *)",
      "Bash(git status *)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(ls *)",
      "Bash(find *)",
      "Bash(grep *)",
      "WebSearch"
    ],
    "deny": []
  }
}
```

**Principle**: allow read-only operations and dev tooling by default; require confirmation for destructive operations (git push, file deletion, etc.).

**Note**: The legacy `:*` suffix syntax (e.g., `Bash(uv run pytest:*)`) is deprecated. Use a space before the wildcard: `Bash(uv run pytest *)`.

### Plugins

Claude Code supports distributable plugins that bundle skills, agents, hooks, MCP servers, and LSP servers. Enable them in `settings.json` via `enabledPlugins`. See the [official docs](https://code.claude.com/docs/en/) for available plugins and authoring your own.
