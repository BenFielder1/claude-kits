# react-dotnet-k8s-kit

A spec-driven delivery kit for Claude Code on this stack: **TypeScript + React + Vite** frontends, **C# / .NET** services, **GitLab**, and **Kubernetes deployed by ArgoCD** from a GitOps repo. It's built for real-world layouts, where work can touch one component or several, across separate folders or repos.

## Flow

```
Jira item ──spec-from-jira──▶ specs/<KEY>/SPEC.md ──orchestrate-work──▶ branches per repo ─(you ask)─▶ draft MRs ─(merge)─▶ ArgoCD syncs
                 │                                         │
           asks you only the                  discover → contract & data → slices
           questions that matter              → integrate → CI/GitOps → audit → docs
```

## What's inside

| Type | Name | Purpose |
| --- | --- | --- |
| Skill | `spec-from-jira` | Fetches a Jira item (and its links, comments and Confluence pages) through MCP, checks the code, asks targeted questions, and writes `SPEC.md` |
| Skill | `orchestrate-work` | Scopes to the active components, plans phases, briefs agents, runs per-component gates, commits per repo, and resumes from `PROGRESS.md` |
| Skill | `dotnet-api` | ASP.NET Core endpoints, domain rules, ProblemDetails, auth, config, health, tests |
| Skill | `dotnet-data` | EF Core model, migrations, expand/contract for rolling deploys |
| Skill | `react-vite-ui` | Conventions-first React, generated client, runtime config for Kubernetes, MSW, tests |
| Skill | `api-contract` | OpenAPI as the cross-repo handshake, breaking-change rules, client generation |
| Skill | `gitlab-delivery` | Branches, commits, `glab` MRs, `.gitlab-ci.yml`, images, GitOps handoff |
| Skill | `k8s-argocd-gitops` | Dockerfiles, Helm/Kustomize, ArgoCD, workload checklist, offline validation |
| Agent | `service-scaffolder` | New component (golden path), or document mode (writes CLAUDE.md for an existing one) |
| Agent | `dotnet-data-engineer` | Entities, migrations, data access (one at a time per service) |
| Agent | `dotnet-engineer` | Backend behaviour and the OpenAPI contract (including contract-first stubs) |
| Agent | `react-engineer` | Frontend behaviour, generated client, MSW mocks |
| Agent | `qa-engineer` | Integration (Testcontainers), contract and E2E tests |
| Agent | `gitlab-ci-engineer` | Pipelines, Dockerfiles, image tagging, handoff |
| Agent | `platform-engineer` | GitOps repo changes, dev first, validated offline |
| Agent | `mr-reviewer` | Read-only review at each gate |
| Agent | `acceptance-auditor` | Read-only FR-by-FR audit (with Deferred for out-of-scope work) |
| Agent | `tech-writer` | Spec sync, docs, MR description drafts |

Templates: `skills/orchestrate-work/templates/` (WORKSPACE, PROGRESS, component CLAUDE.md) and `skills/spec-from-jira/templates/SPEC.template.md`.

All names are different from the Next.js kit's, so both kits can be installed at the same time.

## Setup (once per workspace)

1. **Lay out the repos** under one parent folder (recommended), e.g. `~/work/orders/{orders-api,web-app,platform-gitops}`.
2. **Create `WORKSPACE.md`** in that parent folder (or in a small "workspace" repo) from the template. It lists components, paths, gates, contracts and environments, and the specs live next to it in `specs/`.
3. **Connect the Atlassian/Jira MCP server** in Claude Code, following your organisation's setup, so `spec-from-jira` can read tickets. Without it, the skill asks you to paste the ticket instead.
4. **Optional tools** for full verification: Docker (Testcontainers, local stack), `glab` (authenticated), `helm`, `kustomize`, `kubeconform`, `oasdiff`.
5. **Start Claude Code in the parent folder**, or add the repos you need with `--add-dir` or `/add-dir`.

## Install

**As a plugin** (recommended): put this folder in a Git repo or a local marketplace and install it with `/plugin`. The manifest is in `.claude-plugin/plugin.json`. Names may appear with the `react-dotnet-k8s-kit:` prefix.

**User-level** (all projects):
```bash
cp -r agents/* ~/.claude/agents/ && cp -r skills/* ~/.claude/skills/
```

**Project-level** (one workspace; can be customised):
```bash
mkdir -p .claude && cp -r agents skills .claude/
```

## Use

```
Use spec-from-jira for PROJ-123
→ answers a few questions → specs/PROJ-123/SPEC.md

Use orchestrate-work with specs/PROJ-123/SPEC.md
Use orchestrate-work with specs/PROJ-123/SPEC.md — backend only
Use orchestrate-work with specs/PROJ-123/SPEC.md — only FR-4 to FR-6 in web
```
Re-running the same command resumes from `PROGRESS.md`.

## Guardrails (built in)

- It works on `feature/<KEY>-<slug>` branches, never on protected branches, and stops if a working tree is dirty.
- It pushes, creates MRs or comments on Jira only when you ask.
- It **never changes a cluster**: no `kubectl` writes, `helm install`, or `argocd sync`. Deployment happens when you merge GitOps MRs.
- It doesn't change prod values unless the spec requires it, and then flags them for human approval.
- It writes no secret values. It lists what a human must create.
- Out-of-scope components are never edited. Work they need is recorded as cross-component follow-ups.
