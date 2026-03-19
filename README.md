# Dev Setup Guide: Python 3.14 + FastAPI + React with Claude Code

**A bootstrap guide for new full-stack projects — March 2026 best practices**

This guide covers everything you need to go from zero to a production-ready Python / FastAPI / React + Tailwind monorepo, optimized for development with Claude Code. It distills battle-tested patterns from real projects into copy-pasteable configurations and templates.

**Target audience**: Solo developers or small teams building full-stack web applications in 2026.

**How to use this guide**:
- **New project**: Work through phases in order. Each guide produces concrete deliverables (config files, `.claude/rules/`, `CLAUDE.md` sections) that your project owns permanently.
- **Existing project**: Audit checklist — compare each guide against your `.claude/` config and fill gaps.
- **After bootstrap**: The guide goes dormant. Revisit it when adding new capabilities (e.g., CLI to an API-only project), onboarding a team member, or auditing against updated best practices.

## Quick Start by Project Type

- **Full stack (API + Web UI)**: All phases in order
- **API + CLI (no web)**: Phases 1-4, Backend + CLI from Phase 5, Phase 6
- **API only (no interface)**: Phases 1-4, Backend from Phase 5, Phase 6
- **Frontend only**: Phases 1-3, Frontend from Phase 5, Phase 6

---

## Task Sequence

### Phase 1: Product & AI Setup (do this FIRST)

Phase 1 is the foundation everything else builds on. Skip it or rush it and every later decision lacks grounding — you end up building technically excellent features nobody needs.

- [ ] **[Product Context](product-context.md)** — Define the problem, users, personas, use cases, and scope `[S]`
- [ ] **[Claude Code Setup](claude-code-setup.md)** — CLAUDE.md authorship, hooks, permissions `[M]`
- [ ] **[Claude Code Rules](claude-code-rules.md)** — Path-scoped rules, specialist agents, reference skills `[M]`

### Phase 2: Planning Setup

- [ ] **[Backlog Planning](backlog-planning.md)** — Version-based roadmap, epic/story format, effort estimation `[S]`
- [ ] **[Spec Enforcement](spec-enforcement.md)** — Keep implementation grounded in user needs via self-checks, rules, and skills `[S]`

### Phase 3: Foundation

- [ ] **[Project Structure](project-structure.md)** — Git init, README, .gitignore, LICENSE, directory layout `[S]`

### Phase 4: Language & Tooling

- [ ] **[Python Tooling](python-tooling.md)** — uv, Ruff, BasedPyright, pytest, pre-commit, Python 3.14+ syntax patterns `[S]`
- [ ] **[Project Configuration](project-configuration.md)** — Typed settings, organized constants, structured logging `[S]`

### Phase 5: Stack (pick what applies)

#### Backend API *(skip if no backend API)*
- [ ] **[Domain Modeling](domain-modeling.md)** — Immutable entities, Command/Result pattern, structured failures `[M]`
- [ ] **[Use Case Architecture](use-case-architecture.md)** — Transaction ownership, repository access, connector protocols `[M]`
- [ ] **[Backend Patterns](backend-patterns.md)** — FastAPI routes, use case runner, error envelope, repositories, Unit of Work `[M]`
- [ ] **[External API Resilience](external-api-resilience.md)** — Error classification, retry policies, SSE progress *(skip if no external APIs)* `[M]`

> **Note**: Build the API before the frontend — Orval codegen needs the OpenAPI spec.

#### Frontend *(skip if no frontend)*
- [ ] **[React Tooling](react-tooling.md)** — Vite 8, TypeScript strict mode, Biome `[S]`
- [ ] **[Design Identity](react-design-identity.md)** — Visual identity, anti-AI-slop principles, design system rules `[M]`
- [ ] **[Frontend Architecture](react-frontend-architecture.md)** — IA, app shell, navigation, theme implementation, user state `[M]`
- [ ] **[Interaction Design](interaction-design-patterns.md)** — Progressive disclosure, self-evident UI, confirmation flows, state handling `[M]`
- [ ] **[React API & Testing](react-api-testing.md)** — Orval codegen, custom fetch, QueryClient, Vitest + MSW `[M]`

#### CLI *(skip if no CLI)*
- [ ] **[CLI Patterns](cli-patterns.md)** — Typer app structure, async bridge, Rich menus, progress displays `[M]`

### Phase 6: Quality

- [ ] **[Testing Strategy](testing-strategy.md)** — Test pyramid, placement by layer, fixtures, coverage targets `[S]`

---

## How This Guide Relates to Claude Code

This guide is a **bootstrap-time seed**, not a runtime dependency. Claude Code reads YOUR project's config, not this submodule.

**The lifecycle**:

1. **Read the guide** — understand patterns, copy configs, internalize conventions
2. **Generate your Claude Code config** — each guide's "Deliverables" become files your project owns: `CLAUDE.md`, `.claude/rules/`, `.claude/agents/`, `.claude/skills/`
3. **Guide goes quiet** — Claude Code runs from your project's config. It never reads `docs/dev-setup-guide/`.

The guide teaches "use batch-first repository patterns." Your `.claude/rules/infrastructure-patterns.md` says "All repositories use `find_by_ids()` returning `dict[int, Entity]`." Claude reads the rule, not the guide.

**Cross-pollination**: When a project discovers a better pattern, contribute it back to the guide (only when explicitly asked). Other projects periodically check the guide for new ideas and seed improvements into their own `.claude/rules/`, agents, and skills. The guide is the shared knowledge base; each project's Claude Code config is its working copy.

**Checking for updates**: When a project wants to see what's new in the guide, use git:

```bash
# See what changed since you last pulled the submodule
cd docs/dev-setup-guide && git log --oneline HEAD@{1}..HEAD

# See changes with context (useful for Claude Code agents)
cd docs/dev-setup-guide && git log --stat --no-merges -10

# Diff a specific file to understand what changed
cd docs/dev-setup-guide && git diff HEAD~5 -- python-tooling.md
```

Commit messages in this guide explain *why* patterns changed — Claude Code agents can read the git log to understand what's new and decide whether to update the project's own `.claude/` config accordingly.

---

## Essential Commands Cheat Sheet

```bash
# -- During Development (targeted) -------------------------
uv run pytest tests/path/to/test_file.py -x  # Run affected test file
uv run pytest --lf                            # Rerun failed tests only
pnpm --prefix web test src/path/Component.test.tsx  # Single frontend test

# -- Before Committing (full fast suite) --------------------
uv run pytest                           # All fast tests
pnpm --prefix web test                  # All frontend tests
uv run ruff check . --fix               # Lint + autofix
uv run ruff format .                    # Format
uv run vulture                          # Dead code check

# -- Full Verification (version bump / on request) ---------
uv run pytest -m ""                     # All tests (including slow)
uv run basedpyright src/                # Type check
pnpm --prefix web check && pnpm --prefix web build  # Frontend quality gates

# -- Frontend -----------------------------------------------
pnpm --prefix web dev                   # Vite dev server
pnpm --prefix web sync-api             # Orval codegen from openapi.json
```
