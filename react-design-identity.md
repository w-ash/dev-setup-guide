# Visual Identity & Anti-AI-Slop Design

> **Scope**: How to develop a unique visual identity for your project and avoid AI-generated design patterns
> **Prerequisites**: [React Tooling](react-tooling.md)
> **Deliverables**: Design identity answers documented, `.claude/rules/web-design-system.md` written with your project's specific aesthetic
> **Estimated effort**: M

Before writing a single component, define your application's visual identity. This section is about the *process* of making intentional design choices — not a specific aesthetic to copy.

---

## The Sameness Problem

Every major LLM trains on the same Bootstrap layouts, Tailwind templates, and UI kit screenshots. Without specific guidance, they converge on the **statistical average of "website"** — indigo gradients, uniform card grids, identical spacing, glassmorphism everywhere. The result: every AI-assisted project looks identical. Assembly-line output with no soul, no emotional connection, no craft.

The fix isn't avoiding AI tools — it's giving them a strong, specific identity to enforce.

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

Write these answers down. They become your design brief — the "why" behind every visual choice.

---

## Step 2: Build a Visual System

A visual system is a set of constrained, intentional tokens — not a component library. Components come from the system; the system comes from your identity.

**Define these before building components:**

| Token | What to decide | Why it matters |
|---|---|---|
| **Type scale** | 2-3 font families, a size scale, hierarchy rules | Typography IS identity. Default Tailwind sizes with Inter is the #1 AI-slop signal. |
| **Color palette** | A constrained palette (monochrome + 1-2 accents) | Not the AI default indigo. Choose colors that reflect your app's personality. |
| **Spacing scale** | A defined rhythm (4px, 8px, 12px, 16px, 24px, 32px, 48px) | Vary the rhythm. Uniform spacing between everything is a dead giveaway. |
| **Depth system** | How to distinguish surface levels (shadow, border, background) | Not every card at the same elevation. Create hierarchy through depth. |
| **Motion language** | When things move, how fast, what easing | Animations guide attention, not impress. 150ms for interactions, 300ms for layout. |

**The defensibility test**: for every element on screen, you should be able to explain why it exists and why it's positioned where it is. If you can't, cut it.

---

## Step 3: Avoid Universal Anti-Patterns

These patterns signal "AI generated this" regardless of your specific aesthetic:

**Visual cliches:**
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
- **Over-generation** — more UI elements than necessary. Every unnecessary gridline, heavy border, or decorative effect is cognitive load. Less is more.
- **No design system ownership** — screens look smooth in isolation but spacing scales, type tokens, and composition rules are inconsistent across pages
- **Placeholder content in production** — "Lorem ipsum" and "Song Title x5" never got replaced with real data

**The real-content test**: always design with actual data — long names, empty lists, single items, overflow text. AI-generated UIs fall apart with real content.

---

## Modern Trends Worth Considering (2025-2026)

The dominant design response to AI saturation is **deliberate imperfection and tactile warmth**:

- **Texture and grain** — noise overlays, layered textures that feel physical, not sterile. Breaks the "flat digital" monotony that AI defaults to.
- **Technical mono / code brutalism** — monospaced type, terminal aesthetics, raw data presentation. Particularly strong for developer tools and data-heavy apps.
- **Intentional motion** — animations that guide attention and confirm actions, not impress. Subtle microinteractions that make interfaces feel alive.
- **Asymmetry and broken grids** — expressive layouts that break the 12-column grid with purpose. Not chaos — intentional variation.
- **Anti-corporate warmth** — hand-drawn accents, organic shapes, wonky serifs. Signals "a human made choices here."
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
