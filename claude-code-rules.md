# Claude Code Rules, Agents & Skills

> **Scope**: Path-scoped `.claude/rules/` files, specialist subagents, and reference skills
> **Prerequisites**: [Claude Code Setup](claude-code-setup.md)
> **Deliverables**: `.claude/rules/` directory with path-scoped rule files for each architectural layer; optionally, agent and skill definitions
> **Estimated effort**: M

---

## rules/ — Path-Scoped Instructions

Rules are markdown files in `.claude/rules/` that give Claude persistent, scoped instructions. Unlike CLAUDE.md (which loads every session), rules use `paths:` frontmatter to load **only** when Claude reads or edits matching files. This keeps the context window lean — Claude performs better with less noise.

### Core Principles

1. **Path-scope everything** — without `paths:`, a rule loads unconditionally. Almost everything can be scoped.
2. **No duplication with CLAUDE.md** — CLAUDE.md summarizes ("layer rules are in `.claude/rules/`"), rules detail. Never repeat the same information in both.
3. **5-20 lines per file** — for every line, ask: "Would removing this cause Claude to make a mistake?" If not, cut it. Larger files (up to ~50 lines) are acceptable when the topic genuinely requires it (e.g., test mechanics, frontend patterns).
4. **One topic per file** — don't mix React architecture with visual design rules. Split them.
5. **Only codify what you follow today** — aspirational rules Claude can't verify from code waste context.

### What Goes Where

| Put it in... | When... |
|---|---|
| CLAUDE.md | Every session needs it: project overview, build commands, self-check steps |
| `.claude/rules/` (with paths) | Only relevant when editing specific files: layer rules, import boundaries |
| `.claude/rules/` (no paths) | Universally needed but too detailed for CLAUDE.md (rare — prefer scoping) |
| Auto memory | Claude discovers it itself: build quirks, debugging patterns, user prefs |

### Recommended Rule Files

Start with your **architecture boundaries** — each layer that has its own import rules gets its own file:

```
.claude/rules/
├── domain-purity.md          # paths: ["src/domain/**"]
├── application-patterns.md   # paths: ["src/application/**"]
├── infrastructure-patterns.md # paths: ["src/infrastructure/**"]
├── interface-patterns.md      # paths: ["src/interface/**"]
├── api-layer-patterns.md      # paths: ["src/interface/api/**"]
├── test-patterns.md           # paths: ["tests/**"]
├── web-frontend-patterns.md   # paths: ["web/src/**"]
├── web-design-system.md       # paths: ["web/src/components/**", "web/src/pages/**"]
├── python-conventions.md      # paths: ["src/**/*.py", "tests/**/*.py"]
└── version-management.md      # paths: [7 specific files]
```

### Example: Layer Rule

```markdown
---
paths:
  - "src/domain/**"
---
# Domain Layer Rules
- NEVER import from infrastructure, application, or interface layers
- All entities use `@define(frozen=True, slots=True)` — immutability prevents side-effect bugs
- All transformations must be pure (no side effects, no I/O)
- Repository interfaces are `Protocol` classes only (zero implementation)
```

### Example: Narrow-Scope Rule

For concerns that only matter for 2-3 files, scope tightly rather than polluting a broader rule:

```markdown
---
paths:
  - "pyproject.toml"
  - "src/__init__.py"
  - "scripts/export_openapi.py"
---
# Version Management
Version lives in pyproject.toml only. After bumping:
1. `uv sync` — updates installed metadata
2. `pnpm --prefix web sync-api` — exports OpenAPI schema + Orval codegen
```

### Example: Frontend Split

Large frontend rules should be split by concern — architecture rules load for all `web/src/` files, but design system rules only load when editing components and pages:

```markdown
# web-frontend-patterns.md — paths: ["web/src/**"]
- Three component layers: ui/ (primitives), shared/ (composites), pages/ (routes)
- Server state via Tanstack Query — no Redux/Zustand
- TypeScript strict mode — no any, no @ts-ignore

# web-design-system.md — paths: ["web/src/components/**", "web/src/pages/**"]
- Dark mode is default — never assume light backgrounds
- Display font (Space Grotesk), Body font (Newsreader), Mono font (JetBrains Mono)
- No uniform containers — use 3-level depth system (inset/flat/elevated)
```

### Common Mistakes

- **Dumping everything in CLAUDE.md** — if it only applies to `src/api/`, it shouldn't load when editing tests
- **Rules without `paths:`** — these load every session, eating context. Almost everything can be scoped
- **Duplicating between CLAUDE.md and rules** — a rule that restates "tests are mandatory" when CLAUDE.md already says it wastes context with zero benefit
- **Oversized rule files** — a 70-line file means Claude reads 70 lines every time it touches any file in that scope. Split by sub-concern or trim aggressively
- **Writing aspirational rules** — "always use dependency injection" means nothing if your codebase doesn't. Only codify patterns that actually exist

### Discovery Process for a New Project

Ask Claude Code to draft rules for you:

> Explore this codebase and identify the distinct architectural layers, module boundaries, and coding conventions. For each boundary, draft a `.claude/rules/` file with appropriate path scoping. Keep each file under 20 lines — only rules that prevent mistakes.

Or do it manually:

1. **Map your boundaries** — what directories have different import rules or conventions?
2. **Write one rule file per boundary** — scoped with `paths:` to that directory
3. **Add a shared syntax file** — scoped to all code (`src/**`, `tests/**`)
4. **Extract narrow concerns** — if a rule only matters for 2-3 files, give it a tightly-scoped file
5. **Audit periodically** — delete rules that duplicate CLAUDE.md or no longer match the code

---

## agents/ — Specialist Subagents

Agents are read-only specialists that Claude invokes for deep analysis. They recommend — the main agent implements. Start with two essential agents.

**Agent metadata format** (YAML frontmatter):
```markdown
---
name: architecture-guardian
description: Use this agent when you need architectural review for Clean Architecture compliance
model: sonnet
allowed_tools: ["Read", "Glob", "Grep"]
---
```

**`agents/architecture-guardian.md`** — validates layer dependencies, repository patterns, transaction boundaries. Outputs: Approved / Approved with suggestions / Rejected with violations.

**`agents/test-pyramid-architect.md`** — designs test strategies, identifies correct test level per layer, recommends fixture patterns. Targets: 60% unit / 35% integration / 5% E2E.

**When to add more agents**: when you find yourself repeatedly giving the same specialized guidance (e.g., ORM optimization, frontend testing patterns, log analysis).

---

## skills/ — Reference Documents

Skills are embedded reference documents. Two types:

**Non-invocable** (background context, loaded automatically when relevant):
```markdown
---
name: api-contracts
description: REST API endpoint reference — routes, schemas, error codes
user-invocable: false
---
# API Contracts
[Condensed reference content]
```

**Invocable** (step-by-step workflows triggered by users):
```markdown
---
name: new-module
description: Step-by-step guide for adding a new module to the project
---
# Adding a New Module
## Step 1: Create domain entities...
## Step 2: Define repository protocol...
```

Use skills for: API contract references, design system tokens, database schema docs, repeatable multi-step workflows.
