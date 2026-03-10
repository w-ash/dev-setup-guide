# Project Structure

> **Scope**: Git init, README, .gitignore, LICENSE, .editorconfig, monorepo directory layout, and bootstrap checklist
> **Prerequisites**: [Claude Code Setup](claude-code-setup.md), [Backlog Planning](backlog-planning.md)
> **Deliverables**: Repository initialized, all foundation files created, directory tree established
> **Estimated effort**: S

Everything a new project needs before any code gets written — the "day zero" basics that are invisible in established projects but critical when starting from scratch.

---

## Day Zero Checklist

Do these first. Every item here is trivial individually but painful to retrofit later.

### Part A: Repository & Project Basics

```bash
# 1. Create and initialize
mkdir my-project && cd my-project
git init
git branch -m main                    # Default branch = main
```

- [ ] **README.md** — project name, what it does, how to set up, how to run (see [template below](#readmemd-template))
- [ ] **LICENSE** — `MIT` for open source, proprietary header for private. [choosealicense.com](https://choosealicense.com) if unsure
- [ ] **.gitignore** — Python + Node + IDE + env (see [template below](#gitignore-template))
- [ ] **.editorconfig** — consistent formatting across all editors (see [template below](#editorconfig))
- [ ] **.python-version** — pin your Python version for pyenv/asdf/mise: `echo "3.14" > .python-version`
- [ ] **.env.example** — document all required env vars (committed to git)
- [ ] **.env** — actual secret values (gitignored, never committed)
- [ ] **`poetry init`** → creates `pyproject.toml`, then `poetry install` → creates `.venv/`
- [ ] **`__init__.py`** — create in every Python package dir AND every test dir (prevents module name collisions)
- [ ] **Initial commit** — clean baseline: `git add -A && git commit -m "chore: project scaffold"`

### Part B: Tooling & Stack Setup

These reference other guides — work through them in phase order:

- [ ] Configure Ruff, BasedPyright, pytest → [Python Tooling](python-tooling.md)
- [ ] Pre-commit hooks → `pre-commit install` → [Python Tooling](python-tooling.md)
- [ ] CLAUDE.md + `.claude/` directory → [Claude Code Setup](claude-code-setup.md)
- [ ] Path-scoped rules + agents → [Claude Code Rules](claude-code-rules.md)
- [ ] Scaffold frontend → `pnpm create vite web -- --template react-ts` → [React Tooling](react-tooling.md)
- [ ] Frontend tooling (biome, tsconfig, vite) → [React Tooling](react-tooling.md)
- [ ] API client + test infra → [React API & Testing](react-api-testing.md)
- [ ] Database + Alembic init (if applicable) → `alembic init alembic` → [Database Patterns](database-patterns.md)
- [ ] Health check endpoint — verify the full stack works end to end
- [ ] Run all quality gates — confirm a clean baseline

---

## README.md Template

Every project needs a README. Copy this and fill in the blanks:

````markdown
# Project Name

What it does, who it's for, and what problem it solves. Fill this in from your [Product Context](product-context.md) work.

## The Problem

[What users can't do today, or what's painful about how they do it]

## The Solution

[How this project addresses it — concrete, not abstract]

## Prerequisites

- Python 3.14+
- [Poetry](https://python-poetry.org/) for dependency management
- Node.js 22+ and [pnpm](https://pnpm.io/) (for web UI)

## Quick Start

```bash
# Clone and install
git clone https://github.com/you/my-project.git
cd my-project
poetry install

# Set up environment
cp .env.example .env
# Edit .env with your values

# Run database migrations (if applicable)
poetry run alembic upgrade head

# Start the API server
poetry run uvicorn src.interface.api.app:app --reload --port 8000  # Pick a unique port per project

# Start the frontend (separate terminal)
pnpm --prefix web install
pnpm --prefix web dev
```

## Development

```bash
# Tests
poetry run pytest                     # Fast test suite
pnpm --prefix web test                # Frontend tests

# Code quality
poetry run ruff check . --fix         # Lint + autofix
poetry run ruff format .              # Format
poetry run basedpyright src/          # Type check
pnpm --prefix web check              # Frontend lint
```

## Project Structure

```
src/
├── domain/          # Pure business logic
├── application/     # Use case orchestration
├── infrastructure/  # External adapters (DB, APIs)
├── interface/       # API routes, CLI commands
└── config/          # Settings, constants, logging
```

## License

[MIT](LICENSE) — or your license of choice.
````

**Adapt to your project**: Remove sections that don't apply (no frontend? drop the pnpm commands). Add sections you need (API docs, deployment, architecture diagrams).

---

## .gitignore Template

Comprehensive template for a Python + Node monorepo:

```gitignore
# ── Python ──────────────────────────────────
__pycache__/
*.py[cod]
*$py.class
*.so
.venv/
venv/
dist/
build/
*.egg-info/
*.egg
.pytest_cache/
.ruff_cache/
htmlcov/
.coverage
.coverage.*

# ── Node / Frontend ─────────────────────────
node_modules/
web/dist/
web/.vite/
*.tsbuildinfo

# ── Environment & Secrets ───────────────────
.env
.env.local
.env.*.local
*.pem
*.key

# ── IDE ─────────────────────────────────────
.vscode/
.idea/
*.swp
*.swo
*~
.project
.classpath

# ── OS ──────────────────────────────────────
.DS_Store
Thumbs.db
Desktop.ini

# ── Project-specific ────────────────────────
*.db
*.sqlite3
logs/
.claude/settings.local.json

# ── Alembic (generated) ────────────────────
# Keep alembic/ dir but ignore temp files
alembic/tmp/
```

**Key principle**: `.env.example` is committed (documents required vars). `.env` is gitignored (contains actual secrets). Never commit secrets.

---

## .editorconfig

Ensures consistent formatting regardless of which editor or IDE contributors use:

```ini
# .editorconfig
root = true

[*]
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
charset = utf-8

[*.{py,pyi}]
indent_style = space
indent_size = 4

[*.{ts,tsx,js,jsx,json,yaml,yml,css}]
indent_style = space
indent_size = 2

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
```

**Why bother**: Without this, one contributor uses tabs, another uses 2-space indents, a third uses 4-space — every PR has phantom whitespace changes. `.editorconfig` is respected by VS Code, JetBrains, Vim, Neovim, and most modern editors out of the box.

---

## .env.example

Document every environment variable your project needs. This file is **committed** — it's documentation, not secrets:

```bash
# .env.example — copy to .env and fill in values
# Required
DATABASE_URL=sqlite+aiosqlite:///data/app.db

# External API keys (get from service dashboards)
SPOTIFY_CLIENT_ID=
SPOTIFY_CLIENT_SECRET=
LASTFM_API_KEY=

# Optional (defaults shown)
LOG_LEVEL=INFO
API_PORT=8000
```

---

## Directory Layout

The recommended monorepo layout with Clean Architecture layers. Adjust to your project — not every project needs all layers.

```
my-project/
├── .claude/                          # Claude Code AI configuration
│   ├── settings.json                 # PostToolUse hooks (auto-format on edit)
│   ├── settings.local.json           # Permissions matrix (gitignored)
│   ├── agents/                       # Read-only specialist subagents
│   └── rules/                        # Path-based enforcement rules
├── .github/workflows/
│   └── claude.yml                    # Claude Code @mention trigger
├── src/                              # Python backend (Clean Architecture)
│   ├── domain/                       # Pure business logic, zero deps
│   │   ├── entities/                 # Immutable data models
│   │   ├── exceptions.py
│   │   └── repositories/            # Protocol interfaces only
│   ├── application/                  # Use case orchestration
│   │   ├── runner.py                 # execute_use_case() entry point
│   │   └── use_cases/
│   ├── infrastructure/               # External adapters & persistence
│   │   └── persistence/
│   │       └── repositories/         # Protocol implementations
│   ├── interface/                    # Presentation layer
│   │   └── api/
│   │       ├── app.py                # FastAPI application factory
│   │       ├── middleware.py         # Exception → error envelope
│   │       ├── routes/
│   │       └── schemas/              # Pydantic request/response
│   └── config/
│       ├── __init__.py
│       └── constants.py              # All project constants centralized
├── tests/                            # Mirror of src/ structure
│   ├── conftest.py                   # Root fixtures + auto-markers
│   ├── fixtures/                     # Factory helpers
│   │   ├── factories.py              # make_entity() builders
│   │   └── mocks.py                  # make_mock_uow()
│   ├── unit/                         # Fast, mocked (<100ms each)
│   │   ├── domain/
│   │   └── application/
│   └── integration/                  # Real deps (DB, HTTP)
│       ├── api/                      # Route handler tests
│       └── repositories/             # Real DB tests
├── web/                              # React frontend
│   ├── src/
│   │   ├── api/
│   │   │   ├── client.ts             # customFetch + ApiError
│   │   │   ├── query-client.ts       # QueryClient factory
│   │   │   └── generated/            # Orval codegen (never hand-edit)
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── pages/
│   │   ├── test/
│   │   │   ├── setup.ts              # MSW server bootstrap
│   │   │   └── test-utils.tsx        # renderWithProviders()
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── biome.json
│   ├── openapi.json                  # Copied from FastAPI backend
│   ├── orval.config.ts
│   ├── package.json
│   └── vite.config.ts
├── docs/                             # Project documentation
│   ├── backlog/                      # Planning & roadmap
│   │   ├── README.md                 # Master version matrix
│   │   ├── v0.1.x.md
│   │   └── unscheduled.md
│   └── completed/                    # Archive of shipped work
├── README.md                         # Project README (see template above)
├── LICENSE
├── CLAUDE.md                         # Project instructions for Claude Code
├── pyproject.toml                    # Python deps + ruff + pyright + pytest
├── .pre-commit-config.yaml
├── .editorconfig
├── .python-version
├── .env.example                      # Env var documentation (committed)
├── .env                              # Actual values (gitignored)
└── .gitignore
```

## Key Principles

- **Monorepo** with clear backend/frontend separation (`src/` and `web/`)
- **Clean Architecture** layers: `Interface → Application → Domain ← Infrastructure`
- **Tests mirror the source tree** under `tests/unit/` and `tests/integration/`
- **All Claude Code configuration** lives in `.claude/`
- **Every Python directory needs `__init__.py`** — including all `tests/` subdirs
- **Documentation** follows the backlog pattern (see [Backlog Planning](backlog-planning.md))
