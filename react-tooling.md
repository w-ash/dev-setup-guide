# React Tooling Setup

> **Scope**: Vite bundling, TypeScript strict mode, and Biome linting/formatting for a React frontend
> **Prerequisites**: [Project Structure](project-structure.md)
> **Deliverables**: `web/vite.config.ts`, `web/tsconfig.json`, `web/biome.json` configured and working
> **Estimated effort**: S

---

## Vite Configuration

```typescript
// web/vite.config.ts
import tailwindcss from "@tailwindcss/vite";
import react from "@vitejs/plugin-react";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: { "@": "/src" },
  },
  server: {
    open: true,
    proxy: {
      "/api": {
        target: "http://localhost:8000",
        changeOrigin: true,
      },
    },
  },
  build: {
    outDir: "dist",
  },
});
```

Key features: Tailwind v4 plugin, `@/` import alias, and `/api` proxy to FastAPI during development.

### Tailwind v4 Setup

```bash
pnpm add tailwindcss @tailwindcss/vite
```

Entry point CSS (`web/src/index.css`):
```css
@import "tailwindcss";
```

Tailwind v4 uses automatic content detection — no `tailwind.config.js` needed. Design tokens use `@theme` blocks in CSS instead of JS config.

---

## TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  },
  "include": ["src"]
}
```

`erasableSyntaxOnly`: ensures only type-level syntax is used (no enums, no parameter properties) — aligns with modern bundler expectations.

---

## Biome Configuration

Biome replaces both ESLint and Prettier in a single Rust-based tool.

```json
{
  "$schema": "https://biomejs.dev/schemas/2.4.5/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": {
    "includes": ["**", "!!**/dist", "!!src/api/generated/**", "!!src/components/ui/**"]
  },
  "formatter": { "enabled": true, "indentStyle": "space", "indentWidth": 2 },
  "linter": { "enabled": true, "rules": { "recommended": true } },
  "css": { "parser": { "cssModules": false, "tailwindDirectives": true } },
  "javascript": { "formatter": { "quoteStyle": "double" } },
  "assist": {
    "enabled": true,
    "actions": { "source": { "organizeImports": "on" } }
  }
}
```

**Exclusions**: `api/generated/` (Orval output, auto-generated) and `components/ui/` (shadcn/ui primitives, separately maintained).
