---
name: build-from-spec
description: Use to build a React + Vite + TypeScript frontend and ASP.NET Core + EF Core backend app in a single repo from SPEC.md, end to end, by coordinating specialised subagents through phases, gates, reviews and commits, finishing with a running deployment on Docker Desktop Kubernetes. Also use to resume from PROGRESS.md.
---

# Build an app from SPEC.md (single repo, local Kubernetes)

You are the **orchestrator**. You plan, delegate, integrate and verify. You write almost no code yourself, so your context stays small enough to hold the whole project. Subagents do the work in fresh contexts and return short summaries.

Stack: TypeScript + React + Vite (`frontend/`), C# + ASP.NET Core + EF Core (`backend/`), Kubernetes manifests and scripts (`deploy/`), Git and GitHub (Actions CI). The app deploys to the **Docker Desktop** Kubernetes cluster on the developer's machine.

## 1. Inputs

| File | Role | If missing |
| --- | --- | --- |
| `SPEC.md` (repo root, or a path the user gives) | **What** to build. Source of truth. | Stop and ask. `templates/SPEC.template.md` is a starting point the user can fill in. |
| `CLAUDE.md` | **How** to build it: layout, commands, rules | `repo-scaffolder` writes it from `templates/CLAUDE.template.md`, filling in details from the spec |
| `PROGRESS.md` | State, requirement table, decisions, known issues | You create it from `templates/PROGRESS.template.md` |

Templates are in this skill's `templates/` folder. Put their absolute paths in the brief of any agent that needs them.

**Requirement IDs.** Use the spec's own IDs if it has them. Otherwise, give each requirement a `REQ-n` ID in PROGRESS.md (ID, summary, spec section, area: `backend`, `frontend`, `deploy`, or `both`).

## 2. Start or resume

1. If `PROGRESS.md` exists, **resume**. Read it, run `git status` and `git log --oneline -15`, then continue from the first unchecked task.
2. Otherwise, **start**:
   - Read the spec once, end to end, and derive the phase plan (§4).
   - Create PROGRESS.md.
   - Git: if the repo is on the default branch, create `build/<short-name>`. If the working tree is dirty, stop and ask.
   - Record the environment in PROGRESS.md:
     - `dotnet --version`, `node --version`, `docker info`
     - `kubectl config get-contexts`, and whether a `docker-desktop` context exists and is reachable (`kubectl --context docker-desktop get nodes`)
3. Read CLAUDE.md in full. After that, read the spec one section at a time, as briefs need it. Use the built-in `Explore` agent for code questions.

## 3. The team

| Agent | Stage | Owns | Skill(s) |
| --- | --- | --- | --- |
| `repo-scaffolder` | Setup | Repo layout, solution and projects, Vite app, tooling, CLAUDE.md | all |
| `ef-data-engineer` | Data | Entities, DbContext, configurations, migrations, seed (serial) | `efcore-data` |
| `aspnet-engineer` | Backend | Domain rules, endpoints, validation, errors, auth, backend tests, OpenAPI file | `aspnet-endpoints`, `openapi-contract` |
| `vite-react-engineer` | Frontend | Routes, components, hooks, generated client, MSW, frontend tests | `vite-react-ui`, `openapi-contract` |
| `e2e-qa-engineer` | Test | Backend integration tests, Playwright E2E, contract checks | all |
| `local-k8s-engineer` | Deploy | Dockerfiles, `deploy/k8s`, deploy scripts, local rollout and smoke test | `docker-desktop-k8s` |
| `github-ci-engineer` | CI | `.github/` workflows, PR template, Dependabot | `github-actions-ci` |
| `pr-reviewer` | Review | Nothing (read-only) | all |
| `spec-verifier` | Acceptance | Nothing (read-only) | — |
| `readme-writer` | Docs | README, docs, spec sync | — |

## 4. Phase plan

| Phase | Content | Agents |
| --- | --- | --- |
| 0 Walking skeleton | Repo layout, backend solution, Vite app, a dev database through `docker compose`, CLAUDE.md → CI skeleton ‖ Dockerfiles and k8s manifests → **deploy the skeleton to Docker Desktop and smoke-test it** (the web page loads, `/api/health/ready` is green) | `repo-scaffolder` → `github-ci-engineer` ‖ `local-k8s-engineer` |
| 1 Foundations | Core schema and first migration ‖ pure domain rules → auth, ProblemDetails, the first OpenAPI file → app shell, auth screens, generated client | `ef-data-engineer` ‖ `aspnet-engineer` → `aspnet-engineer` → `vite-react-engineer` |
| 2…N Features | One phase per milestone or feature area, in dependency order. Each one: contract-first stubs (+ migration if needed) → backend implementation ‖ frontend against mocks → frontend wired to the real API | `aspnet-engineer` (+ `ef-data-engineer`) → `aspnet-engineer` ‖ `vite-react-engineer` → `vite-react-engineer` |
| N+1 Integrate & deploy | Integration and E2E tests → redeploy to Docker Desktop, migration Job, smoke + E2E against `http://localhost:<port>` | `e2e-qa-engineer` → `local-k8s-engineer` → `e2e-qa-engineer` |
| Final | Audit → fixes → CI updated for anything new → docs | `spec-verifier` → owners → `github-ci-engineer` → `readme-writer` |

Rules for planning:
- Keep phases small, with 2–5 tasks and a clear set of IDs.
- Put the schema a phase needs first in that phase.
- **The contract comes first.** The backend commits DTOs and `501` stubs, then regenerates `backend/openapi/api.json`. After that, frontend and backend can work in parallel.

## 5. Writing a brief

```
Goal: <one sentence>. Covers: <IDs>.
Read first: SPEC.md § <sections>; CLAUDE.md; skill(s) <names>; files <paths>.
Owns: <paths>. Anything else: report it, don't change it.
Contract: <DTOs, endpoints, error codes, entities, env vars — copied, not paraphrased>.
Done when: <gate command(s)> + <behaviour to verify>.
Return: ≤15 lines — files changed, verification done, deviations, follow-ups.
```

## 6. Parallelism

- Run agents in parallel only when their Owns sets don't overlap and neither depends on the other's output. Launch them in one message.
- These are **serial-only**: `backend/**/Migrations/`, `backend/openapi/api.json`, `Directory.Packages.props` and `*.csproj` package references, `frontend/package.json` and the lockfile, `frontend/src/api/generated/`, `.github/workflows/`, `deploy/k8s/`.
- In practice, the backend and frontend agents can always run side by side once the contract is committed. The deploy agent runs on its own.

## 7. The gate (end of every phase)

1. Run the gate from CLAUDE.md (backend, frontend and deploy validation). By default:
   ```bash
   (cd backend && dotnet build -warnaserror && dotnet test && dotnet format --verify-no-changes)
   (cd frontend && npm run typecheck && npm run lint && npm test && npm run build)
   kubectl kustomize deploy/k8s/overlays/local > /dev/null
   ```
2. If the contract changed, check that the OpenAPI file and the generated client are up to date and free of breaking changes (`openapi-contract`).
3. Launch `pr-reviewer` with the IDs, the spec sections and the base SHA.
4. Fix Critical and Major findings through their owners. Log Minor ones unless they're quick to fix.
5. Update PROGRESS.md, and commit with `phase N: <summary> (<IDs>)`.
6. **Redeploying.** From phase 1 onwards, have `local-k8s-engineer` redeploy and smoke-test at the end of any phase that changed the backend, frontend or manifests. A green local deployment is part of the gate whenever Docker Desktop Kubernetes is available.

## 8. When things go wrong

- **An agent fails:** re-brief it once, narrower. If it fails again, log it and move on to unblocked work.
- **Spec gap:** decide in the way most consistent with the spec, and log it under Decisions (`readme-writer` syncs the spec later). Ask the user only about irreversible or scope-changing choices.
- **Docker Desktop or Kubernetes unavailable:** keep building. Validate manifests offline (`kubectl kustomize` + `kubeconform`), log it, and report that the local deploy wasn't verified.
- **Context getting long:** make PROGRESS.md current and commit. Re-running this skill resumes the work.

## 9. Boundaries

- **Kubernetes:** agents deploy **only** to the `docker-desktop` context, and only through `deploy/scripts/*.sh`, which refuse any other context. Never touch another context, even read-only.
- **Local data:** never delete the database PVC or namespace (`--wipe-data`), or reset the dev database, without asking.
- **GitHub:** commit on the build branch only. Push, open PRs (`gh pr create --draft`), or change repo settings only if the user asks.
- **Secrets:** never committed. Local secrets live in a git-ignored `deploy/k8s/overlays/local/secrets.env` and `backend` user-secrets, and `*.example` files document them.

## 10. Finish

Stop when the `spec-verifier` shows every requirement Done, the gate is green, the local deployment is up and smoke-tested, and E2E passes (or can't run, with the reason logged). Report:
- what was built
- how to run it: the dev loop, and the local Kubernetes deploy (URL, commands)
- anything not done or blocked, with next steps
- decisions that extend or deviate from the spec
