---
name: supabase-migration
description: Use when adding or changing Supabase tables, columns, views, SQL functions/RPCs, triggers, RLS policies, seed data or generated types in a Next.js + Supabase project.
---

# Supabase migration workflow

Read the spec's data model section first. The schema there is the target.

## Steps

1. `npx supabase migration new <snake_case_description>`, then write SQL in the new file. Never edit a migration that's already committed or applied remotely; add a new one.
2. Write the SQL using the patterns below.
3. `npx supabase db reset` (applies all migrations and the seed), then `npm run db:types`.
4. Verify RLS as each role in the seed (see "Testing RLS"), and record the checklist results.
5. Map new generated types into `lib/types.ts`, then run `npm run typecheck`.
6. If the schema now differs from the spec, log a decision so the spec gets updated.

## Patterns

### Tables
- `id uuid primary key default gen_random_uuid()`, `created_at timestamptz not null default now()`, and `updated_at` where rows change (set by a trigger).
- Foreign keys specify `on delete` explicitly (`cascade` for owned children, `restrict` otherwise).
- Invariants belong in the DB: `check` constraints, unique and partial unique indexes (e.g. `unique (org_id, lower(name))`, `where status = 'active'`), and `generated always as (...) stored` for derived columns.
- Use Postgres enums (`create type x as enum (...)`) for small, stable status sets.
- Always `alter table <t> enable row level security;` with no exceptions.
- Index every foreign key and every column used in RLS predicates.

### Profiles
Create a `profiles` row per `auth.users` row with an `after insert` trigger on `auth.users` (`security definer`). It must tolerate a null email and null metadata (OAuth users and anonymous users).

### Membership helpers (avoid recursive RLS)
```sql
create or replace function is_member(p_parent uuid) returns boolean
language sql stable security definer set search_path = public as $$
  select exists (select 1 from memberships where parent_id = p_parent and user_id = auth.uid())
$$;
```
Reference helpers like this in policies instead of querying the same table inside its own policy.

### Security definer RPCs
Use RPCs for multi-row or privileged writes that plain RLS can't express safely, such as create-with-owner, join-by-code, state transitions and transfers.
- `security definer set search_path = public`.
- The first statement rejects anonymous callers where appropriate: `if auth.uid() is null then raise exception 'NOT_AUTHENTICATED'; end if;`
- **Re-check every permission inside.** Definer rights bypass RLS.
- Raise app error codes as the message so the API maps them in one place: `raise exception 'NOT_OWNER' using errcode = 'P0001';`
- `revoke execute on function f from public; grant execute on function f to authenticated;`

### Anonymous (guest) auth, if the spec uses it
Anonymous users have the `authenticated` role, with `(auth.jwt() ->> 'is_anonymous')::boolean = true`. Only add `is_anonymous` checks where the spec treats guests differently. Enable it in `supabase/config.toml` (`[auth] enable_anonymous_sign_ins = true`).

### Realtime, if used
Add a table to the publication (`alter publication supabase_realtime add table <t>;`) only if the UI subscribes to it. Realtime respects RLS select policies.

### Seed
`supabase/seed.sql` must contain a user for every role the spec defines (e.g. owner, member, non-member, guest), plus data in each important state (e.g. empty, active, archived). Reviewers and E2E rely on this.

## Testing RLS
```sql
begin;
set local role authenticated;
set local request.jwt.claims = '{"sub":"<seed-user-uuid>","role":"authenticated"}';
select ... ;          -- expect rows / no rows
insert ... ;          -- expect success / RLS violation
rollback;
```

## RLS checklist
- [ ] Every table has RLS enabled and at least one select policy (or is deliberately private).
- [ ] Non-members can't read or write other groups' rows.
- [ ] Update policies restrict **which columns** can change (use RPCs or column grants where needed).
- [ ] Tables written only through RPCs have no direct insert/update/delete policies.
- [ ] Every security definer function re-checks permissions and sets `search_path`.
- [ ] State rules (e.g. archived or ended is read-only) are enforced in the DB, not just the UI.
- [ ] Realtime tables are in the publication, and their select policies are correct.

## Remote
Never run `supabase db push` or `supabase link` against a remote project unless the user asks. See the `github-vercel-delivery` skill.
