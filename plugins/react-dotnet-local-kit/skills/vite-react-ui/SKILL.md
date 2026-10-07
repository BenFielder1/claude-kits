---
name: vite-react-ui
description: Use when building or changing the TypeScript + React + Vite frontend in frontend/, covering routes, components, server state through the generated API client, forms, permissions, error handling, accessibility, MSW mocks and tests.
---

# React + Vite frontend

If the frontend already has code, **mirror the nearest existing feature** and reuse its components first.

## Defaults (when the scaffold sets them up)
- **Vite** + React + TypeScript `strict` (plus `noUncheckedIndexedAccess`), ESLint, and Prettier if used.
- **Router**: React Router (data routers) or TanStack Router. Pick one and keep to it.
- **Server state**: TanStack Query, with one `queryClient` and query-key factories per feature (`orderKeys.detail(id)`).
- **Forms**: React Hook Form + Zod (`zodResolver`).
- **Styling**: whatever CLAUDE.md records (Tailwind by default). Put shared primitives in `src/components/ui/`.
- **Tests**: Vitest + Testing Library + `@testing-library/user-event` + MSW. Playwright for E2E (owned by `e2e-qa-engineer`).

## Talking to the API
- **Only through the generated client** in `src/api/generated/` (see `openapi-contract`). Wrap it in feature hooks (`useOrder(id)`, `useLockOrder()`). Never hand-write a `fetch` to the backend, and never edit generated files.
- Requests go to **same-origin `/api`**:
  - In dev, Vite proxies `/api` to `http://localhost:5080` (`server.proxy` in `vite.config.ts`).
  - In Kubernetes, the web container's nginx proxies `/api` to the API Service.
  So there's no API base URL to configure, and no CORS.
- Auth with cookies: requests include credentials (same-origin). On a 401, clear the user cache and redirect to sign-in with `?next=`.
- After a mutation, invalidate or update the affected query keys. Don't copy server data into local state.

## Errors, permissions and states
- Map ProblemDetails `errorCode` values to user copy in `src/lib/errors.ts`. Map validation `errors` onto form fields with `setError`.
- Permissions: write pure helpers in `src/lib/permissions.ts` from the spec's permission table, and use them to hide or disable controls. Always handle 401, 403 and 409 regardless, because the server is the real check.
- Every data view has **loading, empty, error and forbidden** states, plus read-only states where the spec defines them. Use an error boundary per route.

## Configuration
Use `import.meta.env.VITE_*` only for build-time constants (the app name, feature defaults). The same image runs everywhere. If something must vary per environment, serve it from `GET /api/config` (backend) or `/config.json` (nginx), not from build-time variables. Nothing secret goes in the frontend.

## Accessibility and UX
- Semantic HTML, a `<label>` for every input, `aria-label` on icon buttons, focus trapped in dialogs, and keyboard paths for everything. Contrast at least AA.
- Mobile-first layouts unless the spec says desktop-only, and no horizontal scroll at 360px.
- Lazy-load route modules.

## Contract-first with mocks
While the backend is still stubbed:
- Write MSW handlers in `src/mocks/handlers/<feature>.ts`, typed with the generated types, including the spec's error cases.
- Use them in tests always, and in dev only if `VITE_USE_MOCKS=true`.
- When the real endpoints land, run `npm run gen:api` and the tests again, and remove any temporary shims.

## Tests
- Component and hook tests per FR with visible behaviour. Find elements by role or label, and assert what the user sees.
- Cover form validation, permission-dependent rendering, error-code copy, and empty, error and loading states.
- Add `data-testid` only where there's no accessible name.

## Gate
`cd frontend && npm run typecheck && npm run lint && npm test && npm run build`

## Checklist
- [ ] Only the generated client is used, at same-origin `/api`, and no generated files were edited.
- [ ] Query keys and invalidation are correct after mutations.
- [ ] Loading, empty, error, forbidden and read-only states all exist.
- [ ] Error codes map to copy, and permissions come from the spec.
- [ ] Accessibility basics are met, and the layout works at 360px.
- [ ] Tests cover the FRs, using MSW for the network.
