---
name: k8s-argocd-gitops
description: Use when changing how a service is containerised or deployed, covering Dockerfiles, Helm charts or Kustomize overlays, ArgoCD Applications/ApplicationSets, per-environment config and secrets references, probes, resources, migration jobs and offline manifest validation. Changes go through Git only.
---

# Kubernetes + ArgoCD via GitOps

**Deployment happens when GitOps MRs are merged, not when agents run commands.** Agents never change a cluster.

**Never:** `kubectl apply/create/edit/patch/delete/scale/rollout`, `helm install/upgrade/uninstall`, `argocd app sync/rollback/set`, or editing live resources.
**Read-only commands** (`kubectl get/describe`, `argocd app get/diff`) are allowed only if the user explicitly allows them for this session.

## 0. Detect the GitOps layout
- **Helm**: `charts/<app>/` plus per-environment `values-<env>.yaml`, or a shared library chart.
- **Kustomize**: `apps/<app>/base` plus `overlays/<env>`.
- **ArgoCD**: `Application` per app/env, or an `ApplicationSet` (with git, list or matrix generators) creating them. Check `syncPolicy` (automated or manual), `syncOptions`, sync waves and hooks.
- **Secrets**: External Secrets Operator (`ExternalSecret` → Vault/cloud secret manager), Sealed Secrets, or SOPS. **Use only the existing mechanism.**
- **Ingress**: Ingress (and which controller), or Gateway API `HTTPRoute`.
Mirror the most similar existing app in the repo.

## 1. Dockerfiles (in the app repos)
- **.NET**: multi-stage. `mcr.microsoft.com/dotnet/sdk:<ver>` for restore, publish (`-c Release`) and the runtime `mcr.microsoft.com/dotnet/aspnet:<ver>` image (or the chiseled/distroless variant if the organisation uses it). Run as non-root (`USER app` or `$APP_UID` on recent images). Recent images listen on **8080** by default. Restore with only the project files copied first, for layer caching.
- **React/Vite**: multi-stage. `node:<lts>` runs `npm ci && npm run build`, then an unprivileged nginx image (e.g. `nginxinc/nginx-unprivileged`) serves `dist/`, with an SPA fallback (`try_files $uri /index.html`), long cache headers for hashed assets, `no-cache` for `index.html` and the runtime config, and the runtime-config mechanism from the `react-vite-ui` skill.
- Pin base image versions, include a `.dockerignore`, and add no secrets and no `latest` tags.

## 2. Workload checklist (per service)
- [ ] Image is referenced by an immutable tag (SHA or release), set per environment.
- [ ] `resources.requests` set. `limits` follow the organisation's convention (often a memory limit, and CPU limits sometimes omitted).
- [ ] `readinessProbe` on `/health/ready`, `livenessProbe` on `/health/live` (never a dependency check), and a `startupProbe` if start-up is slow.
- [ ] `securityContext`: `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true` (mount `emptyDir` for `/tmp` if needed), `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault`.
- [ ] `replicas` ≥ 2 for anything user-facing, plus a `PodDisruptionBudget`, and an HPA if the spec has load requirements.
- [ ] Config comes from a ConfigMap (env vars or a mounted file). Secrets come from the existing secret mechanism, referenced by name only.
- [ ] Standard labels (`app.kubernetes.io/name`, `instance`, `version`, `part-of`, `managed-by`).
- [ ] Graceful shutdown: `terminationGracePeriodSeconds` covers in-flight requests (and a `preStop` sleep if the ingress needs it).
- [ ] NetworkPolicy follows the organisation's convention.

## 3. Database migrations on deploy
Follow the existing pattern. If there isn't one, the recommended approach is a `Job` running the EF Core migration bundle as an **ArgoCD PreSync hook** (`argocd.argoproj.io/hook: PreSync`, with `hook-delete-policy: BeforeHookCreation`). It uses the same image tag as the app and its own DB credentials. Migrations must be expand/contract safe (see `dotnet-data`), because rollbacks don't undo the schema.

## 4. Environments and promotion
- **Change dev first.** Promote to staging and prod in separate MRs or commits, following the WORKSPACE.md rules about who may change what.
- **Never change prod values unless the spec explicitly requires it.** Flag any prod change for human approval.
- New config keys and secret references must exist in **every** environment the app deploys to, or the rollout fails. List any secret values a human must create.
- Use sync waves when ordering matters (for example ConfigMap/ExternalSecret first, then the migration Job, then the Deployment).

## 5. Validate offline
```bash
helm dependency build charts/<app>            # if it has dependencies
helm lint charts/<app> -f charts/<app>/values-<env>.yaml
helm template <app> charts/<app> -f charts/<app>/values-<env>.yaml | kubeconform -strict -summary -ignore-missing-schemas
kustomize build apps/<app>/overlays/<env> | kubeconform -strict -summary -ignore-missing-schemas
```
- Use the repo's own validation script if there is one.
- For CRDs (ExternalSecret, Application, HTTPRoute), add their schema location for kubeconform if the repo provides one. Otherwise, `-ignore-missing-schemas` is accepted, and you report which kinds weren't schema-checked.
- Render before and after, and diff the output (`diff <(git show HEAD:… | …) <(…)`, or render both into files). Include a short summary of the rendered diff in your report.

## Checklist
- [ ] Mirrors the most similar existing app, and uses the existing secret and ingress mechanisms.
- [ ] Workload checklist items are satisfied.
- [ ] Dev changed. Other environments changed only as allowed, and prod is flagged.
- [ ] Offline validation passes, and the rendered diff is summarised.
- [ ] Secret values humans must create are listed (names and paths only, never values).
- [ ] No cluster-changing commands were run.
