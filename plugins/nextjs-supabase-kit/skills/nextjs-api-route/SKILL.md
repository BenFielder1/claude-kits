---
name: nextjs-api-route
description: Use when adding or changing Next.js App Router route handlers (app/api/**/route.ts) backed by Supabase, including Zod validation, auth, RPC calls and error mapping.
---

# Next.js API route workflow

Read the spec's API section for the method, path, body, response and permissions.

## Shared helpers (create once, in `lib/api/`)

```ts
// lib/api/errors.ts
export const ERROR_STATUS = {
  NOT_AUTHENTICATED: 401,
  INVALID_INPUT: 400,
  NOT_FOUND: 404,
  FORBIDDEN: 403,
  CONFLICT: 409,
  INTERNAL: 500,
  // + project codes from the spec, e.g. NAME_TAKEN: 409, NOT_OWNER: 403
} as const;
export type ErrorCode = keyof typeof ERROR_STATUS;

export function apiError(code: ErrorCode, message?: string, details?: unknown) {
  return NextResponse.json({ error: { code, message: message ?? code, details } }, { status: ERROR_STATUS[code] });
}

// The ONLY place Postgres errors are interpreted.
export function fromPostgresError(e: PostgrestError) {
  if (e.code === 'P0001' && e.message in ERROR_STATUS) return apiError(e.message as ErrorCode);
  if (e.code === '23505') return apiError('CONFLICT', 'Already exists'); // refine by constraint name
  if (e.code === '42501' || e.code === 'PGRST116') return apiError('NOT_FOUND');
  console.error(e);
  return apiError('INTERNAL', 'Something went wrong');
}

// lib/api/auth.ts
export async function requireUser(supabase: SupabaseClient) {
  const { data: { user } } = await supabase.auth.getUser();   // not getSession(): verifies the JWT
  return user; // anonymous users are users
}
```

## Handler shape

```ts
// app/api/<resource>/[id]/route.ts
import { NextResponse } from 'next/server';
import { z } from 'zod';
import { createClient } from '@/lib/supabase/server';
import { requireUser } from '@/lib/api/auth';
import { apiError, fromPostgresError } from '@/lib/api/errors';

const Params = z.object({ id: z.string().uuid() });
const Body = z.object({ name: z.string().trim().min(1).max(100) });

export async function PATCH(req: Request, ctx: { params: Promise<{ id: string }> }) {
  const params = Params.safeParse(await ctx.params);
  if (!params.success) return apiError('INVALID_INPUT', undefined, params.error.flatten());

  const supabase = await createClient();
  if (!(await requireUser(supabase))) return apiError('NOT_AUTHENTICATED');

  const body = Body.safeParse(await req.json().catch(() => null));
  if (!body.success) return apiError('INVALID_INPUT', undefined, body.error.flatten());

  const { data, error } = await supabase.from('things')
    .update({ name: body.data.name }).eq('id', params.data.id).select().single();
  if (error) return fromPostgresError(error);

  return NextResponse.json({ thing: toThing(data) });
}
```

## Rules
1. **User-scoped client only** (`lib/supabase/server.ts`). Never the service-role key. RLS and RPCs are the real permission check.
2. **Validate params, query and body with Zod** before touching the DB. Normalise input (trim, case) in the schema.
3. **Privileged writes go through RPCs** (`supabase.rpc('fn', args)`). Don't reimplement their checks in TypeScript, and never add a looser path.
4. **Map errors only via `fromPostgresError`.** Add new codes to `ERROR_STATUS` and to the UI's error copy together.
5. **Return domain types** (camelCase mappers from `lib/types.ts`), never raw rows.
6. **Business rules come from `lib/<domain>/`.** Routes fetch the data, call the pure function and return the result.
7. **Keep handlers thin**, around 60 lines at most. Move logic into `lib/`.
8. **Webhooks and cron** (if the spec has them) verify a signature or secret header, are idempotent, and are the only place a server-only secret may be used. Document every such exception in CLAUDE.md.
9. **Caching**: authenticated handlers are dynamic. Don't add `revalidate` or static caching to user-specific routes.

## Tests
For each route, write a Vitest test with a mocked Supabase client covering success, invalid input, unauthenticated, and one mapped DB error. If local Supabase is running, also call each route once with curl as each seeded role.

## Checklist
- [ ] Method, path, body and response match the spec.
- [ ] Zod covers params, query and body.
- [ ] Every error path returns a known code.
- [ ] Works for each role in the seed, including guests if the spec has them.
- [ ] `npm run typecheck`, `npm run lint` and `npm test` pass.
