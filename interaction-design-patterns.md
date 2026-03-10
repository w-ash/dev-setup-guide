# Interaction Design Patterns

> **Scope**: Progressive disclosure, self-evident interfaces, confirmation flows, status communication, state handling, information hierarchy
> **Prerequisites**: [Design Identity](react-design-identity.md), [Frontend Architecture](react-frontend-architecture.md)
> **Deliverables**: Consistent interaction patterns across all pages, encoded in `.claude/rules/`
> **Estimated effort**: M

How users discover features, understand state, and make decisions. The [design identity](react-design-identity.md) guide covers *what it looks like*. The [frontend architecture](react-frontend-architecture.md) covers *how it's structured*. This guide covers *how it behaves*.

---

## Self-Evident Interfaces

The goal: users should understand what they're looking at and what they can do without reading documentation. Every element should answer three questions at a glance:

1. **What is this?** — clear labels, recognizable patterns
2. **What state is it in?** — status indicators, visual feedback
3. **What can I do?** — discoverable actions, obvious affordances

If users need a tutorial to use your app, the interface failed. If they need to hover over things to find out what they are, the interface failed. Design for scanning, not reading.

### Principles

**Labels over icons alone.** Icons are ambiguous — a gear could mean settings, configuration, or preferences. Pair icons with text labels. The only exception: universally understood icons (close X, back arrow, search magnifier) in contexts where space is genuinely constrained.

**Status needs three channels.** Never communicate status with color alone (accessibility) or icons alone (ambiguity). Combine at least two of: icon, color, text label. Best practice is all three: a green checkmark icon + green tint + "Synced 2h ago".

**Actions describe consequences.** Button labels should be verb + object: "Export to Drive", "Import from CSV", "Remove connection". Never "Submit", "Confirm", "OK", "Yes". The user should know what will happen before they click.

**Plain language over jargon.** "Local → Google Drive" not "push". "Since your last import" not "incremental delta". If your UI uses terminology that requires domain knowledge to parse, rewrite it.

---

## Progressive Disclosure

Show the essential information at rest. Let users expand for more. This manages cognitive load without hiding important features.

### When to Use

| Scenario | Pattern |
|----------|---------|
| Page with 3+ sections of dense data | Primary info visible, details expandable |
| Settings with basic + advanced options | Basic visible, advanced in collapsible section |
| Error messages | Summary visible, stack trace / details expandable |
| Lists with metadata per item | Title + status visible, details on click/expand |
| Form fields with help text | Label visible, description below or in tooltip |

### Patterns

**Layered visual weight, not hidden content.** The most effective progressive disclosure doesn't use expand/collapse at all — it uses typography, color, and spacing to create natural reading layers:

```
ITEM TITLE           ← Large, white, immediate attention
Author Name          ← Normal, slightly muted
Category · 3:42 · 2024  ← Small, muted, scannable metadata
SKU: PRD-00012345    ← Mono, faint, reference data
```

Users naturally scan the top layer. They look deeper only when they need detail. No interaction required.

**Collapsible sections for genuinely secondary content.** Use CSS grid transitions for smooth height animation — `grid-template-rows` from `0fr` to `1fr` with `overflow: hidden` on the child. This avoids the jank of `height: auto` transitions:

```css
.collapsible {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 200ms ease-out;
}
.collapsible.open {
  grid-template-rows: 1fr;
}
.collapsible > div {
  overflow: hidden;
}
```

**Described options, not bare labels.** Radio buttons and selectors should include a short description — not just a label. Users shouldn't have to guess what "Incremental" means when "Only import records added since your last import" is clearer:

```
○ Full Import
  Replace all data with a fresh import from the service

● Incremental Import
  Only import records added since your last import
```

**Error detail expansion.** Show a human-readable summary at rest. Offer "Show details" for the technical context:

```
✕ Export failed: API rate limit exceeded
  [Show details ▾]

  → HTTP 429 at PUT /v1/folders/abc123/items
  → Retry-After: 30s
  → Request ID: req_abc123
```

### Anti-Patterns

- **Hiding primary actions behind hover.** `opacity-0 group-hover:opacity-100` is forbidden for important actions. Use muted-at-rest styling instead (visible but low-contrast, strengthens on hover). Touch devices have no hover state.
- **Accordion overload.** If every section is collapsed by default, users have to click 5 times to see the page. Collapse only genuinely secondary content.
- **"Read more" on short text.** If the full text is under ~3 lines, just show it. Truncation has a cost — the user has to decide whether the hidden content matters.

---

## Confirmation & Destructive Actions

Not every action needs a confirmation dialog. Overusing them trains users to click "OK" reflexively — which means the one time it matters, they'll click through without reading.

### When to Confirm

| Action type | Confirmation needed? |
|-------------|---------------------|
| Reversible in-app action (edit, reorder) | No |
| Navigation away from unsaved changes | Browser `beforeunload` prompt |
| Deleting data that can be re-created (cache, link) | Brief inline confirmation |
| Modifying external services (export, sync) | Dialog with preview |
| Deleting data that cannot be recovered | Dialog with explicit consequences |

### Confirmation Dialog Structure

```
┌─────────────────────────────────────────┐
│  Title: What will happen                │
│                                         │
│  Description: Why this matters          │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  Preview: What will change      │    │
│  │  +12 items added                │    │
│  │  -3 items removed               │    │
│  │  42 unchanged                   │    │
│  └─────────────────────────────────┘    │
│                                         │
│           [Cancel]  [Export to Drive]    │
└─────────────────────────────────────────┘
```

Rules:
- **Title restates what will happen** — "Export 12 items to Q1 Report on Google Drive", not "Are you sure?"
- **Preview shows consequences** — fetch real data if possible (a preview API endpoint). Show adds, removes, unchanged counts.
- **Button labels match the action** — "Remove connection", "Export to Drive", "Delete project". Never "Yes" / "OK" / "Confirm".
- **Default focus on the safe option** (Cancel). Keyboard Enter should not trigger the destructive action.
- **Destructive buttons are visually distinct** — red tint, separated from Cancel by gap or alignment.
- **Allow parameter changes mid-dialog** — if the user can toggle options (e.g., import vs. export direction) inside the dialog and see an updated preview, they make better decisions without restarting the flow.

### The Preview-Before-Commit Pattern

For operations with side effects (especially external service modifications), add a read-only preview endpoint:

```
GET /projects/{id}/connections/{conn_id}/sync/preview?direction=export

→ { items_to_add: 12, items_to_remove: 3, items_unchanged: 42 }
```

The confirmation dialog fetches this on open, showing real numbers instead of "this action cannot be undone." Users make informed decisions. The preview use case reuses existing domain logic in a read-only context — no new business logic needed.

If the preview fetch fails, degrade gracefully: show a warning ("Preview unavailable") but still allow the action with a manual confirmation.

---

## Status Communication

### The StatusIndicator Pattern

Build one reusable status component used everywhere — connection status, operation state, data freshness, health checks. Consistency means users learn the pattern once.

```
[icon] [color] [label]     [optional detail]
  ✓     green   Connected   Last synced 2h ago
  ⚠     yellow  Warning     Token expires in 2d
  ✕     red     Failed      Rate limit exceeded
  ●     gray    Pending     Waiting for auth
```

Design rules:
- **Never color alone** — colorblind users can't distinguish red/green dots. Always pair with icon + text.
- **Contextual detail text** — "Connected 2h ago" not just "Connected". "High confidence (92%)" not just "92%". The label provides meaning; the detail provides context.
- **Consistent mapping** — define your status vocabulary once and use it everywhere:

| Variant | Icon | Color | Use for |
|---------|------|-------|---------|
| success | check circle | green | completed, connected, healthy, high confidence |
| warning | alert triangle | amber | expiring, degraded, needs attention |
| error | x circle | red | failed, disconnected, broken |
| info | info circle | blue | in progress, syncing, processing |
| neutral | circle | gray | never run, not configured, unknown |

### Operation Progress

For long-running operations, combine multiple signals so progress feels tangible:

```
[icon] Importing records               1,234 / 5,678   42/s   ~1m 23s
       ████████████░░░░░░░░░░░  22%
```

Required elements:
- **Status icon** matching the StatusIndicator vocabulary
- **Description** of what's happening (not just "Processing...")
- **Metrics**: current/total, rate, ETA — in monospace for tabular alignment
- **Progress bar** with status-appropriate color + animation

Use `aria-live="polite"` on the container so screen readers announce progress updates without interrupting.

---

## State Handling

Every data-driven component has at least four states. Design all four before shipping any of them.

### The Four States

**Loading** — show skeleton loaders that match the final layout shape. Users perceive structured placeholders as faster than spinners. Use shimmer gradients, not `animate-pulse` (which reads as broken, not loading).

```tsx
// Skeleton matches the real layout — 4 stat cards in a grid
{isLoading && (
  <div className="grid grid-cols-4 gap-4">
    {Array.from({ length: 4 }, (_, i) => (
      <div key={i} className="h-24 rounded-lg bg-surface-elevated animate-shimmer" />
    ))}
  </div>
)}
```

**Empty** — explain why there's no data and suggest the next step. Every empty state is an onboarding opportunity:

```
┌────────────────────────────────────────┐
│         [icon: inbox]                  │
│                                        │
│       No projects yet                  │
│                                        │
│   Create your first project or import  │
│   one from an existing service         │
│                                        │
│        [Create project →]              │
└────────────────────────────────────────┘
```

Rules:
- Explain what would be here ("No projects yet")
- Explain how to populate it ("Create your first project or import one")
- Provide a direct action (link/button to the next step)
- Use an icon for visual presence — text-only empty states look like bugs

**Error** — show what went wrong and how to recover. Never "Something went wrong" without context:

```
✕ Failed to load project data
  API returned 401 Unauthorized — your token may have expired.
  [Reconnect service]  [Retry]
```

Place error boundaries around page content but outside navigation — users should always be able to navigate away from an error. Use `resetKeys={[pathname]}` to auto-clear errors on route change.

**Success / Data** — the happy path. But also handle edge cases: single item vs. many, long text overflow, missing optional fields. Design with real data, not "Item 1", "Item 2".

---

## Information Hierarchy in Data-Dense Views

Data-heavy pages (detail views, dashboards, settings) need clear visual hierarchy so users can scan without reading everything.

### The Weight System

Assign every piece of information a visual weight tier:

| Tier | Treatment | Example |
|------|-----------|---------|
| **Primary** | Large font, full contrast, prominent position | Item title, project name |
| **Secondary** | Normal font, full contrast | Author, description |
| **Tertiary** | Normal font, muted color | Category, duration, date |
| **Reference** | Small/mono font, faint color | SKU, internal ID, timestamp |
| **Chrome** | Smallest, uppercase tracking, faint | Section headers, field labels |

Apply consistently: if "Author" is secondary on the list page, it's secondary on the detail page too.

### Multi-Section Layout

For detail pages with multiple data sections, use a consistent card pattern:

```
┌─ SECTION TITLE (chrome tier) ───────────────┐
│                                              │
│  Primary datum                               │
│  Secondary datum                             │
│  Tertiary · tertiary · tertiary              │
│                                              │
└──────────────────────────────────────────────┘
```

Rules:
- Section titles use small caps / uppercase tracking to distinguish from content
- Use a left-accent border or subtle background shift to group sections — not identical card borders everywhere
- Within each section, maintain the weight hierarchy
- Grid layout (2-col or 3-col) for sections at the same importance level

### Numeric Data

For tables and aligned statistics:
- Right-align numeric columns
- Use `font-variant-numeric: tabular-nums` for aligned digits
- Use monospace font for identifiers, codes, durations
- Format consistently: "1,234" not "1234", "3:42" not "222s", "2h ago" not "2024-03-10T14:23:00Z"

---

## Action Discoverability

Users should never wonder "how do I...?" The interface should make available actions obvious.

### Always-Visible, Variable-Weight Actions

Never hide actions behind hover (`opacity-0 group-hover:opacity-100`). Instead, use **muted-at-rest** styling:

```tsx
// ✓ Always visible, strengthens on hover
<button className="text-text-faint hover:text-text transition-colors">
  <Pencil size={14} />
</button>

// ✕ Hidden until hover — invisible on touch, undiscoverable
<button className="opacity-0 group-hover:opacity-100">
  <Pencil size={14} />
</button>
```

Primary actions (the main thing a user came to do) should be prominent. Secondary actions (edit, delete, reorder) should be visible but recessed.

### Consistent Action Vocabulary

Define your action vocabulary once and use the same word + icon everywhere:

| Action | Label pattern | Icon |
|--------|--------------|------|
| Create | "Add {thing}" or "Create {thing}" | Plus |
| Edit | "Edit {thing}" or inline pencil | Pencil |
| Delete | "Remove {thing}" or "Delete {thing}" | Trash |
| Sync | "Sync to {service}" or "Sync from {service}" | RefreshCw |
| Link | "Link {thing}" | Link |
| Import | "Import from {service}" | Download |

"Remove" and "Delete" are different: remove dissociates (reversible), delete destroys (may be irreversible). Use the right word.

### Direction Labels

For bidirectional operations (sync, import/export), use arrow notation that reads naturally:

```
Local → Google Drive     (export: your data goes to the service)
Google Drive → Local     (import: service data comes to you)
```

Never use "push" / "pull" as user-facing labels — they're developer jargon. The arrow makes direction self-evident.

---

## Encoding These Patterns in `.claude/rules/`

Add interaction rules to your `web-design-system.md` rule file alongside visual rules:

```markdown
## Self-Explanatory Interface Principles

### Status Indicators
- Never color alone — combine icon + color + text label
- Contextual detail text: "Connected 2h ago" not just "Connected"
- Reuse StatusIndicator component for all status display

### Confirmation & Destructive Actions
- Confirmation dialogs for external side effects only — don't cry wolf
- Restate what will happen in the title, not "Are you sure?"
- Action-specific button labels: "Export to Drive" not "Confirm"
- Default focus on Cancel

### Progressive Disclosure
- Basics visible, details expandable
- Described options — radio buttons include descriptions, not just labels
- Error details expandable — summary at rest, full error on expand

### Action Discoverability
- Never hide primary actions behind hover
- Button labels: verb + object ("Import from CSV")
- Same action = same pattern everywhere
```

This ensures every AI-generated component follows your interaction patterns, not just your visual identity.

---

## Audit Checklist

### Self-Evident Design
- [ ] Every interactive element has a text label (not icon-only for primary actions)?
- [ ] Status uses icon + color + text (never color alone)?
- [ ] Button labels describe what will happen (verb + object)?
- [ ] No domain jargon in user-facing text?

### Progressive Disclosure
- [ ] Primary information visible without interaction?
- [ ] Secondary information accessible but not overwhelming?
- [ ] Form options include descriptions, not just labels?
- [ ] Error messages show summary at rest, details on expand?

### Confirmation Flows
- [ ] Destructive actions have confirmation with consequences stated?
- [ ] Routine actions flow without confirmation?
- [ ] Confirmation dialogs show preview data where possible?
- [ ] Default focus on Cancel for destructive dialogs?

### State Handling
- [ ] Loading skeletons match the final layout shape?
- [ ] Empty states explain + suggest next step + offer action?
- [ ] Error states show what went wrong + recovery path?
- [ ] Error boundaries keep navigation accessible?

### Information Hierarchy
- [ ] Visual weight tiers applied consistently across pages?
- [ ] Numeric data right-aligned with tabular-nums?
- [ ] Section headers distinguished from content (small caps, faint)?
- [ ] Real data tested (long names, empty lists, single items)?

### Action Discoverability
- [ ] No actions hidden behind hover (opacity-0)?
- [ ] Primary actions prominent, secondary actions muted but visible?
- [ ] Same action uses same label + icon across all pages?
- [ ] Direction labels use arrows, not push/pull jargon?
