# Visual Identity & Anti-AI-Slop Design

> **Scope**: How to develop a unique visual identity for your project and avoid AI-generated design patterns
> **Prerequisites**: [React Tooling](react-tooling.md)
> **Deliverables**: Design identity answers documented, `.claude/rules/web-design-system.md` written with your project's specific aesthetic
> **Estimated effort**: M

Before writing a single component, define your application's visual identity. The goal: interactions that feel **familiar enough to be intuitive, but distinctive enough to engage**. This guide covers the *process* of making intentional design choices — not a specific aesthetic to copy.

---

## Why This Matters

LLMs train on the same Bootstrap layouts, Tailwind templates, and UI kit screenshots. Without specific guidance, they converge on the **statistical average of "website"** — indigo gradients, uniform card grids, identical spacing, glassmorphism. Every AI-assisted project ends up looking the same.

The fix: give AI tools a strong, specific identity to enforce. Ground every design decision in your users' needs and your app's personality. Users should feel like someone thoughtful built this for them — not like they're using a template.

---

## Step 1: Define Your Design Identity

Answer three questions before touching CSS. These answers drive every visual decision:

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

Write these answers down. They become your design brief — the "why" behind every visual choice. Every token, layout, and interaction should trace back to these answers. If a design choice doesn't serve the user's needs or reinforce the intended feeling, question it.

---

## Step 2: Build a Visual System

A visual system is a set of constrained, intentional tokens — not a component library. Components come from the system; the system comes from your identity.

**Define these before building components:**

| Token | What to decide | Why it matters |
|---|---|---|
| **Type scale** | 2-3 font families, a size scale, hierarchy rules | Typography is identity. Choose fonts that match your app's personality and audience — explore Fontshare, Google Fonts beyond page 1. Use high-contrast weight pairing (light body, bold headlines). |
| **Color palette** | A constrained palette (monochrome + 1-2 accents) | Generate from a single brand seed color using OKLCH or HCT color spaces. Define semantic tokens (`--brand`, `--accent`, `--surface`), not raw hex values. |
| **Spacing scale** | A defined rhythm (4px, 8px, 12px, 16px, 24px, 32px, 48px) | Vary the rhythm between sections. Tighter spacing within groups, more breathing room between them. |
| **Depth system** | How to distinguish surface levels (shadow, border, background) | Define 3-5 named elevation levels with consistent shadow recipes. Create hierarchy through depth — not every card at the same level. |
| **Motion language** | When things move, how fast, what easing | Animations guide attention, not impress. 150ms for interactions, 300ms for layout. |

**The defensibility test**: for every element on screen, you should be able to explain why it exists and why it's positioned where it is. If you can't, cut it.

---

## Step 3: Recognize and Replace AI Default Patterns

These patterns appear in virtually every AI-generated interface. Each one has a specific, better alternative.

**Typography defaults:**
- Inter, Roboto, Open Sans, Lato, Arial — these dominate LLM training data and produce instantly recognizable sameness
- One font weight throughout — intentional design uses high-contrast weight variation (300 for body, 700+ for headlines)
- Default Tailwind type scale with no customization

**Color defaults:**
- `indigo-500` / `indigo-600` as primary color — Tailwind's demo color became the statistical default for every AI-generated button and link
- Purple-to-blue or pink-to-purple gradients as backgrounds or CTAs
- Gradient text on body copy (also fails WCAG contrast)
- Raw Tailwind palette colors with no semantic naming

**Layout defaults:**
- Centered hero text + CTA button + three feature cards with icons — the "SaaS landing page trinity"
- Every container with identical `rounded-xl border bg-card p-4`
- Uniform spacing between all sections with no rhythm variation
- Bento grids filled with decorative stock imagery

**Component defaults:**
- `animate-pulse` skeleton loaders (use shimmer gradients or content-shaped placeholders instead)
- Cards with no hover, focus, or empty states — only the happy path
- Native browser `<select>`, checkboxes, and radio buttons left unstyled in dark themes
- Inconsistent shadows per component — define 3-5 named elevation levels and use them everywhere
- Random blobby decorative background shapes, "light leak" effects

**Structural patterns:**
- **Only the happy path** — handle empty states, error states, loading states, overflow, and edge cases. AI generates the golden-path screenshot; a real app needs every state.
- **Over-generation** — more UI elements than the content needs. Every unnecessary border, gridline, or decorative element adds cognitive load.
- **Inconsistent tokens** — screens look fine in isolation but spacing, type, radius, and shadow values drift across pages
- **Placeholder content in production** — "Lorem ipsum", "Item 1", "Welcome to..." never got replaced with real data

**The real-content test**: design with actual data — long names, empty lists, single items, overflow text. Intentional designs handle these gracefully because every state was considered.

---

## Modern Trends Worth Considering (2025-2026)

Current design direction emphasizes **deliberate craft and tactile warmth**:

- **Texture and grain** — noise overlays, layered textures that feel physical. Layered CSS backgrounds with subtle patterns add depth that flat surfaces lack.
- **Technical mono / code brutalism** — monospaced type, terminal aesthetics, raw data presentation. Particularly strong for developer tools and data-heavy apps.
- **Intentional motion** — animations that guide attention and confirm actions, not impress. Subtle microinteractions that make interfaces feel alive.
- **Asymmetry and broken grids** — expressive layouts that break the 12-column grid with purpose. Not chaos — intentional variation.
- **Organic warmth** — hand-drawn accents, organic shapes, character-rich serifs. Design that feels like someone cared about it.
- **Monochromatic + meaningful accent** — one or two accent colors used sparingly and with purpose (the Linear approach). Powerful alternative to the "rainbow of Tailwind defaults."

These are options, not requirements. Pick what fits your identity — the point is making deliberate choices, not following every trend.

---

## Encoding Your Identity in `.claude/rules/`

Your visual identity should live in a `.claude/rules/web-design-system.md` file scoped to component and page paths. This ensures Claude enforces your specific aesthetic every time it writes UI code.

**Structure your rule file around:**
1. A 2-3 line identity statement (who the app is for, what it should feel like)
2. Your typography hierarchy (which fonts, where each is used)
3. Your specific anti-patterns (what to avoid in YOUR context)
4. Your signature elements (what makes your app visually recognizable)

**Example** — Narada (music metadata tool, dark editorial aesthetic):

```markdown
---
paths:
  - "web/src/components/**"
  - "web/src/pages/**"
---
# Web Design System — Dark Editorial Music Aesthetic

Narada is a power tool for music metadata enthusiasts — the "record store
crate-digger," not the casual listener. Precise, data-rich, intentionally crafted.

## Typography Hierarchy (enforce)
- Display font (Space Grotesk): headings, buttons, nav labels
- Body font (Newsreader): descriptions, prose, metadata values
- Mono font (JetBrains Mono): ISRCs, IDs, durations, timestamps

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
```

A different project would make entirely different choices. A children's education app might specify bold primary colors, rounded playful shapes, and generous whitespace. A fintech dashboard might specify tight data density, monochrome with green/red accents, and no decorative elements. **The structure is the same; the choices are yours.**

The goal is always the same: interactions that feel **familiar enough to be immediately usable** (users shouldn't have to learn a new paradigm) **but distinctive enough to be engaging** (users shouldn't feel like they're using a cookie-cutter template). Root every choice in what your users need and how you want them to feel.
