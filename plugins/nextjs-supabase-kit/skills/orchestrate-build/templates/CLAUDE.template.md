# <Project name>

<2–3 sentences: what the app does and for whom, taken from the spec.>

**`SPEC.md` is the source of truth.** Read the relevant section before building a feature. If code and spec disagree, flag it rather than silently picking one. Requirement IDs are referenced in commits and PRs.

## Stack

- Next.js (App Router) + TypeScript (strict)
- Tailwind CSS
- Next.js route handlers in `app/api/**/route.ts`
- Supabase: Postgres, Auth (<methods from spec: email/password, OAuth, anonymous>), Realtime (<if used>)
- `@supabase/ssr` for cookie sessions · Zod for validation
- Vitest (unit/component) · Playwright (E2E)
- GitHub (repo, Actions CI) · Vercel (hosting, preview deploys)

## Commands

```bash
npm run dev            # dev server
npm run build          # production build
npm run lint           # ESLint
npm run typecheck      # tsc --noEmit
npm test               # Vitest
npm run test:e2e       # Playwright
npx supabase start     # local Supabase (Docker)
npx supabase db reset  # re-apply migrations + seed
npm run db:types       # regenerate lib/database.types.ts
```

**Gate:** `npm run typecheck && npm run lint && npm test && npm run build`

## Project layout

```
app/                   routes (pages) and app/api route handlers
components/            UI components (feature folders + ui/)
lib/
  supabase/            server.ts, client.ts (user-scoped clients)
  api/                 auth.ts (requireUser), errors.ts (codes + mapping)
  <domain>/            PURE business logic + tests
  types.ts             domain types
  database.types.ts    GENERATED — never edit
supabase/migrations/   SQL migrations · supabase/seed.sql
e2e/                   Playwright tests
.github/workflows/     CI
middleware.ts          session refresh (proxy.ts on Next.js 16+)
```

## Rules that must not be broken

1. Never use the Supabase service-role key in app code. RLS plus `security definer` RPCs enforce permissions.
2. Every table has RLS enabled. Privileged multi-step writes go through RPCs that re-check permissions.
3. Pure domain logic lives in `lib/<domain>/`, with no I/O, and is deterministic and unit-tested.
4. Validate every API input with Zod. Errors are returned as `{ error: { code, message } }` using codes in `lib/api/errors.ts`.
5. No secrets in git. Env vars are documented in `.env.example`.
<!-- Add project-specific invariants from the spec: permissions, limits, formulas, derived vs stored data. -->

## Conventions

- Server Components by default. Use `"use client"` only for interactivity or Realtime.
- Tailwind only, mobile-first, tap targets ≥ 44px, accessible labels on every control.
- Generated DB types map to domain types in `lib/types.ts` (camelCase).
- Files kebab-case, components PascalCase, SQL snake_case.

## Workflow

Use the `orchestrate-build` skill for multi-phase work. Stack skills: `supabase-migration`, `nextjs-api-route`, `nextjs-ui`, `domain-logic`, `github-vercel-delivery`.

## Definition of done

- Gate passes. New logic has tests. Schema changes have a migration, regenerated types, and reviewed RLS.
- UI checked at 360px and in each permission state.
- If behaviour differs from SPEC.md, SPEC.md is updated in the same change.
