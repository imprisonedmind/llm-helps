# AGENTS.md — React + TypeScript (shadcn/ui-first, Client/Service pattern)

Purpose: This file guides agents building small React + TypeScript UIs using a Client/Service pattern. Keep components
accessible, typed, minimal, and production-ready.
This file is symlinked and shared across multiple projects; always treat it as a symlink.
It provides code patterns and structures for building React client applications with React Router, Vite, Tailwind, and
shadcn/ui, using the client/service pattern to interact with a Django REST API.
Dev/Prod Rule: Use Bun for local development only. Use npm for production (CI/build/serve). Do not mix runners.

## Startup Checklist (Agent)

- Read this file fully.
- Use Bun for dev workflows; npm only for production.
- Ask: “What should we work on today?” Then propose a short plan and confirm.

## Commands

- Dev (Bun):
    - Install: `bun i`
    - Dev server: `bun run dev`
    - Typecheck: `bun run typecheck`
    - Lint/Format (if configured): `bun run lint && bun run fmt`
- Prod (npm):
    - Clean install: `npm ci`
    - Build: `npm run typecheck && npm run build`
    - Serve: `npm run start` (or `npm run serve` when appropriate)

---

## Project Structure (Baseline)

This section defines the project-agnostic structure that we follow.

```
app/
  routes/                  # React Router file-based routes (layouts, index, params)
    layout.tsx             # root layout for nested routes
    _index.tsx             # index route under current folder
    <feature>/_index.tsx   # nested index route example
    <section>/
      _index.tsx
      layout.tsx
      <resource>/$id/_index.tsx  # param route (kebab-case)
  components/              # UI + feature components (no capitalized folders)
    ui/                    # shadcn/ui primitives (source-copied)
    <feature>/             # feature folder (lowercase)
      example-card.tsx
  lib/                     # shared logic, API, types, context
    api-client/            # client/service pattern for Django REST
      client.ts            # HTTP client wrapper
      example-service.ts   # domain service (kebab-case)
    loaders/               # route data loaders (if used)
    context/               # React providers (auth, scope, etc.)
    types/                 # shared TypeScript types
    utils.ts               # shared utilities
  hooks/                   # reusable hooks (kebab-case)
  root.tsx                 # React Router app root
  app.css                  # global styles (Tailwind)
  components.json          # shadcn config (when present)
```

Conventions:

- File naming: kebab-case for all .tsx and .ts files.
- Feature folders: lowercase (do not capitalize).
- UI primitives live under app/components/ui; compose them in feature folders.
- Use the client/service pattern under app/lib/api-client to talk to Django REST.

### TypeScript Conventions
- Avoid using TypeScript assertion syntax `as Type`. Prefer typed assignment to declare
  intent and keep types explicit, for example: `const value: SomeType = expr`.

---

## TSX Component Generation Rules

- File naming: kebab-case for all .tsx and .ts files (e.g., `app/components/<feature>/event-card.tsx`).
- Default export: export default FileName() at the top of the file.
- Props: define an interface FileNameProps and type the component signature with it.
- React 19 Compiler: assume the React 19 compiler is enabled; structure components so props/state are easily analyzable (pure, predictable).
- Memo hooks: do not reach for `useMemo` or `useCallback`; the compiler handles memoization. If you need stable references, extract helpers or move logic up instead of using these hooks.
- Data source: never fetch in a component; pass data via props if one level deep, or read from a provider if deeper in
  the tree.
- Precompute values: declare consts at the top of the component using helper functions; do not call helper functions
  inline in JSX.
- Props pattern: accept optional className and prefer composition (children) over config-heavy props.
- Variants: use cva or a simple union + mapping for visual variants when needed.
- shadcn integration: wrap/compose primitives from app/components/ui/*; use forwardRef for focusable elements and set
  displayName.
- Styling: Tailwind utilities; tokens via CSS vars; compose with cn; avoid deeply nested selectors.
- Whitespace: leave a blank line between sibling JSX elements ("new HTML tags") for readability.
- Whitespace: leave a blank line between sibling consts/functions and the return block
  Example (generic component following the rules):

```tsx
// app/components/<feature>/example-card.tsx
import * as React from "react"
import {cn} from "~/lib/utils"

// import { useExample } from "~/lib/context/example-provider" // provider example (if needed)
interface ExampleCardProps {
  title: string
  subtitle?: string
  tone?: "neutral" | "positive" | "warning" | "danger"
  className?: string
}

export default function ExampleCard({title, subtitle, tone = "neutral", className}: ExampleCardProps) {
  // const { valueFromProvider } = useExample() // if needed, read from provider (no fetching here)
  // Precompute values (no helper calls inside JSX)
  const label = subtitle ?? ""
  const containerClasses = cn("card p-3", className);
  const toneClass = tone === "positive" ? "text-green-400" : tone === "warning" ? "text-yellow-400" : tone === "danger" ? "text-red-400" : "text-muted-foreground";
  // Always white-space between consts and the return block

  return (
    <div className={containerClasses}>
      <div className="font-medium truncate">{title}</div>
      {/* this space is important and always present between new HTML tags */}
      {label && <div className={cn("text-xs truncate", toneClass)}>{label}</div>}
    </div>
  );
}
```

---

## Client & Service Rules

- Location: `app/lib/api-client/`. Filenames are kebab-case (e.g., `client.ts`, `users-service.ts`).
- Client (`client.ts`): a single HTTP wrapper class providing common concerns:
    - Base URL from env; normalizes slashes; uses credentials as configured.
    - Header injection: JSON `Content-Type`, CSRF token from cookies, Authorization from session (`Bearer <token>`).
    - Methods: `get()` returns parsed JSON; `post/put/patch/delete()` return `Response`; `getBlob()` returns `Blob`;
      `stream()` supports SSE and returns a cleanup function.
    - Session: holds a session service for token/user and exposes `waitForRefresh()` flow before concurrent GET/stream
      calls.
    - Export a singleton as `client` (via `getInstance`) and re-export from `app/lib/api-client/index.ts`.
- Services (`<domain>-service.ts`): classes that accept the client via constructor and expose domain methods:
    - No UI or React imports. Pure data calls and light payload/param shaping (e.g., `URLSearchParams`).
    - Build relative paths; do not hardcode base URLs.
    - Return typed results from `get()`; for mutate calls, check `resp.ok` and parse/throw a useful `Error` message (
      prefer server `detail` when present).
    - Keep helper payload builders `private` inside the service when needed.
- Usage: `import { client } from "~/lib/api-client"` and call services, or `new <Domain>Service(client)` if used
  standalone.
  Minimal examples:

```ts
// app/lib/api-client/client.ts
export interface ApiClientConfig {
  url: string
}

export class ApiClient {
  private static instance: ApiClient | null = null;
  private readonly baseUrl: string;

  // session: SessionsService; // optional session holder
  private constructor(cfg: ApiClientConfig) {
    this.baseUrl = cfg.url.replace(/\/+$/, "");
  }

  static getInstance(cfg?: ApiClientConfig): ApiClient {
    if (!ApiClient.instance) {
      if (!cfg) throw new Error("ApiClient config not provided.");
      ApiClient.instance = new ApiClient(cfg);
    }
    return ApiClient.instance;
  }

  private headers(): Record<string, string> {
    return {'Content-Type': 'application/json'};
  }

  private url(p: string): string {
    return `${this.baseUrl}/${(p || '').replace(/^\/+/, '')}`
  }

  async get<T = unknown>(path: string): Promise<T> {
    const r = await fetch(this.url(path), {
      method: 'GET',
      headers: this.headers(),
      credentials: 'include' as RequestCredentials
    });
    if (!r.ok) throw new Error(r.statusText);
    return r.json() as Promise<T>;
  }

  post(path: string, body?: unknown) {
    return fetch(this.url(path), {
      method: 'POST',
      headers: this.headers(),
      credentials: 'include' as RequestCredentials,
      body: JSON.stringify(body)
    });
  }

  put(path: string, body: unknown) {
    return fetch(this.url(path), {
      method: 'PUT',
      headers: this.headers(),
      credentials: 'include' as RequestCredentials,
      body: JSON.stringify(body)
    });
  }

  patch(path: string, body: unknown) {
    return fetch(this.url(path), {
      method: 'PATCH',
      headers: this.headers(),
      credentials: 'include' as RequestCredentials,
      body: JSON.stringify(body)
    });
  }

  delete(path: string) {
    return fetch(this.url(path), {
      method: 'DELETE',
      headers: this.headers(),
      credentials: 'include' as RequestCredentials
    });
  }
}

export const client = ApiClient.getInstance({url: import.meta.env.API_BASE_URL});
```

```ts
// app/lib/api-client/users-service.ts
import type {ApiClient} from './client'

export interface User {
  id: string;
  name: string
}

export default class UsersService {
  private client: ApiClient;

  constructor(client: ApiClient) {
    this.client = client
  }

  // Read operations return typed JSON
  list(): Promise<User[]> {
    return this.client.get<User[]>('users/')
  }

  getById(id: string): Promise<User> {
    return this.client.get<User>(`users/${id}`)
  }

  // Mutations return Response; check ok and parse server message on error
  async create(input: { name: string }): Promise<User> {
    const resp = await this.client.post('users/', input);
    if (!resp.ok) {
      try {
        const data = await resp.json();
        throw new Error((data?.error || data?.detail) || resp.statusText)
      } catch {
        throw new Error(resp.statusText)
      }
    }
    return resp.json() as Promise<User>;
  }
}
```

---

## Data Fetching

- Fetch data in a route `clientLoader` and return raw Promises (non-blocking).
- Use `<Suspense>` and `<Await>` in JSX to render async results with fallbacks.
- Do not block the page in `clientLoader` by awaiting network calls unless absolutely required for bootstrapping.
- Avoid fetching in `useEffect`; prefer route `clientLoader`. Use effects only when driven by explicit user interaction
  or non-route-bound streams.
- Pass data via props to immediate children; for deeper trees, read from a provider (context).
  Minimal example
- Always have white-space between consts/functions and the return block

```tsx
// app/routes/<section>/_index.tsx
import type {Route} from "./+types/_index";
import {Suspense} from "react";
import {Await} from "react-router";
import {client} from "~/lib/api-client";

export async function clientLoader({request}: Route.ClientLoaderArgs) {
  const url = new URL(request.url);
  const page = Number(url.searchParams.get("page") || "1");
  // non-blocking: return the Promise directly
  const items = client.example.list({page});
  return {items};
}

export default function Index({loaderData}: Route.ComponentProps) {
  const {items} = loaderData;
  // always white-space between consts and return block

  return (
    <div className="p-4">
      <Suspense fallback={<div className="muted">Loading…</div>}>
        <Await resolve={items} errorElement={<div className="muted">Failed to load.</div>}>
          {(page: { results: Array<{ id: string; name: string }> }) => (
            <ul className="space-y-2">
              {page.results.map((it) => (
                <li key={it.id} className="card p-3">{it.name}</li>
              ))}
            </ul>
          )}
        </Await>
      </Suspense>
    </div>
  );
}
```

---

## Patterns & Practices

- Naming: kebab-case for all .tsx and .ts files; feature folders are lowercase.
- Components: follow the TSX rules above (default export at top, <FileName>Props interface, precomputed consts, spacing
  between sibling tags, no inline helper calls in JSX).
- Data flow: follow Data Fetching rules (fetch in clientLoader, render with Suspense/Await, no fetching in components).
- Routing: file-based under app/routes with layout.tsx and _index.tsx; param segments use kebab-case.
- UI: use shadcn/ui primitives under app/components/ui; compose primitives for features, avoid giant components.
- Accessibility: every interactive element has labels/roles and keyboard support.
- Performance: memoize expensive lists, avoid unnecessary context, split code into focused routes/components.
- Errors: show concise user-facing errors; prefer server-provided detail; avoid crashing the page.

## Tooling

- TypeScript: strict mode enabled (see tsconfig.json); JSX runtime set to react-jsx; paths alias "~/*" -> "app/*".
- Vite: uses @react-router/dev, @tailwindcss/vite, and vite-tsconfig-paths plugins; base path \"/\" in dev and
  \"/static/\" in builds.
- Tailwind CSS (v4): configured in app/app.css with tailwindcss-animate plugin; use tailwind-merge for class merging
  where helpful.
- Prettier: configured with printWidth 120 (see .prettierrc).
- ESLint: optional. If used, prefer typescript-eslint, react-hooks, and a11y rules; keep config lightweight.

## Git/CI

- Commits: batch related changes with concise conventional commit messages (use scopes like ui, routes, api-client).
  Example: `chore(ui): rounded corners on buttons`
- Pre-commit: run type generation + typecheck locally. Use the dev runner: `bun run typecheck`.
  Ensure route typegen and TS types are up-to-date before committing.
- PRs: include before/after screenshots for UI where appropriate.
- CI: `typecheck → lint → build`.

## Agent Workflow (Always)

1. Clarify task & constraints; propose options when trade-offs exist.
2. Plan (small checklist); confirm.
3. Build components following TSX rules: kebab-case filenames, default export at top, <Name>Props interface, precomputed
   consts, and spacing between sibling JSX tags.
4. Fetch data via route clientLoader (non-blocking Promises) and render with <Suspense>/<Await>; avoid useEffect-based
   fetching unless absolutely necessary.
5. Connect data to UI: pass via props from loaders; use providers for deeper trees. Components do not call services
   directly except in rare edge cases.
6. Run typegen + typecheck and lint (e.g., bun run typecheck); fix issues. Summarize the change and suggest the next
   increment.
7. Always follow The Grug Brained Developer Philosophy when writing code.
