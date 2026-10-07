# react-dotnet-local-kit

A spec-driven build kit for Claude Code. It builds **one repo** from **one `SPEC.md`**:
- **Frontend:** TypeScript + React + Vite
- **Backend:** C# + ASP.NET Core
- **Database:** Microsoft EF Core (`dotnet-ef`), SQL Server by default, or PostgreSQL
- **Version control:** Git, GitHub and GitHub Actions
- **Deployment:** the local **Docker Desktop Kubernetes** cluster

```
SPEC.md ──build-from-spec──▶ walking skeleton deployed to Docker Desktop
                              → foundations → feature phases (contract-first, backend ‖ frontend)
                              → integrate + redeploy + E2E on localhost → audit → docs
```

## What's inside

| Type | Name | Purpose |
| --- | --- | --- |
| Skill | `build-from-spec` | Orchestrator: plans phases from the spec, briefs agents, gates, redeploys, commits, resumes from `PROGRESS.md` |
| Skill | `aspnet-endpoints` | Domain rules, minimal APIs, ProblemDetails, Identity/cookie auth, health, tests |
| Skill | `efcore-data` | EF Core model, `dotnet ef` migrations, seed, migration bundle for Kubernetes |
| Skill | `vite-react-ui` | Generated client, TanStack Query, forms, permissions, states, MSW, tests |
| Skill | `openapi-contract` | `backend/openapi/api.json` → `frontend/src/api/generated`, breaking-change rules |
| Skill | `github-actions-ci` | Branch/PR conventions, CI jobs, freshness checks, Dependabot |
| Skill | `docker-desktop-k8s` | Dockerfiles, Kustomize, DB StatefulSet, migration Job, deploy/smoke/teardown scripts |
| Agent | `repo-scaffolder` | Creates the repo and CLAUDE.md (or documents an existing repo) |
| Agent | `ef-data-engineer` | Persistence and migrations (one at a time) |
| Agent | `aspnet-engineer` | Backend behaviour and the contract |
| Agent | `vite-react-engineer` | Frontend behaviour, mock mode → real mode |
| Agent | `e2e-qa-engineer` | Integration, contract and E2E tests |
| Agent | `local-k8s-engineer` | Containers, manifests, deploy to `docker-desktop` |
| Agent | `github-ci-engineer` | GitHub Actions and repo hygiene |
| Agent | `pr-reviewer` | Read-only review at each gate |
| Agent | `spec-verifier` | Read-only acceptance audit, including the live local deploy |
| Agent | `readme-writer` | README, config docs, spec sync |

Templates live in `skills/build-from-spec/templates/`: `CLAUDE.template.md`, `PROGRESS.template.md`, and an optional `SPEC.template.md` to start a spec from. No names clash with the other kits.

## Repo shape it produces

```
backend/   (<App>.Api · Domain · Infrastructure · tests · openapi/api.json)
frontend/  (src/features · src/api/generated · e2e)
deploy/    (k8s/base · k8s/overlays/local · scripts)
docker-compose.yml (dev DB) · .github/workflows/ci.yml · SPEC.md · PROGRESS.md · CLAUDE.md
```
The web container serves the SPA and proxies `/api` to the API, so the app is same-origin (cookies work, no CORS) at `http://localhost:8080` by default.

## Prerequisites (on your machine)
- .NET SDK (LTS), Node LTS, and Docker Desktop with **Kubernetes enabled**
- Optional: `kubeconform`, `oasdiff`, `actionlint`, and the GitHub CLI (only needed for PRs)

## Install
- **Plugin:** `/plugin install react-dotnet-local-kit@<your-marketplace>`
- **User-level:** `cp -r agents/* ~/.claude/agents/ && cp -r skills/* ~/.claude/skills/`
- **Project-level:** `mkdir -p .claude && cp -r agents skills .claude/`

## Use
1. Put `SPEC.md` at the repo root. To start one, copy `SPEC.template.md` from `skills/build-from-spec/templates/`.
2. In Claude Code, in the repo: **"Use the build-from-spec skill to build this app from SPEC.md."**
3. To resume, send the same message again.

## Guardrails
- It works on a `build/<name>` branch, and stops if the working tree is dirty.
- It deploys **only** to the `docker-desktop` context, through scripts that refuse anything else.
- It never wipes the local database without your say-so (`--wipe-data` plus a confirmation).
- Secrets live in git-ignored files (`.env`, `deploy/k8s/overlays/local/secrets.env`) and in user-secrets. Only `*.example` files are committed.
- It pushes, opens PRs or changes GitHub settings only when you ask.
