# Visual Identity & Anti-AI-Slop Design

> **Scope**: How to develop a unique visual identity for your project and avoid AI-generated design patterns
> **Prerequisites**: [Product Context](product-context.md), [React Tooling](react-tooling.md)
> **Deliverables**: Design identity answers documented, `.claude/rules/web-design-system.md` written with your project's specific aesthetic
> **Estimated effort**: M

---

## The Sameness Problem

Every major LLM trains on the same Bootstrap layouts, Tailwind templates, and UI kit screenshots. Without specific guidance, they converge on the **statistical average of "website"** — indigo gradients, uniform card grids, identical spacing, glassmorphism everywhere. The result: every AI-assisted project looks identical.

The fix: give AI tools a strong, specific identity to enforce. Ground every design decision in your users' needs and your app's personality. Users should feel like someone thoughtful built this for them — not like they're using a template.

---

## Step 1: Define Your Design Identity

Answer three questions before touching CSS. These answers drive every visual decision.

Your answers here should flow directly from your [Product Context](product-context.md). If you defined "power users who value density and control" as your primary persona, that eliminates generous whitespace and playful colors before you've opened a CSS file. The persona constrains the aesthetic — let it.

**Who is your audience?**
- Power users who value density and keyboard shortcuts? (Linear, Raycast)
- Creative professionals who expect expressive, sensory interfaces? (Figma, Ableton)
- General consumers who need clarity and low cognitive load? (Notion, Stripe)
- Developers who appreciate monospace aesthetics and raw data? (GitHub, terminal UIs)

**What is the app trying to accomplish?**
- A data tool should prioritize scannability and information hierarchy
- A media app should give content room to breathe
- A creative tool should make workflows feel like composing, not filling out forms
- A productivity app should minimize friction and reduce choices

**How should users feel when using it?**
- Powerful and in control? → Dense layouts, keyboard-first, monochrome with meaningful accents
- Calm and focused? → Generous whitespace, muted palette, subtle transitions
- Immersed and inspired? → Rich media, dark themes, atmospheric textures
- Playful and exploratory? → Bold colors, asymmetric layouts, personality in micro-copy

Write these answers down. They become your design brief — the "why" behind every visual choice.

---

## Step 2: Build a Visual System

A visual system is a set of constrained, intentional tokens — not a component library. Components come from the system; the system comes from your identity.

**Define these before building components:**

| Token | What to decide | Why it matters |
|---|---|---|
| **Type scale** | 2-3 font families, a size scale, hierarchy rules | Typography IS identity — it's the single strongest signal of whether a human made design choices. See the font selection callout below. |
| **Color palette** | A constrained palette (monochrome + 1-2 accents) | Not the AI default indigo. Choose colors that reflect your app's personality. |
| **Spacing scale** | A defined rhythm (4px, 8px, 12px, 16px, 24px, 32px, 48px) | Vary the rhythm. Uniform spacing between everything is a dead giveaway. |
| **Depth system** | How to distinguish surface levels (shadow, border, background) | Not every card at the same elevation. Create hierarchy through depth. |
| **Motion language** | When things move, how fast, what easing | Animations guide attention, not impress. 150ms for interactions, 300ms for layout. |

**The defensibility test**: for every element on screen, you should be able to explain why it exists and why it's positioned where it is. If you can't, cut it.

### Font Selection

**Never use Inter.** It's the default that every AI tool and Tailwind template reaches for — the typographic equivalent of `bg-indigo-500`. If your app uses Inter, it signals that nobody made a design choice.

Font choice should match the character of your system:

- Music/media tool → geometric sans + editorial serif (Space Grotesk + Newsreader)
- Developer tool → monospace-forward (JetBrains Mono, Berkeley Mono)
- Creative tool → humanist sans (Satoshi, General Sans)
- Documentation → readable serif (Literata, Source Serif)

**The test**: swap your font for Inter. If nothing feels lost, your font wasn't doing any work.

---

## Step 3: Avoid Universal Anti-Patterns

### Define Your Anti-Aesthetic

Before listing what to do, list what you WON'T do. Keep a running list of visual territories to avoid — a negative-space approach that forces original decisions. Example: "No glowing grids. No gradients that look like a sci-fi poster. No identical card layouts. No decorative blobs." This is more effective than positive principles alone because it eliminates the defaults AI tools reach for, forcing every remaining choice to be deliberate.

**Visual clichés:**
- **Default typeface (Inter)** — see Font Selection above
- Purple/blue/indigo gradient as primary palette (the Tailwind default that every LLM reaches for)
- Glassmorphism/frosted glass as the entire design foundation (fine as a surgical accent on one element)
- Random blobby decorative background shapes that serve no purpose
- Every container with identical `rounded-xl border bg-card p-4` — no visual hierarchy
- Uniform spacing between all sections — no rhythm variation
- `animate-pulse` skeleton loaders (use shimmer gradients or content-shaped placeholders)
- Native browser `<select>`, checkboxes, and radio buttons in a dark theme

**Structural anti-patterns:**
- **Only designing the happy path** — you must handle empty states, error states, loading states, single items, overflow, and long text. AI only generates the golden-path screenshot.
- **Frankenstein layouts** — sections that feel randomly assembled (hero, then cards, then testimonials, then CTA) without narrative flow
- **Over-generation** — more UI elements than necessary. Every unnecessary gridline, heavy border, or decorative effect is cognitive load.
- **No design system ownership** — screens look smooth in isolation but spacing scales, type tokens, and composition rules are inconsistent across pages
- **Placeholder content in production** — "Lorem ipsum" and "Item 1" never got replaced with real data

**The real-content test**: always design with actual data — long names, empty lists, single items, overflow text. AI-generated UIs fall apart with real content.

---

## Modern Trends Worth Considering (2026)

Current design direction emphasizes **deliberate craft over decorative complexity**:

### Visual Language
- **Texture and grain** — noise overlays, layered textures that feel physical. CSS backgrounds with subtle patterns add depth that flat surfaces lack.
- **Technical mono / code brutalism** — monospaced type, terminal aesthetics, raw data presentation. Particularly strong for developer tools and data-heavy apps.
- **Asymmetry and broken grids** — expressive layouts that break the 12-column grid with purpose. Not chaos — intentional variation.
- **Monochromatic + meaningful accent** — one or two accent colors used sparingly and with purpose (the Linear approach). Powerful alternative to the "rainbow of Tailwind defaults."

### Interaction Philosophy
- **Self-evident over discoverable** — if users need a tutorial, the interface failed. Every element should communicate its purpose and state without documentation. This means text labels over icon-only buttons, status indicators that combine icon + color + text, and action labels that describe consequences ("Export to Drive" not "Submit").
- **Progressive disclosure as information architecture** — not just "hide things behind accordions." Use visual weight (typography size, color intensity, spatial position) to create natural reading layers. Users scan the top layer and drill deeper only when needed.
- **Preview-before-commit** — for any operation with side effects, show what will change *before* the user confirms. Real data previews ("12 items added, 3 removed") replace vague warnings ("This action cannot be undone").
- **Intentional motion** — animations that guide attention and confirm actions, not impress. Entrance animations on route change, staggered list loads, smooth height transitions for collapsible content. 150ms for interactions, 300ms for layout shifts.
- **Graceful state handling** — loading skeletons that match final layout shape (not spinners), empty states that explain + suggest + offer an action (not blank space), error states that show recovery paths (not "Something went wrong").

### Emerging Patterns
- **Spatial interfaces** — depth through layering (inset/flat/elevated surfaces), not just drop shadows. Background grain, subtle gradients, and surface hierarchy create dimensionality without skeuomorphism.
- **Organic warmth** — hand-drawn accents, organic shapes, character-rich serifs. Design that feels like someone cared about it.
- **AI-aware design** — as more interfaces are AI-generated, distinctiveness becomes a feature. Projects that invest in a specific visual identity stand out from the sea of indigo-gradient sameness. The bar for "looks professional" has risen because generic-professional is now free.

These are options, not requirements. Pick what fits your identity — the point is making deliberate choices, not following every trend. For detailed interaction patterns (progressive disclosure, confirmation flows, status communication), see [Interaction Design Patterns](interaction-design-patterns.md).

---

## Component Primitives as Design Enforcement

Design identity docs describe *values* — warm neutrals, rounded corners, generous spacing. Values get interpreted differently by each page. A "rounded corner" directive produces `rounded-md` on one page, `rounded-lg` on another, and `rounded-xl` on a third. A "consistent button styling" directive produces three hand-rolled inline class strings that look almost — but not quite — the same. Over a few months this accumulates into a fractured UX where the same interaction *concept* looks different across pages.

**The fix: enforce consistency through code, not code review.**

Extract shared primitives early — not after drift, but as part of initial design system setup. Then name them in your `.claude/rules/` file so AI tools reach for the component instead of re-deriving styling from principles.

### What to extract

These patterns appear across multiple pages and must look identical everywhere:

| Primitive | Why it drifts | What to extract |
|---|---|---|
| **Buttons** | Each page hand-rolls `rounded-lg border bg-card px-4 py-2 ...` | A `<Button>` component with variant + size props |
| **Input styles** | Inline class strings diverge on padding, radius, focus ring | Composable class constants (`baseInputClass`, `selectInputClass`) |
| **Cards** | Different radius, shadow, padding across pages | Documented class string in the rules file (e.g., `rounded-xl border bg-card p-6 shadow-sm`) |
| **Mode toggles** | Three implementations for the same 2-option switcher | A `<SegmentedControl>` with sliding indicator (see [Interaction Design Patterns](interaction-design-patterns.md#segmented-controls--mode-toggles)) |
| **Page states** | Loading/error/empty reinvented per page | `<PageLoading>`, `<PageError>`, `<PageEmpty>` components |
| **Page header** | h1 + icon + flex layout hand-rolled on each page | A `<PageHeader>` component |

### The rule

If the same visual pattern appears on two pages, it must come from a shared component or constant — never from two inline class strings that happen to match today. Today's matching strings are tomorrow's subtle inconsistencies.

---

## Encoding Your Identity in `.claude/rules/`

Your visual identity should live in a `.claude/rules/web-design-system.md` file scoped to component and page paths. This ensures Claude enforces your specific aesthetic every time it writes UI code.

**Structure your rule file around:**
1. A 2-3 line identity statement (who the app is for, what it should feel like)
2. Your typography hierarchy (which fonts, where each is used)
3. Your specific anti-patterns (what to avoid in YOUR context)
4. Your signature elements (what makes your app visually recognizable)

**Example** — a music metadata tool with a dark editorial aesthetic:

```markdown
---
paths:
  - "web/src/components/**"
  - "web/src/pages/**"
---
# Web Design System — [Your Aesthetic Name]

[Your one-line identity statement. e.g., "A focused tool for data professionals. Precise, dense, intentionally crafted."]

## Typography Hierarchy (enforce)
- Display font (Space Grotesk): headings, buttons, nav labels
- Body font (Newsreader): descriptions, prose, metadata values
- Mono font (JetBrains Mono): IDs, durations, timestamps, codes

## Visual Identity
- 3-level depth system (inset/flat/elevated) — no uniform containers
- Background grain texture overlay — always present
- Asymmetric borders — left-accent bars over full border boxes
- Entrance animations on route change, staggered list loads

## Avoid
- Indigo/blue gradients, glassmorphism, blobby shapes
- Uniform card layouts, uniform spacing
- animate-pulse skeletons (use shimmer gradient)
- Native form controls in dark theme

## Component Primitives (always use these)
- All buttons: `<Button>` component (variants: primary/secondary/destructive, sizes: default/sm)
- Text inputs: compose from `baseInputClass` constant — never hand-roll inline
- Mode toggles: `<SegmentedControl>` with sliding indicator
- Page states: `<PageLoading>`, `<PageError>`, `<PageEmpty>`
- Top-level cards: `rounded-xl border bg-surface-elevated p-6 shadow-sm`
```

A different project would make entirely different choices. A children's education app might specify bold primary colors, rounded playful shapes, and generous whitespace. A fintech dashboard might specify tight data density, monochrome with green/red accents, and no decorative elements. **The structure is the same; the choices are yours.**
