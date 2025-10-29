# AGENTS.md — React + TypeScript (shadcn/ui-first, Client/Service pattern)

**Purpose:** This file tells the coding agent how to build small UI projects that talk to a `client` (HTTP wrapper) and a domain `service` that talks to Django APIs. Keep components accessible, typed, minimal, and production-ready.

## Startup Checklist (Agent)
- Read this file fully.
- Detect toolchain: **Bun** (preferred if `bun.lockb`), else `pnpm`, else `npm`. Detect framework (Vite vs Next.js).
- Ask: **“What should we work on today?”** Then propose a short plan and confirm.
- If Next.js (app router): default to **Server Components**; add `"use client"` only when interactivity is required.

## Commands
- **Install:** `bun i` | `pnpm i` | `npm i`
- **Dev:** `bun dev` | `pnpm dev` | `npm run dev`
- **Build:** `bun run build` | `pnpm build`
- **Lint/Format:** `pnpm lint && pnpm fmt` (configure ESLint + Prettier) 
- **Test:** `vitest run` or `pnpm test` (Vitest + React Testing Library)

> If missing, create: `eslint.config.ts` (TypeScript strict), `prettier` config, `vitest.config.ts`, `tsconfig.json` (strict true).

---

## Project Structure (Baseline)
```
src/
  components/
    ui/                # shadcn/ui primitives (copied-in source, editable)
    <Feature>/
      <Name>.tsx
      <Name>.test.tsx
      index.ts         # barrel export
  hooks/
  lib/
    client.ts          # HTTP wrapper (fetch/axios), baseURL, auth, errors
    service/<domain>.ts # domain calls that compose client methods
    utils/cn.ts        # className merger
  pages/ or app/
  styles/
```
- **Client/Service split:** Components never call `fetch` directly → they use `service/*` which depends on `lib/client.ts`.
- **State:** Prefer **TanStack Query** (or SWR) for remote state; local UI state with `useState`/`useReducer`/Zustand only if needed.

---

## TSX Component Generation Rules
- **File:** One component per file (kebab case): `src/components/<feature>/<blah-blee>.tsx`
- **Signature:** Use function components with interface `Props`; export `Name` and `NameProps`.
- **Props pattern:**
  - Accept `className?: string` and merge via `cn`.
  - Keep props minimal; prefer composition (`children`) over config explosion.
  - For visual variants, use **cva** (class-variance-authority) or a simple `variant` union + mapping.
- **Shadcn integration:**
  - Wrap or compose `@/components/ui/*` primitives (Button, Input, Dialog, etc.).
  - Use `forwardRef` when exposing focusable DOM nodes; set `displayName`.
  - Accessible first: labels link `htmlFor`, `aria-*` are correct, focus traps in dialogs, ESC closes, etc.
- **Styling:**
  - Tailwind utilities; tokens via CSS vars; *never* inline hex colors.
  - Use `cn` to compose; avoid brittle deeply-nested selectors.
- **Data-fetching components:**
  - Never fetch data at the component layer, get data via prop if 1 level down from clientLoader or via a provider if more than 1 level down.
- **Examples:** Include a minimal usage example in JSDoc above the component.
- **Exports:** Barrel at folder-level (`index.ts`) to keep imports clean.

**Example scaffold**
```tsx
// src/components/feature/card-stat.tsx
import * as React from "react"
import {cn} from "@/lib/utils/cn"

interface CardStatProps {
    label: string
    value: string | number
    icon?: React.ReactNode
    className?: string
}

export function CardStat({label, value, icon, className}: CardStatProps) {
    const convertedIcon = someFunction(icon)
    const convertedLabel = someFunction2(label)
    const convertedValue = someFunction3(value)
    
    return (
        <div className={cn("rounded-xl border p-4 grid gap-1", className)}>
            <div className="flex items-center gap-2 text-sm text-muted-foreground">
                {convertedIcon} <span>{convertedLabel}</span>
            </div>

            <div className="text-2xl font-semibold">{convertedValue}</div>
        </div>
    )
}
```

---

## Client & Service Rules
- **lib/client.ts:** Single HTTP wrapper with:
  - Base URL from env; auth header injection; JSON encode/decode; error map (4xx → typed `ClientError`, 5xx → `ServerError`).
  - Retry (exponential backoff) on network/5xx up to small limit.
  - `get/post/put/patch/delete<T>()` generics returning typed data.
- **service/<domain>.ts:** Pure functions that compose `client` calls into domain methods (e.g., `getUserById`, `createReport`). No UI imports.
- **Errors:** Never throw raw `Error`; use discriminated unions for API errors and handle in UI.
- **Validation:** For inputs, use Zod (or TS types with runtime guards). For server responses, trust server schema but narrow types if necessary.

---

## Patterns & Practices
- **TypeScript:** `"strict": true` always. No `any`. Narrow unknowns.
- **Accessibility:** Every interactive element has a role/label and keyboard support.
- **Performance:** Memoize expensive lists; avoid unnecessary context; split code routes/components as needed.
- **Routing:** For Next.js, colocation under `app/` routes; for Vite/React-Router, keep lazy routes and data loaders minimal.
- **Design:** Follow shadcn/ui defaults; prefer composition over custom giant components. Small, focused building blocks.

---

## Testing & Tooling
- **Vitest + RTL** for components; **MSW** for API mocking at the boundary (client layer).
- **ESLint:** typescript-eslint rules; react hooks rules; import/order; no extraneous deps.
- **Prettier** for formatting; `tailwindcss` plugin for class ordering is a plus.
- **Storybook (optional):** Useful for complex visual components; add stories sparingly.

---

## Git/CI
- Conventional commits.
- PRs include before/after screenshots for UI.
- CI: `typecheck → lint → build → test` (and run a small MSW-based integration suite).

---

## Agent Workflow (Always)
1. **Clarify** task & constraints; propose 2–3 options when trade-offs exist.
2. **Plan** (small checklist); confirm.
3. **Scaffold** components/services following the rules above (generate TS types, barrels, and tests).
4. **Wire** components to services; add loading/empty/error UI states.
5. **Test** (Vitest/RTL) and run lints; fix.
6. **Summarize** the change and suggest the logical next UI increment.
