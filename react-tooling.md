# React Tooling Setup

> **Scope**: Vite bundling, TypeScript strict mode, and Biome linting/formatting for a React frontend
> **Prerequisites**: [Project Structure](project-structure.md)
> **Deliverables**: `web/vite.config.ts`, `web/tsconfig.json`, `web/biome.json` configured and working
> **Estimated effort**: S

Requires **Node.js 20.19+** or **22.12+**.

---

## Vite Configuration

Vite 8 uses **Rolldown** (Rust-based) as the unified bundler for both dev and production, replacing esbuild + Rollup.

```typescript
// web/vite.config.ts
import tailwindcss from "@tailwindcss/vite";
import react from "@vitejs/plugin-react";  // v6: uses Oxc instead of Babel
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    tsconfigPaths: true,  // Reads paths from tsconfig.json — no manual alias needed
  },
  server: {
    forwardConsole: true,  // Browser console errors appear in terminal
    open: true,
    proxy: {
      "/api": {
        target: "http://localhost:8000",  // Match your API port
        changeOrigin: true,
      },
    },
  },
  build: {
    outDir: "dist",
  },
});
```

Key features: Tailwind v4 plugin, `@/` alias via tsconfig paths (Vite 8 reads `tsconfig.json` natively), `/api` proxy to FastAPI during development, and `forwardConsole` to surface browser errors in the terminal (especially useful with coding agents). Choose unique ports per project so multiple projects can run simultaneously — update both the Vite proxy target and the FastAPI CORS origin to match.

### Code Splitting

Vite 8 replaces Rollup's `manualChunks` with Rolldown's `codeSplitting` groups. Use this to isolate heavy vendor libraries into separately-cacheable chunks:

```typescript
build: {
  rolldownOptions: {
    output: {
      codeSplitting: {
        groups: [
          { name: "heavy-lib", test: /heavy-lib/, priority: 20 },
          { name: "vendor", test: /node_modules/, priority: 10, minSize: 50_000 },
        ],
      },
    },
  },
},
```

Higher `priority` claims modules first when multiple patterns match. Combine with dynamic `import()` for libraries only needed on interaction (e.g., graph layout engines) to defer their download entirely.

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
