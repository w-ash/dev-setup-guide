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

Agents are markdown files in `.claude/agents/` that define specialist subagents. Claude invokes them for deep analysis — they recommend, the main agent implements. Each agent runs in its own context window with its own tool restrictions.

### Frontmatter Fields

```markdown
---
name: architecture-guardian
description: Use when you need architectural review for Clean Architecture compliance
model: sonnet
color: "#a855f7"
tools: Read, Glob, Grep
permissionMode: plan
maxTurns: 8
skills: subagent-guide
---
[Agent system prompt — role, competencies, response pattern, success criteria]
```

All available fields:

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Unique identifier (lowercase, hyphens) |
| `description` | Yes | When Claude should delegate — be specific so Claude knows when to invoke |
| `model` | No | `sonnet`, `opus`, `haiku`, or `inherit` (default: inherit from parent) |
| `color` | No | Status line color — hex string (e.g., `"#a855f7"`) |
| `tools` | No | Tool allowlist, comma-separated. Omit to inherit all tools |
| `disallowedTools` | No | Tool denylist (alternative to `tools`) |
| `maxTurns` | No | Upper bound on agent turns before stopping |
| `skills` | No | Skills to preload into agent context, comma-separated |
| `permissionMode` | No | `default`, `plan` (read-only), `acceptEdits`, `bypassPermissions` |
| `memory` | No | Persistent cross-session memory: `user`, `project`, or `local` |

### Key Fields Explained

**`tools`** — restricts what the agent can use. Read-only analysis agents should list `Read, Glob, Grep` only. Agents that run tests or CLI commands also need `Bash`. Agents that modify files need `Edit` or `Write` (rare for advisory agents).

**`maxTurns`** — prevents runaway agents. Set generous upper bounds: 8-10 for analysis agents, 12-15 for agents that run commands. Costs nothing during normal operation — the guardrail only fires when an agent loops.

**`skills`** — critical for agent effectiveness. Subagents do NOT inherit skills from the parent conversation — they start with a blank slate. Without `skills:`, an ORM specialist has to search the codebase for the database schema every invocation. With `skills: database-schema`, it starts with the schema already in context.

**`permissionMode: plan`** — system-enforced read-only. The agent cannot write files even if it tries. Ideal for analysis-only agents like architecture reviewers. Belt-and-suspenders with a restricted `tools` list.

**`memory: project`** — creates a persistent directory at `.claude/agent-memory/<name>/` that survives across conversations. The agent can write notes about patterns it discovers. Use for agents that learn project-specific knowledge over time (e.g., an ORM optimizer remembering which queries had N+1 issues). Creates a version-controllable `.claude/agent-memory/` directory. Auto-enables Read/Write/Edit for the memory directory only.

### Starter Agents

**`agents/architecture-guardian.md`** — validates layer dependencies, repository patterns, transaction boundaries. Read-only (`tools: Read, Glob, Grep`, `permissionMode: plan`). Outputs: Approved / Approved with suggestions / Rejected with violations.

**`agents/test-pyramid-architect.md`** — designs test strategies, identifies correct test level per layer, recommends fixture patterns. Needs Bash to run tests (`tools: Read, Glob, Grep, Bash`). Targets: 60% unit / 35% integration / 5% E2E.

**When to add more agents**: when you find yourself repeatedly giving the same specialized guidance (e.g., ORM optimization, frontend testing patterns, log analysis).

### Agent Prompt Structure

Agent prompts are the markdown body after the frontmatter. Aim for 100-300 lines:

1. **Role + scope** (~5 lines) — what the agent is, what it's an expert in
2. **Core competencies** (~30-80 lines) — codebase-specific knowledge, patterns to enforce
3. **Tool restrictions + why** (~10 lines) — what commands are allowed, what's off-limits
4. **Response pattern** (~10 lines) — analyze → recommend → explain rationale → anticipate issues
5. **Success criteria** (~10 lines) — what a good recommendation looks like

### DO / DON'T

- **DO** restrict tools to the minimum needed — less is safer and more focused
- **DO** set `maxTurns` on every agent — even generous limits prevent runaway cost
- **DO** preload skills the agent always needs — eliminates redundant file searches every session
- **DO** use `permissionMode: plan` for read-only analysis agents
- **DO** include `<example>` tags in `description` so Claude knows exactly when to delegate
- **DON'T** give agents Write/Edit tools unless they genuinely modify files (most advisory agents don't)
- **DON'T** add `memory` to every agent — only agents that accumulate project-specific patterns across sessions benefit (e.g., ORM optimizer). Analysis agents that derive advice from rules (architecture guardian) or from code they read fresh (test architect) don't need memory
- **DON'T** exceed 400 lines in agent prompts — be specific but not verbose
- **DON'T** create agents for one-off tasks — use the built-in Agent tool with a prompt instead

---

## skills/ — Reference Documents

Skills are markdown files in `.claude/skills/<name>/SKILL.md` that serve as embedded reference documents. They're the primary way to give agents (and the main conversation) domain knowledge.

### Two Types

**Non-invocable** (background context, loaded when relevant):
```markdown
---
name: database-schema
description: Database table definitions, relationships, indexes, and query patterns
user-invocable: false
---
# Database Schema
[Condensed reference content — tables, columns, relationships]
```

**Invocable** (step-by-step workflows triggered by `/skill-name`):
```markdown
---
name: new-module
description: Step-by-step guide for adding a new module to the project
---
# Adding a New Module
## Step 1: Create domain entities...
## Step 2: Define repository protocol...
```

### Skills + Agents Integration

The `skills:` field in agent frontmatter preloads skill content into the agent's context at startup. This is the primary way to give agents domain knowledge:

```markdown
# In .claude/agents/orm-optimizer.md
---
skills: database-schema, api-contracts
---
```

The agent starts every session with the database schema and API contracts already loaded — no file searching needed.

**Skill naming**: the `name:` field in skill frontmatter is the identifier used in agent `skills:` fields. Keep names short and descriptive.

### When to Create a Skill vs a Rule

| Use... | When... |
|--------|---------|
| **Rule** (`.claude/rules/`) | Guidance scoped to file paths — auto-loads when editing matching files |
| **Skill** (`.claude/skills/`) | Reference material not tied to file paths — loaded explicitly by agents or users |

Use skills for: API contract references, database schema docs, design system tokens, repeatable multi-step workflows, and any reference material agents need.
