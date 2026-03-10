# Frontend Architecture: IA, Shell, Theme & User Flows

> **Scope**: Information architecture, app shell, navigation, design system implementation, user state, and user flow documentation
> **Prerequisites**: [Design Identity](react-design-identity.md), [React Tooling](react-tooling.md)
> **Deliverables**: Page inventory, navigation model, layout routes, semantic color tokens, dark/light mode, user state pattern, user flow document, UI audit checklist
> **Estimated effort**: M

The design identity guide answers *what should this look like?* This guide answers *how is the app structured, how does the user move through it, and how does the design system get implemented in code?*

Doing this upfront means every page you build inherits navigation, shared chrome, dark mode, and user state from the start — no retrofitting.

---

## Step 1: Information Architecture

Before building any pages, decide what pages exist and how they relate. This determines navigation, routing, URL structure, and what the user sees first.

### Page Inventory

List every page the app will eventually have, even if most are future work. Mark which ship in v0.1.x vs later:

```markdown
| Page | Purpose | Ships in | Default route? |
|---|---|---|---|
| Dashboard | At-a-glance summary, quick actions | v0.2.x | Yes (eventually) |
| Items | Primary data table, filtering, detail views | v0.2.x | No |
| Import | Data ingestion flow (upload, preview, confirm) | v0.1.x | Yes (until Dashboard) |
| Settings | User config, category mappings, theme toggle | v0.1.x (stub) | No |
```

**Why now**: Knowing the full page set upfront means you build the shell once and slot pages into it.

### Navigation Model

Pick a navigation pattern based on your page count and app type:

| Pattern | When to use | Example apps |
|---|---|---|
| **Left sidebar** | 4-8 pages, data-heavy, desktop-first | Linear, Notion, finance apps |
| **Top nav** | 2-4 pages, content-focused, simple hierarchy | Blogs, marketing sites |
| **Bottom tabs** | 3-5 pages, mobile-first | Mobile apps, progressive web apps |
| **No nav** (single page) | 1 page, utility tool | Calculators, converters |

For most full-stack apps with a backend: **left sidebar**. It scales to 8+ pages without redesign, accommodates user identity display, and is the industry standard for data tools.

### URL Structure

Define routes for all pages, including future ones. This locks in URL structure early so links and bookmarks remain stable as features ship:

```
/                   → Dashboard (or redirect to /import until Dashboard ships)
/items              → Items list
/import             → Import flow
/settings           → Settings
```

### Record It

Create `docs/user-flows.md` with your page inventory, navigation model, and URL structure. This becomes the source of truth for the frontend.

---

## Step 2: User Flows

Document how users move through the app before building UI. This surfaces missing pages, awkward transitions, and state gaps early — when they're cheap to address.

### Personas

Define 1-2 personas with enough specificity to drive UI decisions:

```markdown
**Power User** (primary)
Uses the app weekly. Comfortable with data-dense interfaces. Primary question:
"What's the status?" Sessions are short (~10 min). Wants to get in, see the
answer, and get out.
```

Generic personas ("tech-savvy professional") don't drive decisions. Specific ones ("checks the dashboard every Monday morning for 5 minutes before a meeting") do — they tell you whether to optimize for density, speed, or discoverability.

### User Stories

Write acceptance criteria for every interaction. Format:

```markdown
**US-IMPORT-1**: As a user, I want to upload a CSV for a specific month.

Acceptance criteria:
- [ ] Given I navigate to Import, then my identity is pre-filled
- [ ] Given I select a file and month, when I click Upload, then the file is parsed
- [ ] Given a file was already uploaded for this month, when I re-upload, then the
      previous data is replaced
```

### User Journeys

Map end-to-end flows as tables — these reveal missing screens and awkward transitions:

```markdown
### Journey: First-Time Setup

| Step | Screen | Action | Result |
|---|---|---|---|
| 1 | Welcome (full-screen) | Open app | See setup form |
| 2 | Welcome | Enter names, submit | Data created |
| 3 | Profile Picker | Select identity | Stored in localStorage |
| 4 | Main app | Redirected | Sidebar visible, identity shown |
```

**Why journeys matter**: User stories describe isolated interactions. Journeys reveal dependency chains — e.g., "after import, where does the user see the data?" surfaces the need for a list page, which needs a nav item, which needs a sidebar, which needs the app shell.

---

## Step 3: App Shell & Layout Routes

The app shell is the persistent chrome around page content — sidebar, header, footer. Build it before building pages.

### Layout Route Pattern

Use React Router's layout routes so the shell renders once and pages swap inside it:

```tsx
// Conceptual structure — adapt to your router
const router = createBrowserRouter([
  {
    // App shell wraps all authenticated/setup pages
    element: <AppLayout />,  // Sidebar + <Outlet />
    children: [
      { path: "/", element: <DashboardPage /> },
      { path: "/items", element: <ItemsPage /> },
      { path: "/import", element: <ImportPage /> },
      { path: "/settings", element: <SettingsPage /> },
    ],
  },
]);
```

```tsx
// AppLayout.tsx
function AppLayout() {
  return (
    <div className="flex min-h-screen">
      <Sidebar />
      <main className="flex-1 overflow-y-auto">
        <Outlet />
      </main>
    </div>
  );
}
```

### Gating Screens

Some screens render *outside* the shell — setup flows, identity selection, onboarding. These are full-screen gating pages that block access to the main app until a condition is met:

```tsx
function App() {
  // Three-state gate:
  // 1. needs-setup → full-screen SetupPage (no shell)
  // 2. needs-identity → full-screen ProfilePicker (no shell)
  // 3. ready → AppLayout with sidebar + routes

  if (personsQuery.data && personsQuery.data.length < 2) {
    return <SetupPage />;
  }
  if (!currentUserId || !isValidUser(currentUserId, personsQuery.data)) {
    return <ProfilePicker />;
  }
  return <RouterProvider router={router} />;
}
```

### Nav Items

Build nav items that support three states: active (current route), enabled (clickable), and disabled (future page, visible but grayed out):

```tsx
function NavItem({ to, label, icon: Icon, disabled = false }) {
  if (disabled) {
    return (
      <span className="flex items-center gap-3 px-3 py-2 text-muted cursor-not-allowed">
        <Icon size={18} /> {label}
      </span>
    );
  }
  return (
    <NavLink
      to={to}
      className={({ isActive }) =>
        `flex items-center gap-3 px-3 py-2 rounded-lg transition-colors
         ${isActive ? "bg-primary/10 text-primary font-medium" : "text-muted hover:text-foreground"}`
      }
    >
      <Icon size={18} /> {label}
    </NavLink>
  );
}
```

**Why disabled items**: Showing the full navigation communicates the app's scope and direction. Users (and you) can see what's coming.

### Page Layout Rules

Once the shell exists, individual pages should NOT:
- Wrap themselves in `min-h-screen` (the shell provides this)
- Set their own background color (the shell's `<main>` does this)
- Include their own navigation (the sidebar handles this)

Pages should only contain their own content, starting with a header/title area.

---

## Step 4: Design System Implementation

The [design identity guide](react-design-identity.md) defines *what* your visual system is. This section covers *how* to implement it in code with Tailwind v4.

### Font Loading

Load the fonts you chose in your [design identity](react-design-identity.md) via `<link>` in `index.html` (not CSS `@import`, which blocks rendering):

```html
<!-- index.html <head> — use the fonts from your design identity -->
<link href="https://api.fontshare.com/v2/css?f[]=your-display-font@variable&display=swap" rel="stylesheet">
<link href="https://fonts.googleapis.com/css2?family=Your+Mono+Font:wght@400;500&display=swap" rel="stylesheet">
```

Register them in `app.css`:

```css
@theme {
  --font-sans: 'Your Display Font', sans-serif;
  --font-mono: 'Your Mono Font', monospace;
}
```

### Semantic Color Tokens

Define colors as CSS custom properties that swap between light and dark mode. Use `@theme` (NOT `@theme inline`) — inline bakes values at build time, breaking runtime dark mode switching.

```css
/* app.css */
:root {
  --color-background: oklch(0.98 0.005 80);     /* light surface */
  --color-foreground: oklch(0.25 0.01 60);      /* dark text */
  --color-card: oklch(0.97 0.005 80);
  --color-card-foreground: oklch(0.25 0.01 60);
  --color-muted: oklch(0.55 0.01 60);
  --color-primary: oklch(0.55 0.2 250);         /* your brand accent */
  --color-destructive: oklch(0.55 0.2 25);      /* error / danger */
  color-scheme: light;
}

.dark {
  --color-background: oklch(0.15 0.01 60);
  --color-foreground: oklch(0.92 0.005 80);
  --color-card: oklch(0.20 0.01 60);
  --color-card-foreground: oklch(0.92 0.005 80);
  --color-muted: oklch(0.55 0.01 60);
  --color-primary: oklch(0.60 0.2 250);
  --color-destructive: oklch(0.60 0.2 25);
  color-scheme: dark;
}

@theme {
  --color-background: var(--color-background);
  --color-foreground: var(--color-foreground);
  /* ... register all tokens so Tailwind generates utilities */
}
```

Then use `bg-background`, `text-foreground`, `bg-card` etc. in markup — no `dark:` prefixes needed for most elements.

### Dark/Light Mode Infrastructure

Three pieces: a Tailwind custom variant, a FOIT-prevention script, and a React provider.

**1. Tailwind variant** (app.css):
```css
@custom-variant dark (&:where(.dark, .dark *));
```

**2. FOIT prevention** (index.html `<head>`, before any stylesheets):
```html
<script>
  // Synchronous — must NOT be type="module" or defer
  (function() {
    var theme = localStorage.getItem('app:theme');
    var prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    if (theme === 'dark' || (!theme && prefersDark) || (theme === 'system' && prefersDark)) {
      document.documentElement.classList.add('dark');
    }
  })();
</script>
```

This runs before first paint, ensuring the correct theme is applied before any content renders.

**3. Theme provider** — a React context or lightweight store (Zustand) that exposes `theme` (system/light/dark) and `setTheme()`. Listen for `matchMedia` change events when in "system" mode to react to OS-level theme changes.

### Metadata

Replace Vite defaults with your project's identity:

- **Favicon**: Replace the Vite logo with your own (even a simple emoji-based favicon)
- **Page title**: Set per-route titles (not "Vite App")
- **`<meta name="description">`**: One sentence about what the app does

---

## Step 5: Client-Side User State

Most apps need to remember *who* the current user is across sessions. Even apps without authentication need identity management if multiple users share the tool.

### When You Need This

- Multiple users share the app (e.g., a household, a small team)
- The app pre-fills forms based on who's using it
- The UI shows different data per user

### Pattern: Lightweight Identity Store

For apps without auth, store the user ID in localStorage via a state management library:

```tsx
// Conceptual — adapt to your state library
const useIdentityStore = create(
  persist(
    (set) => ({
      currentUserId: null,
      setCurrentUserId: (id) => set({ currentUserId: id }),
      clearIdentity: () => set({ currentUserId: null }),
    }),
    { name: 'app:currentUserId' },
  ),
);
```

**Key considerations**:
- Store only the ID (not the name) — IDs are stable, names can change
- Handle stale IDs — if the stored ID doesn't match any user in the database, treat it as "no identity" and show the picker
- Handle hydration timing — the store reads localStorage asynchronously; don't flash a "no user" state while hydrating
- Provide a way to switch identity without a full page reload

### App State Machine

The combination of setup status + identity creates a simple state machine:

```
no users in DB       → SetupPage (full-screen, create users)
users exist, no ID   → ProfilePicker (full-screen, select identity)
users exist, valid ID → Main app with shell
```

This state machine should be the top-level decision in your App component.

---

## Step 6: UI Audit Checklist

Run this checklist before marking any UI milestone as complete. Each item catches a specific class of oversight that compounds if left unchecked.

### Identity & Polish
- [ ] Custom page titles per route (not "Vite App")?
- [ ] Custom favicon (not the Vite logo)?
- [ ] Custom fonts loaded and rendering (not system defaults)?
- [ ] No purple/indigo gradients (unless that's your intentional palette)?
- [ ] No placeholder text in production ("Lorem ipsum", "Welcome to...")?

### Information Hierarchy
- [ ] Most important data visually dominant on each page?
- [ ] Secondary information visually recessed (smaller, muted)?
- [ ] Whitespace groups related items?
- [ ] Cards and containers used purposefully (not uniform grid)?

### States
- [ ] Empty states for every list/table (with guidance text, not just blank)?
- [ ] Loading states that don't flash for fast responses?
- [ ] Error states with actionable messages (not just "Something went wrong")?
- [ ] Hover and focus states on all interactive elements?
- [ ] Keyboard navigation works for primary flows?

### Data-Dense Apps (if applicable)
- [ ] Numeric values right-aligned in tables?
- [ ] Number formatting consistent across the app?
- [ ] Distinct visual treatment for different value categories (positive/negative, success/error)?
- [ ] `font-variant-numeric: tabular-nums` on columns of aligned numbers?

### Dark Mode (if applicable)
- [ ] All pages render correctly in both modes?
- [ ] Native elements (selects, scrollbars, form controls) styled?
- [ ] Sufficient contrast in both modes?
- [ ] No hardcoded color values that bypass tokens?

### Technical
- [ ] No console errors in normal operation?
- [ ] Semantic HTML (headings in order, landmarks)?
- [ ] WCAG contrast ratios pass?

---

## Bootstrapping Sequence

When setting up a new project's frontend, do these in order:

1. **Define IA** — page inventory, nav model, URL structure → record in `docs/user-flows.md`
2. **Write user flows** — personas, stories, journeys → same document
3. **Implement design tokens** — fonts, colors, semantic variables in `app.css`
4. **Build dark/light infrastructure** — FOIT script, custom variant, theme provider
5. **Build app shell** — sidebar, layout route, nav items (including disabled future pages)
6. **Build gating screens** — setup page, identity picker (if needed)
7. **Set up user state** — identity store, app state machine
8. **Update metadata** — favicon, page titles, description
9. **Then build feature pages** — they slot into the existing shell

Steps 1-8 ensure every feature page you build inherits navigation, theming, and user state automatically.
