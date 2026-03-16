# Frontend Architecture: IA, Shell, Theme & User Flows

> **Scope**: Information architecture, app shell, navigation, design system implementation, user state, and user flow documentation
> **Prerequisites**: [Design Identity](react-design-identity.md), [React Tooling](react-tooling.md)
> **Deliverables**: Page inventory, navigation model, layout routes, semantic color tokens, dark/light mode, user state pattern
> **Estimated effort**: M

The design identity guide answers *what should this look like?* This guide answers *how is the app structured, how does the user move through it, and how does the design system get implemented in code?*

---

## Step 1: Information Architecture & User Flows

Before building any pages, decide what pages exist and how they relate. Document how users move through the app before building UI — this surfaces missing pages, awkward transitions, and state gaps early.

### Page Inventory & URL Structure

List every page the app will eventually have, even if most are future work:

```markdown
| Page | Purpose | Ships in | Route | Default? |
|---|---|---|---|---|
| Dashboard | At-a-glance summary | v0.2.x | / | Yes (eventually) |
| Items | Data table, filtering | v0.2.x | /items | No |
| Import | Data ingestion flow | v0.1.x | /import | Yes (until Dashboard) |
| Settings | User config, theme | v0.1.x (stub) | /settings | No |
```

Lock in URL structure early so links and bookmarks remain stable as features ship.

### Navigation Model

| Pattern | When to use | Example apps |
|---|---|---|
| **Left sidebar** | 4-8 pages, data-heavy, desktop-first | Linear, Notion, finance apps |
| **Top nav** | 2-4 pages, content-focused | Blogs, marketing sites |
| **Bottom tabs** | 3-5 pages, mobile-first | Mobile apps, PWAs |
| **No nav** | 1 page, utility tool | Calculators, converters |

For most full-stack apps: **left sidebar**. It scales to 8+ pages and accommodates user identity display.

### Personas & User Journeys

Define 1-2 specific personas (not "tech-savvy professional" — instead: "checks the dashboard every Monday for 5 minutes"). Then map end-to-end flows as tables to reveal missing screens:

```markdown
| Step | Screen | Action | Result |
|---|---|---|---|
| 1 | Welcome | Open app | See setup form |
| 2 | Welcome | Submit | Data created |
| 3 | Profile Picker | Select identity | Stored in localStorage |
| 4 | Main app | Redirected | Sidebar visible |
```

Record all of this in `docs/user-flows.md` — page inventory, navigation model, personas, journeys, and user stories with acceptance criteria.

---

## Step 2: App Shell & Layout Routes

The app shell is the persistent chrome around page content — sidebar, header, footer. Build it before building pages.

### Layout Route Pattern

Use React Router's layout routes so the shell renders once and pages swap inside it:

```tsx
const router = createBrowserRouter([
  {
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

Some screens render *outside* the shell — setup flows, identity selection. These are full-screen gating pages:

```tsx
function App() {
  if (personsQuery.data && personsQuery.data.length < 2) return <SetupPage />;
  if (!currentUserId || !isValidUser(currentUserId, personsQuery.data)) return <ProfilePicker />;
  return <RouterProvider router={router} />;
}
```

### Nav Items

Build nav items that support three states: active (current route), enabled (clickable), and disabled (future page):

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

**Why disabled items**: Showing the full navigation communicates the app's scope. Users can see what's coming.

### Page Layout Rules

Pages should NOT set their own `min-h-screen`, background color, or navigation — the shell provides these. Pages contain only their own content.

---

## Step 3: Design System Implementation

The [design identity guide](react-design-identity.md) defines *what* your visual system is. This section covers *how* to implement it in code with Tailwind v4.

### Font Loading

Load fonts via `<link>` in `index.html` (not CSS `@import`, which blocks rendering):

```html
<link href="https://api.fontshare.com/v2/css?f[]=your-display-font@variable&display=swap" rel="stylesheet">
<link href="https://fonts.googleapis.com/css2?family=Your+Mono+Font:wght@400;500&display=swap" rel="stylesheet">
```

Register in `app.css`:
```css
@theme {
  --font-sans: 'Your Display Font', sans-serif;
  --font-mono: 'Your Mono Font', monospace;
}
```

### Semantic Color Tokens

Define colors as CSS custom properties that swap between light and dark mode. Use `@theme` (NOT `@theme inline`):

```css
:root {
  --color-background: oklch(0.98 0.005 80);
  --color-foreground: oklch(0.25 0.01 60);
  --color-card: oklch(0.97 0.005 80);
  --color-muted: oklch(0.55 0.01 60);
  --color-primary: oklch(0.55 0.2 250);
  --color-destructive: oklch(0.55 0.2 25);
  color-scheme: light;
}

.dark {
  --color-background: oklch(0.15 0.01 60);
  --color-foreground: oklch(0.92 0.005 80);
  --color-card: oklch(0.20 0.01 60);
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

Then use `bg-background`, `text-foreground`, `bg-card` etc. — no `dark:` prefixes needed.

### Dark/Light Mode Infrastructure

Three pieces:

**1. Tailwind variant** (app.css):
```css
@custom-variant dark (&:where(.dark, .dark *));
```

**2. FOIT prevention** (index.html `<head>`, before stylesheets):
```html
<script>
  (function() {
    var theme = localStorage.getItem('app:theme');
    var prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    if (theme === 'dark' || (!theme && prefersDark) || (theme === 'system' && prefersDark)) {
      document.documentElement.classList.add('dark');
    }
  })();
</script>
```

**3. Theme provider** — a React context or Zustand store exposing `theme` (system/light/dark) and `setTheme()`. Listen for `matchMedia` change events in "system" mode.

### Metadata

Replace Vite defaults: custom favicon, per-route page titles, and `<meta name="description">`.

---

## Step 4: Client-Side User State

Most apps need to remember *who* the current user is across sessions.

### When You Need This

- Multiple users share the app (household, small team)
- The app pre-fills forms based on who's using it
- The UI shows different data per user

### Pattern: Lightweight Identity Store

For apps without auth, store the user ID in localStorage:

```tsx
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
- Store only the ID — IDs are stable, names change
- Handle stale IDs — if the stored ID doesn't match any user, show the picker
- Handle hydration timing — don't flash "no user" while localStorage loads
- Provide identity switching without full page reload

### App State Machine

```
no users in DB       → SetupPage (full-screen)
users exist, no ID   → ProfilePicker (full-screen)
users exist, valid ID → Main app with shell
```

This state machine should be the top-level decision in your App component.
