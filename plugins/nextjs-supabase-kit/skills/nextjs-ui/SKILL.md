---
name: nextjs-ui
description: Use when building or changing Next.js App Router pages and components styled with Tailwind, including forms, permission-aware controls, loading/error/empty states and Supabase Realtime subscriptions.
---

# Next.js + Tailwind UI workflow

Read the spec's pages/UI section and the API response shapes first. Don't invent fields.

## Architecture
- **Server Components by default.** Fetch the first render on the server with the user-scoped Supabase client (or by calling the same `lib/` functions the API uses).
- **Client components** only for interactivity, forms and Realtime. Keep them leaf-level and pass server data in as props.
- **Writes** go through the app's `/api` routes (or Server Actions if the project's CLAUDE.md says so; don't mix the two).
- **Route files**: every data-driven route has `loading.tsx` and `error.tsx`. Use `notFound()` for missing or forbidden resources.
- **Auth-gated pages** redirect to sign-in with `?next=<path>` and return there afterwards.

## Styling
- Tailwind only, mobile-first: design at 360px, then add `sm:`/`md:`/`lg:`.
- Tap targets ≥ 44px (`min-h-11`), no hover-only affordances, and no horizontal page scroll.
- Put shared primitives in `components/ui/` (Button, Input, Dialog/Sheet, Tabs, Toast, EmptyState) and reuse them. Don't restyle each page.
- Support dark mode if the spec asks for it (use the `dark:` variant consistently).

## Forms
- Validate on the client with the same Zod schema as the API where it can be shared (export it from `lib/`).
- Disable submit while it's pending or invalid, show field-level errors, and keep the user's input when a submit fails.
- Map API error codes to human copy in one table (`lib/ui/error-messages.ts`), never inline strings.

## Permissions in the UI
- Build a permission table from the spec (control → condition) and implement it as small pure helpers (`canEdit(user, thing)`).
- Hide or disable what the user can't do, but always handle the 401/403/409 response anyway. The server is the real check.
- Make read-only states (archived, ended, locked) come from one flag and apply everywhere.

## State and data freshness
- After a mutation, use `router.refresh()` or re-query. Only use optimistic updates for cheap, reversible actions, and roll back with a toast on error.
- **Realtime (if used):**
  ```ts
  useEffect(() => {
    const ch = supabase.channel(`scope:${id}`)
      .on('postgres_changes', { event: '*', schema: 'public', table: 't', filter: `parent_id=eq.${id}` }, refreshDebounced)
      .subscribe();
    return () => { supabase.removeChannel(ch); };
  }, [id]);
  ```
  On an event, debounce by about 250ms and refetch; don't patch state by hand. Also refetch on `visibilitychange` and `online`. Child tables without the filter column should bump a parent `updated_at` so events reach subscribers.

## Accessibility
- Use semantic elements, give every input a `<label>`, give icon buttons an `aria-label`, trap focus in dialogs and sheets, make everything keyboard-reachable, and meet colour contrast AA.

## Tests
Write Testing Library tests for form validation, permission-dependent rendering, and any non-trivial display logic (sorting, grouping, formatting).

## Checklist
- [ ] Loading, empty, error, forbidden and read-only states all exist.
- [ ] Each permission state has been checked (per the spec's roles).
- [ ] Works at 360px with no horizontal scroll.
- [ ] Error codes map to copy.
- [ ] Realtime updates appear in a second browser within about a second (if used).
- [ ] `npm run typecheck`, `npm run lint`, `npm test` and `npm run build` pass.
