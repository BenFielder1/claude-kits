---
name: react-vite-ui
description: Use when building or changing a TypeScript + React + Vite frontend, covering routes, components, data fetching via the generated API client, forms, permissions, runtime config for Kubernetes, accessibility and tests.
---

# React + Vite frontend workflow

## 0. Detect before you write
Check the router (React Router / TanStack Router), the server-state library (TanStack Query / RTK Query / SWR), the forms (React Hook Form, Formik), the schema validation (Zod, Yup), the styling (Tailwind, CSS Modules, styled-components, or a design system / component library), the API client (generated, and how), the i18n setup, the test stack, and the folder structure (feature folders or by type). **Use what exists.** Find the closest existing feature and mirror its structure. Use design-system components before writing new primitives.

## 1. Rules
- **TypeScript strict**: no `any` and no non-null `!` without a comment. Shared types come from the generated client, not hand-copied interfaces.
- **API access only through the generated client** (see `api-contract`). No hand-written `fetch` calls to backend endpoints, and never edit generated files.
- **Server state lives in the query library.** Use stable query keys (`['orders', id]`), invalidate or update after mutations, and don't copy server data into local state.
- **Map errors** from ProblemDetails `errorCode` to user copy in one module (e.g. `src/lib/errors.ts`). Show field errors from validation problems against the matching fields.
- **Permissions**: write pure helpers (`canEditOrder(user, order)`) from the spec's permission table, and apply them to controls. Always handle 401 (re-auth), 403 and 409 anyway, because the backend is the real check.
- **Every data view has loading, empty, error and forbidden states**, plus read-only where the spec defines it.
- **Accessibility**: semantic elements, labelled inputs, `aria-label` on icon buttons, focus management in dialogs, keyboard paths, and AA contrast.
- **Performance**: lazy-load routes (`React.lazy` or the router's lazy routes). Memoise only when profiling shows a need.

## 2. Configuration for Kubernetes (build once, deploy many)
`import.meta.env.VITE_*` values are **baked in at build time**, and the same image goes to dev, staging and prod. So environment-specific values (API base URL, auth authority, feature flags) must come from **runtime config**, using whichever pattern the repo already has:
- a `config.json` fetched at startup, served by the container from a ConfigMap mount, or
- `window.__ENV__` written by the container entrypoint from environment variables.
Only build-time constants belong in `VITE_*`. Nothing secret belongs in the frontend at all.

## 3. Contract-first work with mocks
When the backend isn't ready, or isn't in scope:
- Generate the client from the agreed contract (or from the stubbed contract committed by the backend).
- Write **MSW** handlers that return contract-shaped data for the new endpoints, including the error cases from the spec. Use them in tests, and in dev only when the repo already supports a mock mode.
- When the real endpoint lands, regenerate the client, delete any temporary types, and run the tests again.

## 4. Tests
- **Vitest + Testing Library** for components and hooks: behaviour-driven (find by role or label, then assert what's visible), with MSW for the network.
- Cover each FR's acceptance criteria that has visible behaviour, plus form validation, permission-dependent rendering and error-code copy.
- E2E (Playwright) belongs to `qa-engineer`. Add `data-testid` only where there's no accessible name.

## Gate
`npm ci && npm run typecheck && npm run lint && npm test && npm run build`, or the component's own gate. Use the repo's package manager (npm/pnpm/yarn), based on its lockfile.

## Checklist
- [ ] Mirrors the nearest existing feature, and uses design-system components.
- [ ] Only the generated client is used, the client was regenerated after contract changes, and no generated files were edited.
- [ ] Loading, empty, error, forbidden and read-only states all exist.
- [ ] Permission helpers come from the spec, and 401/403/409 are handled.
- [ ] Environment-specific values come from runtime config, not `VITE_*`.
- [ ] Tests cover the FR acceptance criteria, and accessibility basics are met.
