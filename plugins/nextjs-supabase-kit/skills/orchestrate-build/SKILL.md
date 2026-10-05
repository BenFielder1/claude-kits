---
name: orchestrate-build
description: Use to build a Next.js + Supabase app (or a large feature) end to end from a spec by coordinating specialised subagents through phases, gates, reviews and commits. Also use to resume a build from PROGRESS.md.
---

# Orchestrate a spec-driven build

You are the **orchestrator**. You plan, delegate, integrate and verify. You write almost no code yourself, so your context stays small enough to hold the whole project. Subagents do the work in fresh contexts and hand back short summaries.

Stack assumed: Next.js App Router, TypeScript (strict), Tailwind CSS, Next.js route handlers, Supabase (Postgres, Auth, Realtime), GitHub, Vercel.

## 1. Inputs

| File | Role | If missing |
| --- | --- | --- |
| `SPEC.md` (or the path the user gives) | **What** to build. Source of truth. | Stop and ask for it. Don't invent a product. |
| `CLAUDE.md` | **How** to build it: rules, layout, commands | `project-scaffolder` creates it from `templates/CLAUDE.template.md` and fills it from the spec |
| `PROGRESS.md` | Build state, decisions, known issues | You create it from `templates/PROGRESS.template.md` |

**Templates** are in this skill's `templates/` folder, next to this file. Subagents can't find it on their own, so put the absolute path in any brief that needs one.

**Requirement IDs.** Briefs, reviews and commits all refer to requirements by ID. If the spec already has IDs (FR-1, REQ-3, …), use them. If it doesn't, give every distinct requirement a `REQ-n` ID in a table in `PROGRESS.md` (ID, one-line summary, spec section), and use those.

## 2. Start or resume

1. If `PROGRESS.md` exists, **resume**. Read it, run `git log --oneline -15` and `git status`, then continue from the first unchecked task.
2. Otherwise, **start**:
   - Read `SPEC.md` once, end to end, and derive the phase plan (§4).
   - Create `PROGRESS.md` with the requirement table and the phase checklist.
   - Git: if you're on the default branch, create `build/<short-project-name>` and work there.
3. Read `CLAUDE.md` in full. After that, read the spec only by section when writing a brief. Use the built-in `Explore` agent for questions about existing code rather than reading files yourself.

## 3. The team

| Agent | Stage | Use for | Writes |
| --- | --- | --- | --- |
| `project-scaffolder` | Setup | Repo bootstrap, tooling, config, CLAUDE.md | Config, layout |
| `db-engineer` | Data | Migrations, RLS, SQL functions, seed, generated types | `supabase/**`, `lib/database.types.ts` |
| `domain-logic-engineer` | Core logic | Pure business rules and algorithms, tests first | `lib/<domain>/**` |
| `api-engineer` | Backend | Route handlers, auth helpers, error mapping | `app/api/**`, `lib/api/**` |
| `ui-engineer` | Frontend | Pages, components, forms, Realtime | `app/**` (not `api`), `components/**` |
| `test-engineer` | Test | Playwright E2E, coverage gaps | `e2e/**`, test files |
| `devops-engineer` | Delivery | GitHub Actions CI, Vercel config, env docs, migration deploy | `.github/**`, `vercel.json` |
| `code-reviewer` | Review | Independent review at each phase gate | Nothing (read-only) |
| `spec-auditor` | Acceptance | Grades every requirement with evidence | Nothing (read-only) |
| `docs-writer` | Docs | README, SPEC sync from decisions | Docs only |

Each agent loads its matching skill: `supabase-migration`, `nextjs-api-route`, `nextjs-ui`, `domain-logic` or `github-vercel-delivery`.

## 4. Deriving the phase plan

Use this skeleton, and fill the middle from the spec's milestones or feature areas:

| Phase | Content | Agents |
| --- | --- | --- |
| 0 Scaffold | Next.js + tooling + Supabase init + CLAUDE.md + CI skeleton | `project-scaffolder` → `devops-engineer` |
| 1 Foundations | Core schema, RLS and auth ‖ pure domain logic → auth pages and `lib/api` helpers | `db-engineer` ‖ `domain-logic-engineer` → `api-engineer` + `ui-engineer` |
| 2…N Features | One phase per milestone or feature area, in dependency order. Each: schema delta (if any) → API → UI | `db-engineer` → `api-engineer` → `ui-engineer` |
| N+1 Harden & accept | Polish, error states, accessibility ‖ E2E → audit → fixes → CI/deploy readiness → docs | `ui-engineer` ‖ `test-engineer` → `spec-auditor` → owners → `devops-engineer` → `docs-writer` |

Guidelines for feature phases:
- Keep phases small, with 2–5 tasks and a clear set of requirement IDs each.
- Order by dependency: things you create before things that use them (e.g. teams before invites, orders before refunds).
- Build **all** of the schema a phase needs at the start of that phase, so API and UI agents don't wait on migrations mid-phase.
- Pure logic with no DB dependency (pricing, scoring, scheduling, validation rules) goes to `domain-logic-engineer` early, in parallel with schema work.

## 5. Writing a brief

Every subagent starts with nothing. The brief is everything it knows.

```
Goal: <one sentence>. Covers: <requirement IDs>.
Read first: SPEC.md § <sections>; skill <name>; existing files <paths>.
Owns: <files/folders it may create or edit>. Anything else: report it, don't change it.
Contract: <types, signatures, routes, tables, response shapes it must provide or consume — copied, not paraphrased>.
Done when: <exact commands that must pass> and <behaviour to verify>.
Return: ≤15 lines — files changed, what you verified and how, deviations from spec, follow-ups.
```

A good brief is narrow and concrete. For example: "Build `POST /api/teams` and `POST /api/teams/[id]/invites` per SPEC § API rows 1–2, calling RPCs `create_team` and `invite_member`." A brief like "Build the teams backend" is too vague.

## 6. Parallelism

- Run agents in parallel only if their **Owns** sets don't overlap and neither needs the other's output. Launch them in one message.
- These are **serial-only** (one agent at a time, team-wide): `supabase/migrations/`, `lib/database.types.ts`, `lib/types.ts`, `lib/api/errors.ts`, `package.json` and the lockfile, `.github/workflows/`.
- If parallel tasks share a type, add it to `lib/types.ts` first and commit, then fan out.
- Use `isolation: "worktree"` for risky parallel work, and merge it yourself.

## 7. The phase gate

1. Run the gate commands from CLAUDE.md yourself. The default is `npm run typecheck && npm run lint && npm test && npm run build`.
2. Launch `code-reviewer` with the phase's IDs, spec sections and base commit.
3. Send a fix brief to the owning agent for each Critical or Major finding. Log Minor findings under Known issues unless they're quick fixes.
4. Re-run step 1. If there were Critical fixes, review again.
5. Update `PROGRESS.md`, then `git commit -m "phase N: <summary> (<IDs>)"`. Record the new SHA as the next phase's base.

## 8. When things go wrong

- **An agent fails or returns something unusable:** re-brief it once with a narrower scope and the error. If it fails again, log it under Known issues and move on to unblocked work.
- **A change is needed outside an agent's Owns:** brief the owning agent, or make a trivial one-line fix yourself.
- **The spec has a gap or contradiction:** pick the option most consistent with the rest of the spec and log it under Decisions (`docs-writer` syncs SPEC.md later). Ask the user only about irreversible or product-changing choices.
- **The environment blocks you** (no Docker, no credentials, no network): log it, build everything that doesn't depend on it, and report it at the end.
- **Your context is getting long:** make sure PROGRESS.md is current and commit. A fresh session running this skill resumes from there.

## 9. Git, GitHub and Vercel boundaries

- Commit at every gate on the build branch. Never commit secrets or `.env*.local`.
- Push, open PRs, link Vercel, run `supabase db push` to a remote project, or deploy **only if the user asks**. By default, `devops-engineer` makes all of these ready but doesn't run them.
- If the user asks for a PR at the end, use `gh pr create --draft` and write a body that lists the phases, the requirement coverage from the audit, and any known issues.

## 10. Finish

Stop when `spec-auditor` shows every requirement as Done, the last gate passes, and E2E passes (or can't run, with the reason logged). Report:
- what was built (a few lines)
- anything not done or blocked, with the next step for each
- decisions that deviate from or extend the spec
- how to run it locally in 3–4 commands, and what's needed to deploy
