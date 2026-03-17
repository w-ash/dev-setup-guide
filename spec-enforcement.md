# Spec-Grounded Development

> **Scope**: Keep implementation grounded in user needs — read the spec before building, check it after, update when reality changes
> **Prerequisites**: [Product Context](product-context.md), [Claude Code Setup](claude-code-setup.md), [Claude Code Rules](claude-code-rules.md), [Backlog Planning](backlog-planning.md)
> **Deliverables**: CLAUDE.md planning self-check, `.claude/rules/spec-integrity.md`, optionally `.claude/skills/plan-feature/SKILL.md`
> **Estimated effort**: S

---

## What a Behavioral Spec Is

A behavioral spec defines **what the user does and why** — persona, workflow phases, user stories with Given/When/Then acceptance criteria, and user journeys. It sits between domain knowledge and implementation details, linking to both but owning neither.

It is NOT the backlog. The backlog says *how* and *when*. The spec says *what* and *why*. The spec accumulates over the project's life — a living document, not a PRD. For the structure and template, see [Product Context](product-context.md).

---

## Three Levels of Spec Connection

### Tier 1: Self-Check in CLAUDE.md

Add to your CLAUDE.md:

```markdown
## Planning Self-Check (before implementing a feature)

1. Which user stories in `docs/user-flows.md` does this serve?
2. What does the backlog spec in `docs/backlog/` say about implementation?
3. After implementing, do the user story's Given/When/Then criteria pass?
4. Did development reveal missing stories or stale criteria? Propose updates to `docs/user-flows.md`.
```

Mirrors the Testing Self-Check pattern — always present, no opt-in.

### Tier 2: Format Rule for Spec Files

Keeps spec format consistent as it evolves. Create `.claude/rules/spec-integrity.md`:

```markdown
---
paths:
  - "docs/backlog/**"
  - "docs/user-flows.md"
---
# Behavioral Spec Rules
- User stories use a consistent format (e.g., `**US-AREA-N**: As a [role], I want [goal]`) with Given/When/Then bullets
- Version annotations `(v0.X.x)` on stories match backlog version files
- New stories go in the relevant workflow section with a version annotation
- Preserve Given/When/Then format when updating acceptance criteria
- Prefer marking stories superseded (with a note) over deleting — preserves the decision trail
- Completed backlog tasks get `- [x]` and `Status: Completed (YYYY-MM-DD)`
```

This overlaps with `backlog-format.md` from [Backlog Planning](backlog-planning.md) — consolidate or split by concern.

### Tier 3: Planning Skill for Complex Features

For substantial features spanning multiple user stories. Create `.claude/skills/plan-feature/SKILL.md`:

```markdown
---
name: plan-feature
description: Ground a feature in user-flows.md and backlog before planning, validate after implementation, propose spec updates
argument-hint: version (e.g., "v0.7.0") or feature name (e.g., "spending trends")
---

# Plan Feature: $ARGUMENTS

Ground this feature in the existing behavioral spec before designing or implementing.

## Step 1: Load Context
- Read `docs/user-flows.md` — identify which user stories relate to `$ARGUMENTS`
- Read `docs/backlog/README.md` — find the version this maps to
- Read the relevant `docs/backlog/v0.X.x.md` — understand the implementation spec
- If `$ARGUMENTS` is a version number, use it directly. If it's a feature name, search the backlog version matrix.

## Step 2: Identify User Stories
List the specific user stories from user-flows.md that this feature satisfies. If no existing story covers this feature, note that — a new story will be needed in Step 5.

## Step 3: Cross-Reference
- Does the backlog spec cover everything the user stories require?
- Are there acceptance criteria in user-flows.md that the backlog doesn't address?
- Are there backlog tasks that go beyond what user-flows.md describes?
- Flag any misalignments.

## Step 4: Plan Implementation
With both documents as context, design the implementation.

## Step 5: Spec Evolution (if needed)
If planning revealed gaps, propose specific changes:
- **New user stories**: Draft in Given/When/Then format, placed in the relevant workflow section with version annotation
- **Updated criteria**: Show old vs. new acceptance criteria
- **Version annotation changes**: If a story's scope shifted to a different version

Present proposals inline — the user decides whether to apply them.

## Step 6: Post-Implementation Validation
After implementing, walk through each relevant user story's Given/When/Then criteria:
- For each criterion, confirm the implementation satisfies it (with file paths or test names as evidence)
- Flag any criteria not yet satisfied and note what remains
```

---

## When to Use Each Tier

| Spec surface area | Recommended tiers |
|---|---|
| No spec yet | None — write the spec first ([Product Context](product-context.md)) |
| < 10 user stories | Tier 1 only |
| 10-30 user stories | Tier 1 + Tier 2 |
| 30+ stories or multi-story features | All three tiers |

---

## Keeping the Spec Current

Specs evolve through use. Claude proposes additions or updates inline during planning; the user decides. When what you built doesn't match what the spec says the user needs, fix the code — the spec reflects intent. When the spec turns out to be wrong (missing edge case, flow that doesn't work), update the spec with a note. Either way, keep the two in sync.
