# Backlog Planning

> **Scope**: Version-scoped roadmap files, epic/story format, effort estimation, and planning lifecycle
> **Prerequisites**: [Claude Code Setup](claude-code-setup.md)
> **Deliverables**: `docs/backlog/README.md` (master status), version files (`v0.X.x.md`), `unscheduled.md`
> **Estimated effort**: S

A structured approach to project planning using version-scoped roadmap files. Tracks epics, stories, dependencies, and effort — with a master README as the single source of truth for project status.

---

## Why This Structure

Most projects either have no roadmap (everything in someone's head) or use issue trackers that become graveyards of stale tickets. This file-based approach:

- **Lives in the repo** — version-controlled, reviewed in PRs, always up to date
- **Scales to one person or a small team** — no tool overhead
- **Provides transparency** — anyone (including Claude Code) can see the full roadmap
- **Preserves history** — completed work moves to an archive, not deleted

---

## Directory Structure

```
docs/
├── backlog/
│   ├── README.md              # Master roadmap — version matrix + status
│   ├── v0.1.x.md             # First initiative (epics + stories)
│   ├── v0.2.x.md             # Second initiative
│   ├── v0.3.x.md             # Third initiative
│   └── unscheduled.md        # Ideas without version assignment
├── completed/
│   ├── README.md              # Index of shipped versions + dates
│   ├── v0.1.x.md             # Archived after shipping
│   └── v0.2.x.md
└── ...other docs...
```

**Rules**:
- One file per minor version series (v0.1.x, v0.2.x, etc.)
- Each file contains multiple patch versions (v0.1.0, v0.1.1, v0.1.2)
- When all versions in a file ship, move the file to `completed/`
- `unscheduled.md` captures ideas that don't have a version yet

---

## Master README (backlog/README.md)

The README is a one-page view of the entire project direction.

### Template

```markdown
# Project Roadmap

## Version Matrix

| Version | Goal | Status | Effort |
|---|---|---|---|
| v0.1.0 | Core data model + CLI scaffold | Completed (2026-01-15) | M |
| v0.1.1 | Import from Service A | Completed (2026-01-22) | S |
| v0.2.0 | Web UI foundation | In Progress | L |
| v0.2.1 | Dashboard + list views | Not Started | M |
| v0.3.0 | Service B integration | Not Started | XL |

## Infrastructure Readiness

Shows which capabilities exist at each version:

| Capability | v0.1.x | v0.2.x | v0.3.x |
|---|---|---|---|
| CLI interface | ✅ | ✅ | ✅ |
| Web UI | — | ✅ | ✅ |
| Service A | ✅ | ✅ | ✅ |
| Service B | — | — | ✅ |
| Background jobs | — | — | ✅ |

## Key Technical Decisions

- **Database**: SQLite for v0.x, PostgreSQL for v1.0
- **API**: FastAPI with OpenAPI spec → Orval codegen
- **Frontend**: React 19 + Tanstack Query + Tailwind v4
```

### What Makes This Effective

- **Version matrix** is the first thing anyone sees — instant project orientation
- **Infrastructure readiness** shows capability progression across versions
- **Technical decisions** are recorded once, not scattered across issues

---

## Version File Format

Each version file follows a consistent structure.

### Template

```markdown
# v0.2.x: [Initiative Name]

For strategic overview, see the [planning overview](README.md).

---

### v0.2.0: [Feature Milestone] (Vertical Slice 1)
**Goal**: [Single, measurable outcome]
**Context**: [Why this now, what dependencies]
**What this unlocks**: [User impact]
**Key tech choices**: [Notable architectural decisions]

#### [Epic Name] Epic

- [ ] **[Story Title]**
    - Effort: [XS|S|M|L|XL|XXL]
    - Status: Not Started
    - What: [One-sentence feature description]
    - Why: [Business/architectural justification]
    - Dependencies: [List of prerequisite stories/versions]
    - Notes:
        - [Detailed implementation guidance]
        - [Database schema changes, API routes, component structure]
        - [Test expectations]

- [ ] **[Story Title]**
    - Effort: S
    - Status: Not Started
    - What: ...
    - Why: ...

---

### v0.2.1: [Next Feature Milestone] (Vertical Slice 2)
**Goal**: ...

#### [Epic Name] Epic

- [ ] **[Story Title]**
    ...
```

### Completed Stories

When a story ships, check the box and add a completion date:

```markdown
- [x] **Import Listening History**
    - Effort: M
    - Status: Completed (2026-02-10)
    - What: Import play history from Service A API
    - Why: Foundation for all recommendation features
    - Dependencies: v0.1.0 (data model)
    - Notes:
        - Paginated API fetch with rate limiting
        - Batch upsert into tracks table
        - Tests: happy path, empty response, rate limit retry
```

---

## Story Attributes

### Effort Estimates

Relative sizing — never time-based. Prevents false precision and focuses on complexity:

| Size | Meaning | Example |
|---|---|---|
| XS | Trivial change, <30 min of thought | Add a constant, fix a typo |
| S | Small, well-understood | New API endpoint with existing patterns |
| M | Medium, some design needed | New use case with domain logic |
| L | Large, crosses multiple layers | New entity + repository + API + UI |
| XL | Very large, significant design | New service integration end-to-end |
| XXL | Epic-level, break down further | Full new feature area |

### Status Values

| Status | Meaning |
|---|---|
| Not Started | Defined but no work begun |
| In Progress | Actively being worked on |
| Blocked | Waiting on a dependency |
| Completed (date) | Shipped with completion date |

### Dependencies

Explicit prerequisite tracking enables parallel planning:

```markdown
- Dependencies: v0.1.0 (data model), "Import History" story above
```

Reference version numbers for cross-version deps, story titles for within-version deps.

---

## Epic Grouping

Group related stories into epics for natural work batching:

```markdown
#### Data Import Epic

- [ ] **Service A History Import** (M)
- [ ] **Service B History Import** (L)
- [ ] **Deduplication Logic** (S)

#### Web UI Epic

- [ ] **History Dashboard** (M)
- [ ] **Import Progress Display** (S)
```

Epics don't need their own effort estimates — sum the stories. Epics don't have status — they're done when all stories are done.

---

## Lifecycle

```
unscheduled.md  →  backlog/v0.X.x.md  →  completed/v0.X.x.md
  (ideas)           (planned work)         (shipped archive)
```

1. **Capture** — new ideas go into `unscheduled.md` with minimal detail
2. **Plan** — when ready to commit, move to a version file with full story attributes
3. **Build** — update story status as work progresses
4. **Archive** — when all stories in a version file ship, move to `completed/`
5. **Update README** — keep the master version matrix current

---

## Unscheduled Ideas

`unscheduled.md` is a low-ceremony parking lot for ideas that don't have a version yet:

```markdown
# Unscheduled Backlog

Ideas and features without version assignment. Move to a version file when ready to commit.

## Data & Enrichment
- Genre tagging from MusicBrainz
- BPM detection via audio analysis
- Lyrics integration

## UI Improvements
- Dark/light theme toggle
- Keyboard shortcuts for common actions
- Mobile-responsive layout

## Integrations
- Apple Music connector
- YouTube Music connector
- Discogs collection import
```

Keep it simple — bullets with enough context to remember the idea. Full story attributes come when it moves to a version file.

---

## Bootstrapping from This Guide

When using the [Getting Started guide](README.md) to set up a new project, create your first version file by mapping guide phases to backlog stories. This ensures every task is tracked and nothing falls through the cracks.

### Example: `docs/backlog/v0.0.x.md`

```markdown
# v0.0.x: Project Bootstrap

From the [Getting Started guide](../../docs/dev-setup-guide/README.md).

---

### v0.0.1: Foundation & Tooling

#### Foundation Epic
- [ ] **Project Structure** — Effort: S — [guide](../../docs/dev-setup-guide/project-structure.md)
    - What: Create monorepo directory layout with Clean Architecture layers

#### Language & Tooling Epic
- [ ] **Python Tooling** — Effort: S — [guide](../../docs/dev-setup-guide/python-tooling.md)
    - What: Configure Poetry, Ruff, BasedPyright, pytest, pre-commit
- [ ] **Python 3.14+ Syntax** — Effort: S — [guide](../../docs/dev-setup-guide/python-syntax.md)
    - What: Review modern syntax patterns, add to CLAUDE.md
- [ ] **Project Configuration** — Effort: S — [guide](../../docs/dev-setup-guide/project-configuration.md)
    - What: Set up typed settings, constants module, structured logging

---

### v0.0.2: Stack Setup

#### Backend Epic
- [ ] **FastAPI Backend** — Effort: M — [guide](../../docs/dev-setup-guide/fastapi-backend.md)
    - What: Use case runner, thin routes, error envelope, OpenAPI spec
- [ ] **Domain Modeling** — Effort: M — [guide](../../docs/dev-setup-guide/domain-modeling.md)
    - What: Immutable entities, Command/Result pattern, structured failures
- [ ] **Database Patterns** — Effort: M — [guide](../../docs/dev-setup-guide/database-patterns.md)
    - What: Batch-first repos, Unit of Work, eager loading
    - Notes: Skip if project has no database
- [ ] **External API Resilience** — Effort: M — [guide](../../docs/dev-setup-guide/external-api-resilience.md)
    - What: Error classifier, retry policies, HTTP client factories
    - Notes: Skip if project makes no external API calls

#### Frontend Epic
- [ ] **React Tooling** — Effort: S — [guide](../../docs/dev-setup-guide/react-tooling.md)
    - What: Vite, TypeScript strict mode, Biome
- [ ] **Design Identity** — Effort: M — [guide](../../docs/dev-setup-guide/react-design-identity.md)
    - What: Visual identity, anti-AI-slop principles, design system rules
- [ ] **React API & Testing** — Effort: M — [guide](../../docs/dev-setup-guide/react-api-testing.md)
    - What: Orval codegen, custom fetch, QueryClient, Vitest + MSW

#### CLI Epic
- [ ] **CLI with Typer** — Effort: M — [guide](../../docs/dev-setup-guide/cli-typer.md)
- [ ] **CLI Rich Patterns** — Effort: S — [guide](../../docs/dev-setup-guide/cli-rich-patterns.md)

---

### v0.0.3: Quality & Process

#### Quality Epic
- [ ] **Testing Strategy** — Effort: S — [guide](../../docs/dev-setup-guide/testing-strategy.md)
    - What: Test pyramid, placement rules, factory fixtures, auto-markers

#### Process Epic
- [ ] **CI/CD** — Effort: XS — [guide](../../docs/dev-setup-guide/ci-cd.md)
    - What: GitHub Actions quality gates, Claude Code @mention workflow
```

**Adapt to your project**: Skip epics that don't apply (no CLI? drop the CLI epic). Skip individual stories (no database? drop Database Patterns). Version priorities should reflect your product goals from [Product Context](product-context.md) — the backlog serves the product, not the other way around.
