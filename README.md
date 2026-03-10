# Dev Setup Guide: Python 3.14 + FastAPI + React with Claude Code

**A bootstrap guide for new full-stack projects — March 2026 best practices**

This guide covers everything you need to go from zero to a production-ready Python / FastAPI / React + Tailwind monorepo, optimized for development with Claude Code. It distills battle-tested patterns from real projects into copy-pasteable configurations and templates.

**Target audience**: Solo developers or small teams building full-stack web applications in 2026.

**How to use this guide**:
- **New project**: Work through phases in order, starting with Product Context.
- **Existing project**: Use each guide as an audit checklist — check off what you have, create backlog stories for gaps. Start with Phase 1 to verify your product context and CLAUDE.md are solid.

---

## Task Sequence

### Phase 1: Product & AI Setup (do this FIRST)

Understand what you're building before configuring how to build it. Product context feeds directly into CLAUDE.md, the backlog, and design decisions.

- [ ] **[Product Context](product-context.md)** — Define the problem, users, success criteria, and scope `[XS]`
- [ ] **[Claude Code Setup](claude-code-setup.md)** — CLAUDE.md authorship, PostToolUse hooks, permissions `[M]`
- [ ] **[Claude Code Rules](claude-code-rules.md)** — Path-scoped rules, specialist agents, reference skills `[M]`

### Phase 2: Planning Setup

Set up the backlog before building anything. Use it to plan all remaining guide tasks — create a version file with sub-versions for Foundation, Language & Tooling, Stack, Quality, and CI/CD. This gives you (and Claude Code) a roadmap to work through systematically.

- [ ] **[Backlog Planning](backlog-planning.md)** — Version-based roadmap, epic/story format, effort estimation `[S]`

### Phase 3: Foundation

- [ ] **[Project Structure](project-structure.md)** — Git init, README, .gitignore, LICENSE, directory layout, bootstrap checklist `[S]`

### Phase 4: Language & Tooling

- [ ] **[Python Tooling](python-tooling.md)** — Poetry, Ruff, BasedPyright, pytest, pre-commit `[S]`
- [ ] **[Python 3.14+ Syntax](python-syntax.md)** — Modern patterns: 8 PEPs with DO/DON'T examples `[S]`
- [ ] **[Project Configuration](project-configuration.md)** — Typed settings, organized constants, structured logging `[S]`

### Phase 5: Stack (pick what applies)

#### Backend API
- [ ] **[FastAPI Backend](fastapi-backend.md)** — Clean Architecture, use case runner, routes, error envelope, OpenAPI `[M]`
- [ ] **[Domain Modeling](domain-modeling.md)** — Immutable entities, Command/Result pattern, structured failures `[M]`
- [ ] **[Database Patterns](database-patterns.md)** — Batch-first repos, Unit of Work, eager loading *(skip if no DB)* `[M]`
- [ ] **[External API Resilience](external-api-resilience.md)** — Error classification, retry policies, SSE progress `[M]`

#### Frontend
- [ ] **[React Tooling](react-tooling.md)** — Vite, TypeScript strict mode, Biome `[S]`
- [ ] **[Design Identity](react-design-identity.md)** — Visual identity, anti-AI-slop principles, design system rules `[M]`
- [ ] **[React API & Testing](react-api-testing.md)** — Orval codegen, custom fetch, QueryClient, Vitest + MSW `[M]`

#### CLI
- [ ] **[CLI with Typer](cli-typer.md)** — App structure, async bridge, command modules, error handling `[M]`
- [ ] **[CLI Rich Patterns](cli-rich-patterns.md)** — Interactive menus, progress displays, styled output `[S]`

### Phase 6: Quality

- [ ] **[Testing Strategy](testing-strategy.md)** — Test pyramid, placement by layer, fixtures, coverage targets `[S]`

### Phase 7: CI/CD

- [ ] **[CI/CD](ci-cd.md)** — GitHub Actions quality gates, Claude Code @mention workflow `[XS]`

---

## Essential Commands Cheat Sheet

```bash
# -- During Development (targeted) -------------------------
poetry run pytest tests/path/to/test_file.py -x  # Run affected test file
poetry run pytest --lf                            # Rerun failed tests only
pnpm --prefix web test src/path/Component.test.tsx  # Single frontend test

# -- Before Committing (full fast suite) --------------------
poetry run pytest                           # All fast tests
pnpm --prefix web test                      # All frontend tests
poetry run ruff check . --fix               # Lint + autofix
poetry run ruff format .                    # Format

# -- Full Verification (version bump / on request) ---------
poetry run pytest -m ""                     # All tests (including slow)
poetry run basedpyright src/                # Type check
pnpm --prefix web check && pnpm --prefix web build  # Frontend quality gates

# -- Frontend -----------------------------------------------
pnpm --prefix web dev                       # Vite dev server (port 5173)
pnpm --prefix web generate                  # Orval codegen from openapi.json

# -- CLI ----------------------------------------------------
poetry run my-app --help                    # Show CLI help
poetry run my-app command --verbose         # Run with debug logging
```
