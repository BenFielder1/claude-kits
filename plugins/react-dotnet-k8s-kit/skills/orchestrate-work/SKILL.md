---
name: orchestrate-work
description: Use to deliver a SPEC.md (often produced by spec-from-jira) across a React/Vite frontend, .NET backend services and a Kubernetes/ArgoCD GitOps repo — in one component or several folders/repos — by coordinating specialised subagents through discovery, phases, gates, reviews and per-repo commits. Also use to resume from PROGRESS.md.
---

# Orchestrate delivery across components

You are the **orchestrator**. You plan, delegate, integrate and verify. You write almost no code yourself, so your context stays small enough to hold every component in scope. Subagents do the work in fresh contexts and return short summaries.

Stack: TypeScript + React + Vite (frontend), C# / .NET (backend services), Git on GitLab, Kubernetes deployed by ArgoCD from a GitOps repo.

## 1. Inputs

| File | Role | If missing |
| --- | --- | --- |
| Spec (`<specsDir>/<KEY>/SPEC.md`, or the path the user gives) | **What** to build, with FRs, components and acceptance criteria | Suggest `spec-from-jira <KEY>` and stop |
| `WORKSPACE.md` (current directory or a parent) | **Where**: the components, their paths, gates, contracts and environments | Single repo: create a minimal one from `templates/WORKSPACE.template.md` and confirm it with the user. Several repos: ask for the paths. |
| Each component's `CLAUDE.md` | **How** that component is built: conventions, commands, rules | `service-scaffolder` in **document mode** writes it before any work in that component |
| `PROGRESS.md` (next to the spec) | State, decisions, known issues | Create it from `templates/PROGRESS.template.md` |

Templates are in this skill's `templates/` folder. Put their absolute paths in any brief that needs them.

## 2. Scope: work only on what's asked

The **active components** are the components in the spec's frontmatter, narrowed by the user's request. For example, "backend only", "just the web app" or "only FR-3 to FR-5" all narrow the scope. Then:

- **Never change a component outside the active set.** If a requirement needs one, log it under Known issues as a cross-component dependency and carry on.
- **Check access.** Every active component's path must be readable and writable in this session. If one isn't (another repo not added), tell the user to add it (for example by starting Claude Code with `--add-dir <path>`, or using `/add-dir`) and continue with the components you can reach.
- **Partial-stack rules:**
  - *Frontend without backend:* build against the provider's **existing** contract file. For new endpoints in the spec, use contract stubs written by the frontend in mock form (MSW handlers that match §5 of the spec), and log that the backend still has to deliver them.
  - *Backend without frontend:* deliver the API and its contract, then report the client regeneration the frontend needs.
  - *GitOps without app changes:* a manifest-only change, validated offline.

## 3. Start or resume

1. If `PROGRESS.md` exists next to the spec, **resume**. Read it, run `git status` and `git log --oneline -10` in each active component, then continue from the first unchecked task.
2. Otherwise, **start**:
   - Read the spec once, end to end, and `WORKSPACE.md`.
   - Create `PROGRESS.md` with the FR table (ID, component, phase, status), the active components, and the phase checklist.
   - **Git, per active component:** if it's on the default branch with a clean tree, create `feature/<KEY>-<slug>`. If the tree is dirty, stop and ask; never stash or discard someone's work.
3. Use the built-in `Explore` agent for code questions instead of reading files yourself.

## 4. The team

| Agent | Stage | Works in | Skill(s) |
| --- | --- | --- | --- |
| `service-scaffolder` | Setup / discovery | Any component (new or undocumented) | all relevant |
| `dotnet-data-engineer` | Data | Backend service: entities, EF Core migrations, data access | `dotnet-data` |
| `dotnet-engineer` | Backend | Backend service: domain, application, API endpoints, tests | `dotnet-api`, `api-contract` |
| `react-engineer` | Frontend | Frontend: routes, components, data hooks, tests | `react-vite-ui`, `api-contract` |
| `qa-engineer` | Test | Integration, contract and E2E tests across components | all relevant |
| `gitlab-ci-engineer` | CI | `.gitlab-ci.yml`, Dockerfiles, MR templates in app repos | `gitlab-delivery` |
| `platform-engineer` | Deploy | GitOps repo: Helm/Kustomize, ArgoCD apps, env config | `k8s-argocd-gitops` |
| `mr-reviewer` | Review | Read-only, at each gate | all relevant |
| `acceptance-auditor` | Acceptance | Read-only, FR-by-FR with evidence | — |
| `tech-writer` | Docs | READMEs, runbooks, spec sync | — |

Every brief names the **component path**, and the agent works only inside it.

## 5. Phase plan

Build the plan from the active components and the spec's §11 delivery hints. Skip any phase with nothing in scope.

| Phase | Content | Agents |
| --- | --- | --- |
| 0 Discover | For each active component: make sure CLAUDE.md exists (`service-scaffolder` document mode if not), record its conventions, and run the **baseline gate** on the untouched branch, logging any failures that existed before you started | `service-scaffolder`, you |
| 1 Contract & data | Backend: agree the API contract first (DTOs and endpoint stubs, then regenerate the OpenAPI file), then entities and migrations | `dotnet-engineer` (contract) → `dotnet-data-engineer` |
| 2…N Feature slices | Group FRs into vertical slices. For each slice: backend logic and endpoints ‖ frontend against the agreed contract with mocks → regenerate the client against the real contract → wire it up | `dotnet-engineer` ‖ `react-engineer` |
| N+1 Integrate | Integration tests (API with a real DB), E2E against the local stack, contract-compatibility check | `qa-engineer` |
| N+2 Delivery | CI changes, image build, and GitOps manifests/config per environment (dev first) | `gitlab-ci-engineer` → `platform-engineer` |
| Final | Audit → fixes → docs → MR descriptions | `acceptance-auditor` → owners → `tech-writer` |

**The contract first** is what lets frontend and backend run in parallel. Once the contract change is committed in the backend (stubs return `501`), the frontend can build against the generated client with mock handlers while the backend fills in the stubs.

## 6. Writing a brief

```
Goal: <one sentence>. Covers: <FR IDs>. Jira: <KEY>.
Component: <name> at <absolute path>. Work only inside it.
Read first: <spec path> § <sections>; <component>/CLAUDE.md; skill(s) <names>; files <paths>.
Owns: <paths within the component>. Anything else: report it, don't change it.
Contract: <DTOs, endpoints, error codes, types, events — copied from the spec or the contract file>.
Done when: <the component's gate command> + <behaviour to verify>.
Return: ≤15 lines — files changed, verification done, deviations, follow-ups (including cross-component ones).
```

## 7. Parallelism

- Run agents in parallel only when they're in **different components**, or in the same component with **disjoint Owns** and no dependency between them. Launch parallel agents in one message.
- These are **serial-only** within a component: EF Core migrations and the model snapshot, the committed OpenAPI file, `Directory.Packages.props` / `*.csproj` package references, `package.json` and the lockfile, `.gitlab-ci.yml`, and GitOps env values files.
- Use `isolation: "worktree"` when two agents need the same repo at the same time.

## 8. The gate (per phase, per touched component)

1. Run the component's gate from WORKSPACE.md or its CLAUDE.md. Typical gates:
   - **Backend:** `dotnet build -warnaserror && dotnet test && dotnet format --verify-no-changes`
   - **Frontend:** `npm ci && npm run typecheck && npm run lint && npm test && npm run build`
   - **GitOps:** `helm lint` / `kustomize build` plus `kubeconform` for each changed environment
   Compare against the baseline. Only **new** failures block.
2. If the contract changed, run the breaking-change check (see `api-contract`). Breaking changes need an explicit spec decision.
3. Launch `mr-reviewer` with the FR IDs, the spec sections, and the component paths with their base SHAs.
4. Fix Critical and Major findings through the owning agent. Log Minor findings unless they're quick to fix.
5. Commit per component: `<KEY> <summary> (FR-x, FR-y)`. Update PROGRESS.md with the SHAs.

## 9. When things go wrong

- **An agent fails:** re-brief it once, narrower. If it fails again, log it and move on to unblocked work.
- **Spec gap:** decide in the way that's most consistent with the spec, and log it under Decisions (`tech-writer` syncs the spec later). Ask the user only about irreversible, cross-team or scope-changing choices.
- **Environment blocker** (no Docker for Testcontainers, no registry or cluster access, a missing SDK): log it, keep going, and report it.
- **Context is getting long:** make sure PROGRESS.md is current and commit everything. Re-running this skill resumes the work.

## 10. Git, GitLab and cluster boundaries

- Work on feature branches only, and never commit to a default or protected branch.
- Push, create MRs (`glab mr create --draft`), or comment on Jira **only if the user asks**. Once asked, create one MR per touched component, link the MRs to each other and to the Jira key, and state the merge order (usually backend, then frontend, then GitOps).
- **Never** run `kubectl apply/edit/delete/scale`, `helm install/upgrade`, `argocd app sync`, or anything else that changes a cluster. Deployment happens when GitOps MRs are merged. Read-only cluster commands (`kubectl get`, `argocd app diff`) only if the user explicitly allows them.
- Never change production environment values unless the spec explicitly requires it. Even then, leave it for human MR approval and flag it in the report.

## 11. Finish

Stop when the `acceptance-auditor` shows every in-scope FR as Done, the gates pass in every touched component, and integration/E2E pass (or can't run, with the reason logged). Report:
- per component: the branch, commits, and what changed
- FRs done, and FRs deferred to other components or teams
- decisions and assumptions that extend the spec
- blockers, with the next step for each
- MR merge order and the deployment path (which ArgoCD apps pick it up, in which environments)
