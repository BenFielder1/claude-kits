---
name: domain-logic
description: Use when implementing a project's core business rules or algorithms (pricing, scoring, ranking, scheduling, eligibility, state machines, validation) as pure, tested TypeScript modules in lib/.
---

# Pure domain logic

Business rules are the part of the app most likely to be subtly wrong and least likely to be caught by clicking around. Keep them isolated, pure and heavily tested.

## Where and what
- `lib/<domain>/` has one file per concept (e.g. `pricing.ts`, `ranking.ts`, `schedule.ts`), each with a `*.test.ts` alongside.
- Put the exported types (inputs and outputs) at the top of each file, or in `lib/<domain>/types.ts`.
- Only domain types go in and out. Never DB rows, `Request`/`Response` or Supabase types.

## Rules
1. **Pure.** No Supabase, `fetch`, `fs`, env vars, `Date.now()`, `Math.random()` or globals. Time, randomness seeds and config are parameters.
2. **Deterministic.** The same input always gives the same output, including ordering. Define explicit, stable tie-breakers (e.g. by timestamp, then by id).
3. **Exact arithmetic where it matters.** Use integer minor units for money, and compare ratios by cross-multiplication rather than floats. Only format for display in the UI.
4. **One source of truth for constants.** If a constant is duplicated in SQL (e.g. a lookup table in a generated column), add a test that reads the migration and checks the two match.
5. **Total functions.** Handle empty input, a single item, maximum sizes and invalid values explicitly, either by throwing a typed error or by returning a discriminated result. Never return `undefined` silently.
6. **Small public surface.** Export what the API and UI need, and keep helpers private.

## Workflow (tests first)
1. Turn every rule and example in the spec into a test case, and cite the spec section in the `describe` name.
2. Add edge cases: empty, single, ties, limits (min and max), invalid input, and large input.
3. Add a determinism test: shuffle the input 20 times and check the output is identical.
4. Implement until green, then refactor for clarity.
5. Run `npm test -- lib/<domain>` and `npm run typecheck`.

## Documenting ambiguity
If the spec doesn't define a case (e.g. how ties break), choose the most consistent behaviour, encode it in a test named `"assumption: …"`, and report it so the orchestrator logs a decision.
