# Interaction Design Patterns

> **Scope**: Progressive disclosure, self-evident interfaces, confirmation flows, status communication, state handling, information hierarchy
> **Prerequisites**: [Design Identity](react-design-identity.md), [Frontend Architecture](react-frontend-architecture.md)
> **Deliverables**: Consistent interaction patterns across all pages, encoded in `.claude/rules/`
> **Estimated effort**: M

How users discover features, understand state, and make decisions. The [design identity](react-design-identity.md) guide covers *what it looks like*. The [frontend architecture](react-frontend-architecture.md) covers *how it's structured*. This guide covers *how it behaves*.

---

## Self-Evident Interfaces

Every element should answer three questions at a glance: **What is this?** (clear labels), **What state is it in?** (status indicators), **What can I do?** (discoverable actions).

**Labels over icons alone.** Icons are ambiguous. Pair icons with text labels. Exception: universally understood icons (close X, back arrow, search magnifier) in space-constrained contexts.

**Status needs three channels.** Never color alone (accessibility) or icons alone (ambiguity). Combine at least two of: icon, color, text label. Best: green checkmark + green tint + "Synced 2h ago".

**Actions describe consequences.** "Export to Drive", "Import from CSV", "Remove connection". Never "Submit", "Confirm", "OK", "Yes".

**Plain language over jargon.** "Local -> Google Drive" not "push". "Since your last import" not "incremental delta".

---

## Progressive Disclosure

Show the essential information at rest. Let users expand for more.

| Scenario | Pattern |
|----------|---------|
| Page with 3+ sections of dense data | Primary info visible, details expandable |
| Settings with basic + advanced | Basic visible, advanced in collapsible section |
| Error messages | Summary visible, details expandable |
| Lists with metadata per item | Title + status visible, details on click |
| Form fields with help text | Label visible, description below or in tooltip |

**Layered visual weight, not hidden content.** The most effective progressive disclosure uses typography and color to create natural reading layers — no interaction required:

```
ITEM TITLE           <- Large, full contrast
Author Name          <- Normal, slightly muted
Category . 3:42      <- Small, muted, scannable
SKU: PRD-00012345    <- Mono, faint, reference
```

**Collapsible sections** for genuinely secondary content: use CSS `grid-template-rows: 0fr` -> `1fr` with `transition` and `overflow: hidden` for smooth height animation.

**Described options, not bare labels.** Radio buttons should include descriptions: "Only import records added since your last import" beats a bare "Incremental" label.

**Error detail expansion.** Summary at rest, "Show details" for technical context (HTTP status, request ID, retry timing).

### Anti-Patterns

- **Hiding primary actions behind hover.** Touch devices have no hover. Use muted-at-rest styling instead.
- **Accordion overload.** If every section is collapsed, users click 5 times to see the page.
- **"Read more" on short text.** If full text is under ~3 lines, just show it.

---

## Segmented Controls & Mode Toggles

A horizontal group of 2-5 mutually exclusive options that switch a view mode. Not tabs (page navigation), not radio buttons in a form (data collection).

The generic implementation (two buttons swapping `bg-muted` / `bg-accent`) looks like separate buttons with no visual connection. The purposeful implementation uses a **sliding indicator** — an absolutely positioned background that moves via CSS `transform` + `transition`. This is standard across major design systems (Mantine, Radix Themes, Apple, Wise).

**Implementation**: native `<input type="radio">` elements inside a `role="radiogroup"` container, with a sliding indicator div positioned using `useRef` measurements + `useEffect`. `ResizeObserver` handles responsive re-measurement. CSS `transition: transform 200ms ease-out` handles animation. Focus ring via `has-focus-visible:ring-2`.

| Situation | Use instead |
|---|---|
| More than 5 options | Select / dropdown |
| Independent on/off toggles | Switch components |
| Navigation between pages | Tabs or nav links |
| Action pairs that aren't mutual modes | Regular buttons |

---

## Confirmation & Destructive Actions

Not every action needs confirmation. Overuse trains reflexive "OK" clicking.

| Action type | Confirmation? |
|-------------|--------------|
| Reversible in-app action (edit, reorder) | No |
| Navigation from unsaved changes | Browser `beforeunload` |
| Deleting re-creatable data (cache, link) | Brief inline confirmation |
| Modifying external services (export, sync) | Dialog with preview |
| Deleting unrecoverable data | Dialog with explicit consequences |

**Dialog structure**: Title restates what will happen ("Export 12 items to Q1 Report"), preview shows real consequences (adds/removes/unchanged counts via a preview API endpoint), button labels match the action ("Export to Drive" not "Confirm"), default focus on Cancel, destructive buttons visually distinct.

If the preview fetch fails, degrade gracefully: show a warning but still allow the action.

---

## Status Communication

Build one reusable StatusIndicator component used everywhere:

| Variant | Icon | Color | Use for |
|---------|------|-------|---------|
| success | check circle | green | completed, connected, healthy |
| warning | alert triangle | amber | expiring, degraded, needs attention |
| error | x circle | red | failed, disconnected, broken |
| info | info circle | blue | in progress, syncing, processing |
| neutral | circle | gray | never run, not configured, unknown |

Include contextual detail text: "Connected 2h ago" not just "Connected".

### Operation Progress

For long-running operations, show: status icon, description of what's happening, metrics (current/total, rate, ETA in monospace), and a progress bar with status-appropriate color. Use `aria-live="polite"` for screen reader updates.

---

## State Handling

Every data-driven component has four states. Design all four before shipping.

**Loading** — skeleton loaders matching the final layout shape. Shimmer gradients, not `animate-pulse`.

**Empty** — explain why (no data yet), suggest next step, provide direct action (button/link). Use an icon for visual presence — text-only empty states look like bugs.

**Error** — show what went wrong and how to recover. Never "Something went wrong" alone. Place error boundaries around page content but outside navigation — users must always be able to navigate away. Use `resetKeys={[pathname]}` to auto-clear on route change.

**Success / Data** — the happy path. Handle edge cases: single item vs. many, long text overflow, missing optional fields. Design with real data.

---

## Information Hierarchy in Data-Dense Views

### The Weight System

| Tier | Treatment | Example |
|------|-----------|---------|
| **Primary** | Large font, full contrast | Item title, project name |
| **Secondary** | Normal font, full contrast | Author, description |
| **Tertiary** | Normal, muted color | Category, duration, date |
| **Reference** | Small/mono, faint | SKU, internal ID, timestamp |
| **Chrome** | Smallest, uppercase tracking, faint | Section headers, field labels |

Apply consistently across pages: if "Author" is secondary on the list, it's secondary on the detail page.

### Numeric Data

Right-align numeric columns. Use `font-variant-numeric: tabular-nums`. Monospace for identifiers and durations. Format consistently: "1,234" not "1234", "3:42" not "222s", "2h ago" not raw ISO timestamps.

---

## Action Discoverability

**Never hide actions behind hover** (`opacity-0 group-hover:opacity-100`). Use muted-at-rest styling that strengthens on hover. Primary actions prominent, secondary actions visible but recessed.

**Consistent vocabulary**: define action labels once (Add, Edit, Remove, Delete, Sync, Import) with consistent icons and use them everywhere. "Remove" dissociates (reversible); "Delete" destroys (may be irreversible).

**Direction labels**: "Local -> Google Drive" for export, "Google Drive -> Local" for import. Never "push"/"pull" — developer jargon.

---

## Encoding These Patterns in `.claude/rules/`

Add interaction rules to your `web-design-system.md` rule file:

```markdown
## Interaction Patterns
- Never color alone for status — combine icon + color + text
- Confirmation dialogs for external side effects only
- Restate what will happen in the title, not "Are you sure?"
- Action-specific button labels: "Export to Drive" not "Confirm"
- Never hide primary actions behind hover
- Button labels: verb + object ("Import from CSV")
- Same action = same pattern everywhere
```

---

## Audit Checklist

### Identity & Polish
- [ ] Custom page titles per route (not "Vite App")?
- [ ] Custom favicon (not the Vite logo)?
- [ ] Custom fonts loaded and rendering?
- [ ] No placeholder text in production?

### Self-Evident Design
- [ ] Every interactive element has a text label (not icon-only)?
- [ ] Status uses icon + color + text (never color alone)?
- [ ] Button labels describe what will happen (verb + object)?
- [ ] No domain jargon in user-facing text?

### Progressive Disclosure
- [ ] Primary information visible without interaction?
- [ ] Secondary information accessible but not overwhelming?
- [ ] Error messages show summary at rest, details on expand?

### Confirmation Flows
- [ ] Destructive actions have confirmation with consequences stated?
- [ ] Routine actions flow without confirmation?
- [ ] Default focus on Cancel for destructive dialogs?

### State Handling
- [ ] Loading skeletons match the final layout shape?
- [ ] Empty states explain + suggest next step + offer action?
- [ ] Error states show what went wrong + recovery path?
- [ ] Error boundaries keep navigation accessible?

### Information Hierarchy
- [ ] Visual weight tiers applied consistently across pages?
- [ ] Numeric data right-aligned with tabular-nums?
- [ ] Real data tested (long names, empty lists, single items)?

### Dark Mode
- [ ] All pages render correctly in both modes?
- [ ] Native elements (selects, scrollbars, forms) styled?
- [ ] Sufficient contrast in both modes?
- [ ] No hardcoded color values bypassing tokens?

### Action Discoverability
- [ ] No actions hidden behind hover?
- [ ] Same action uses same label + icon across all pages?

### Technical
- [ ] No console errors in normal operation?
- [ ] Semantic HTML (headings in order, landmarks)?
- [ ] Keyboard navigation works for primary flows?
