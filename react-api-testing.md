# React API Client & Testing

> **Scope**: Orval code generation, custom fetch wrapper, QueryClient configuration, and Vitest + MSW test setup
> **Prerequisites**: [React Tooling](react-tooling.md), [Backend Patterns](backend-patterns.md) (for OpenAPI spec)
> **Deliverables**: Orval generating hooks from OpenAPI, `customFetch` + `ApiError` working, Vitest + MSW test harness running
> **Estimated effort**: M

---

## Orval — API Code Generation

Orval generates TypeScript types, React Query hooks, and MSW mock handlers from your OpenAPI spec.

```typescript
// web/orval.config.ts
import { defineConfig } from "orval";

export default defineConfig({
  myProject: {
    input: { target: "./openapi.json" },
    output: {
      mode: "tags-split",
      target: "src/api/generated",
      schemas: "src/api/generated/model",
      client: "react-query",
      mock: true,
      override: {
        mutator: { path: "src/api/client.ts", name: "customFetch" },
        query: { useQuery: true, useSuspenseQuery: false },
      },
    },
  },
});
```

- `tags-split`: splits generated files by OpenAPI tag (e.g., `items/items.ts`, `health/health.ts`)
- `mock: true`: auto-generates MSW handlers for testing
- `mutator`: points to a custom fetch wrapper that handles error envelopes
- Regenerate after API changes: `pnpm --prefix web generate`
- **Never hand-edit** files in `src/api/generated/`

---

## Custom Fetch + ApiError

```typescript
// web/src/api/client.ts
export class ApiError extends Error {
  status: number;
  code: string;
  details?: Record<string, string>;

  constructor(
    status: number,
    code: string,
    message: string,
    details?: Record<string, string>,
  ) {
    super(message);
    this.name = "ApiError";
    this.status = status;
    this.code = code;
    this.details = details;
  }
}

export async function customFetch<T>(
  url: string,
  init: RequestInit = {},
): Promise<T> {
  const response = await fetch(url, init);

  if (response.status === 204) {
    return { data: undefined, status: 204, headers: response.headers } as T;
  }

  const body = await response.json();

  if (!response.ok) {
    const error = body?.error;
    throw new ApiError(
      response.status,
      error?.code ?? "UNKNOWN_ERROR",
      error?.message ?? "An unknown error occurred",
      error?.details,
    );
  }

  return { data: body, status: response.status, headers: response.headers } as T;
}
```

This wraps every API response into an `{data, status, headers}` envelope that Orval expects, and converts error responses into typed `ApiError` instances.

---

## QueryClient Factory

```typescript
// web/src/api/query-client.ts
import { QueryClient } from "@tanstack/react-query";
import { ApiError } from "./client";

export function createQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 30_000,
        retry: (failureCount, error) => {
          if (error instanceof ApiError) {
            return error.status >= 500 && failureCount < 2;
          }
          return false;
        },
      },
    },
  });
}
```

Only retries on 5xx server errors (never on 4xx client errors). 30-second stale time prevents unnecessary refetches.

---

## Vitest Configuration

Vitest 4.x notes: `coverage.include` must be defined explicitly, `workspace` is renamed to `projects`, and pool options are top-level (`maxWorkers` replaces `maxThreads`/`maxForks`).

```typescript
// web/vitest.config.ts
import { defineConfig, mergeConfig } from "vitest/config";
import viteConfig from "./vite.config";

export default mergeConfig(
  viteConfig,
  defineConfig({
    test: {
      environment: "jsdom",
      globals: true,
      setupFiles: ["./src/test/setup.ts"],
      include: ["src/**/*.test.{ts,tsx}"],
      exclude: ["src/api/generated/**"],
      coverage: {
        include: ["src/**"],  // Required in Vitest 4.x
      },
    },
  }),
);
```

Merges the Vite config (aliases, plugins) so tests resolve `@/` imports identically to the app.

---

## Test Utilities

### MSW Server Bootstrap

```typescript
// web/src/test/setup.ts
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { setupServer } from "msw/node";
import { afterAll, afterEach, beforeAll } from "vitest";

// Import auto-generated MSW handlers from Orval
import { getHealthMock } from "@/api/generated/health/health.msw";
import { getItemsMock } from "@/api/generated/items/items.msw";

export const server = setupServer(
  ...getHealthMock(),
  ...getItemsMock(),
);

beforeAll(() => server.listen({ onUnhandledRequest: "warn" }));
afterEach(() => { server.resetHandlers(); cleanup(); });
afterAll(() => server.close());
```

### Overriding Handlers for Error States

Override the auto-generated happy-path handlers per-test to test error scenarios:

```typescript
import { http, HttpResponse } from "msw";
import { server } from "@/test/setup";

test("shows error message on API failure", async () => {
  server.use(
    http.get("/api/v1/items", () =>
      HttpResponse.json(
        { error: { code: "INTERNAL_ERROR", message: "DB unavailable" } },
        { status: 500 },
      ),
    ),
  );
  // render component and assert error UI is displayed
});
```

### renderWithProviders

```tsx
// web/src/test/test-utils.tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { type RenderOptions, render } from "@testing-library/react";
import type { ReactElement, ReactNode } from "react";
import { MemoryRouter, type MemoryRouterProps } from "react-router";

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
      mutations: { retry: false },
    },
  });
}

function Providers({ children, routerProps }: { children: ReactNode; routerProps?: MemoryRouterProps }) {
  return (
    <QueryClientProvider client={createTestQueryClient()}>
      <MemoryRouter {...routerProps}>{children}</MemoryRouter>
    </QueryClientProvider>
  );
}

interface ExtendedRenderOptions extends Omit<RenderOptions, "wrapper"> {
  routerProps?: MemoryRouterProps;
}

export function renderWithProviders(
  ui: ReactElement,
  { routerProps, ...options }: ExtendedRenderOptions = {},
) {
  return render(ui, {
    wrapper: ({ children }) => <Providers routerProps={routerProps}>{children}</Providers>,
    ...options,
  });
}

export { act, screen, waitFor, within } from "@testing-library/react";
export { default as userEvent } from "@testing-library/user-event";
```

Use `renderWithProviders()` for any component that needs hooks, routing, or queries. Use plain `render()` for pure presentational components.
