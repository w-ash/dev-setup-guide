# Backlog Planning

> **Scope**: Version-scoped roadmap files, epic/story format, effort estimation, and planning lifecycle
> **Prerequisites**: [Claude Code Setup](claude-code-setup.md)
> **Deliverables**: `docs/backlog/README.md` (master status), version files (`v0.X.x.md`), `unscheduled.md`, `.claude/rules/backlog-format.md`
> **Estimated effort**: S

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

---

## Version File Format

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
    - What: [One-sentence feature description]
    - Why: [Business/architectural justification]
    - Dependencies: [List of prerequisite stories/versions]
    - Status: Not Started
    - Notes:
        - [Detailed implementation guidance]
        - [Database schema changes, API routes, component structure]
        - [Test expectations]

- [ ] **[Story Title]**
    - Effort: S
    - What: ...
    - Why: ...
    - Dependencies: None
    - Status: Not Started

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
- [x] **Import External Data**
    - Effort: M
    - What: Import data from Service A API
    - Why: Foundation for all recommendation features
    - Dependencies: v0.1.0 (data model)
    - Status: Completed (2026-02-10)
    - Notes:
        - Paginated API fetch with rate limiting
        - Batch upsert into tracks table
        - Tests: happy path, empty response, rate limit retry
```

---

## Story Attributes

### Effort Estimates

Relative sizing — never time-based:

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

## Writing Stories from Personas

Backlog stories should trace back to the personas and use cases you defined in [Product Context](product-context.md). This ensures every story serves a real user need, not just a technical capability.

**User story format** (use alongside the story template above):

> "As a [persona], I want [capability] so that [benefit]."

**Example**: If your primary persona is "The Weekly Curator" — someone who manages 15+ playlists and updates them weekly:

```markdown
- [ ] **Filter tracks by recent play count**
    - Effort: M
    - What: As the Weekly Curator, I want to filter liked tracks by play count in the last 30 days so I can build a "current obsessions" playlist
    - Why: Manual cross-referencing of play counts is the primary pain point
    - Dependencies: v0.1.0 (track import), v0.1.1 (play history import)
    - Status: Not Started
    - Acceptance criteria:
        - Filter returns only tracks with >= N plays in the last 30 days
        - Results update when play history is re-imported
        - Works with tracks from any connected service, not just one
```

The persona name in the story ("As the Weekly Curator") is a quick litmus test: if you can't name which persona wants this feature, question whether it belongs in the backlog. Features that don't serve a specific persona tend to be scope creep.

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
- Third-party metadata enrichment
- Automated categorization
- Full-text search integration

## UI Improvements
- Dark/light theme toggle
- Keyboard shortcuts for common actions
- Mobile-responsive layout

## Integrations
- Service C connector
- Service D connector
- Legacy data import (CSV/JSON)
```

Keep it simple — bullets with enough context to remember the idea. Full story attributes come when it moves to a version file.

---

## Bootstrapping from This Guide

When using the [Getting Started guide](README.md) to set up a new project, create your first version file by mapping guide phases to backlog stories.

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
- [ ] **Python Tooling & Syntax** — Effort: S — [guide](../../docs/dev-setup-guide/python-tooling.md)
    - What: Configure uv, Ruff, BasedPyright, pytest, pre-commit; review Python 3.14+ syntax patterns
- [ ] **Project Configuration** — Effort: S — [guide](../../docs/dev-setup-guide/project-configuration.md)
    - What: Set up typed settings, constants module, structured logging

---

### v0.0.2: Stack Setup

#### Backend Epic
- [ ] **Domain Modeling** — Effort: M — [guide](../../docs/dev-setup-guide/domain-modeling.md)
    - What: Immutable entities, Command/Result pattern, structured failures
- [ ] **Backend Patterns** — Effort: M — [guide](../../docs/dev-setup-guide/backend-patterns.md)
    - What: Use case runner, thin routes, error envelope, repositories, Unit of Work
    - Notes: Database sections optional if project has no DB
- [ ] **External API Resilience** — Effort: M — [guide](../../docs/dev-setup-guide/external-api-resilience.md)
    - What: Error classifier, retry policies, HTTP client factories
    - Notes: Skip if project makes no external API calls

#### Frontend Epic
- [ ] **React Tooling** — Effort: S — [guide](../../docs/dev-setup-guide/react-tooling.md)
    - What: Vite 8, TypeScript strict mode, Biome
- [ ] **Design Identity** — Effort: M — [guide](../../docs/dev-setup-guide/react-design-identity.md)
    - What: Visual identity, anti-AI-slop principles, design system rules
- [ ] **Frontend Architecture** — Effort: M — [guide](../../docs/dev-setup-guide/react-frontend-architecture.md)
    - What: IA, page inventory, app shell, navigation, theme implementation, user state
- [ ] **Interaction Design** — Effort: M — [guide](../../docs/dev-setup-guide/interaction-design-patterns.md)
    - What: Progressive disclosure, confirmation flows, state handling, UI audit checklist
- [ ] **React API & Testing** — Effort: M — [guide](../../docs/dev-setup-guide/react-api-testing.md)
    - What: Orval codegen, custom fetch, QueryClient, Vitest + MSW

#### CLI Epic
- [ ] **CLI Patterns** — Effort: M — [guide](../../docs/dev-setup-guide/cli-patterns.md)
    - What: Typer app structure, async bridge, Rich menus, progress displays

---

### v0.0.3: Quality & Process

#### Quality Epic
- [ ] **Testing Strategy** — Effort: S — [guide](../../docs/dev-setup-guide/testing-strategy.md)
    - What: Test pyramid, placement rules, factory fixtures, auto-markers

```

**Adapt to your project**: Skip epics that don't apply (no CLI? drop the CLI epic). Skip individual stories (no database? drop Database Patterns). Version priorities should reflect your product goals from [Product Context](product-context.md).

---

## Encoding in `.claude/rules/`

Create `.claude/rules/backlog-format.md` so Claude Code enforces consistent backlog structure across all projects using this guide:

```markdown
---
paths:
  - "docs/backlog/**"
  - "docs/completed/**"
---
# Backlog Format Rules

## Directory Structure
- `docs/backlog/README.md` — master roadmap: version matrix, infrastructure readiness, tech decisions
- `docs/backlog/v0.X.x.md` — one file per minor version series (may contain multiple patch versions)
- `docs/backlog/unscheduled.md` — ideas without version assignment
- `docs/completed/` — archived version files + index after all stories ship

## Master README Format
- **Current Version** and **Current Initiative** at the top
- Version matrix table: Version | Goal | Status | Details (with link to section)
- Status indicators: ✅ Completed | 🔨 In Progress | 🔜 Not Started
- Infrastructure Readiness Matrix showing capabilities across versions
- Technology Decision Records for key architecture choices
- Reference section: effort estimate definitions + status options

## Version File Format
Header: `# v0.X.x: [Initiative Name]`
Sub-versions: `### v0.X.Y: [Feature Milestone]`
Each sub-version includes: **Goal**, **Context**, **What this unlocks**, **Key tech choices**

### Story Format (mandatory fields, this exact order)
- [ ] **Story Title**
    - Effort: XS|S|M|L|XL|XXL
    - What: One-sentence description
    - Why: Business/architectural justification
    - Dependencies: version refs or story titles (None if none)
    - Status: Not Started | In Progress | Blocked | Completed (YYYY-MM-DD)
    - Notes:
        - Implementation guidance, schema changes, test expectations
        - Sub-bullets for detailed technical notes

### Epic Grouping
`#### [Epic Name] Epic` — groups related stories. Epics have no status or effort of their own.

## Unscheduled Format
Category sections (`## Category Name`), each item: `**Title** (effort)` with description.
Detailed planning notes are OK for items with significant research.

## Lifecycle Operations

### Completing a story
1. Check the box: `- [x]`
2. Update Status to `Completed (YYYY-MM-DD)` with actual date
3. Update `docs/backlog/README.md` version matrix status

### Completing a version file
1. Move from `docs/backlog/` to `docs/completed/`
2. Update `docs/completed/README.md` index
3. Update `docs/backlog/README.md` matrix to ✅ Completed

## Effort Sizing (relative, NEVER time-based)
- XS: trivial, isolated | S: small, well-understood, 1-2 areas
- M: cross-module, small unknowns | L: architectural impact, >=3 subsystems
- XL: high unknowns, cross-team | XXL: high risk, break down further

## Conventions
- Always convert relative dates to absolute (e.g., "Thursday" → "2026-03-20")
- New ideas → unscheduled.md first, version file when committed
- Each version delivers a vertical slice — backend + frontend together — so every increment is testable end-to-end
```

This rule ensures every project using this guide produces identically structured backlog files — making cross-project planning, status checks, and agent-driven updates consistent.
