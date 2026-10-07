---
name: docker-desktop-k8s
description: Use when containerising the app or deploying it to the local Docker Desktop Kubernetes cluster, covering Dockerfiles (ASP.NET + EF migration bundle, Vite + nginx), Kustomize base and local overlay, database StatefulSet, migration Job, secrets from a git-ignored file, deploy/teardown/smoke scripts and troubleshooting.
---

# Deploying to Docker Desktop Kubernetes

The target is the **Docker Desktop** single-node cluster (kubectl context `docker-desktop`). Images are built locally and used without a registry. Everything is plain Kustomize (`kubectl apply -k`), so there's no extra tooling.

## Safety rules
- Every script starts by checking the context and refusing anything else:
  ```bash
  CTX=docker-desktop
  kubectl config get-contexts "$CTX" >/dev/null 2>&1 || { echo "No $CTX context. Enable Kubernetes in Docker Desktop."; exit 1; }
  K="kubectl --context $CTX"
  ```
  Use `$K` for every command, and **never** rely on the current context.
- Agents use the scripts, not ad-hoc `kubectl apply`. Changes to manifests happen in `deploy/k8s` and get applied through `deploy-local.sh`.
- **Never delete the DB PVC or namespace without asking.** `teardown-local.sh` keeps data unless it's passed `--wipe-data`.

## Images
Tag images `<app>-api:<git-short-sha>` and `<app>-web:<git-short-sha>` (append `-dirty` if the tree has changes), and set them in the overlay with `kustomize edit set image`, or a generated `images:` block written by the script. Never use `:latest`. Set `imagePullPolicy: IfNotPresent` (or `Never`) so Kubernetes uses the local image.

**Image visibility:** with Docker Desktop's default Kubernetes setup, the cluster shares Docker's image store, so `docker build` is enough. If the cluster was created with the **kind** provisioner option in newer Docker Desktop versions, check that locally built images are visible. Pods stuck in `ErrImageNeverPull` or `ImagePullBackOff` mean they aren't. In that case, the script must load the images into the cluster's nodes or use a local registry. Detect which case applies, and document it in the README.

### `backend/Dockerfile`
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:<ver> AS build
WORKDIR /src
COPY global.json Directory.*.props *.sln ./
COPY .config/ .config/
# Copy each src/<Project>/<Project>.csproj into its folder here, then `dotnet restore`, for layer caching
RUN dotnet tool restore
COPY . .
RUN dotnet publish src/<App>.Api -c Release -o /app/publish
RUN dotnet ef migrations bundle --self-contained -r linux-$(uname -m | sed 's/x86_64/x64/;s/aarch64/arm64/') \
    --project src/<App>.Infrastructure --startup-project src/<App>.Api -o /app/efbundle

FROM mcr.microsoft.com/dotnet/aspnet:<ver> AS runtime
WORKDIR /app
COPY --from=build /app/publish .
COPY --from=build /app/efbundle /app/efbundle
USER $APP_UID
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "<App>.Api.dll"]
```
Adjust the restore-caching step to the real project layout. `uname -m` in the build stage matches the node's architecture when building locally, so the bundle runs on the Docker Desktop node.

### `frontend/Dockerfile` (+ `frontend/nginx.conf`)
```dockerfile
FROM node:<lts> AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginxinc/nginx-unprivileged:<stable> AS runtime
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
```
```nginx
server {
  listen 8080;
  root /usr/share/nginx/html;
  location /api/ { proxy_pass http://api:8080; proxy_set_header Host $host; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header X-Forwarded-Proto $scheme; }
  location /assets/ { add_header Cache-Control "public, max-age=31536000, immutable"; try_files $uri =404; }
  location / { add_header Cache-Control "no-cache"; try_files $uri /index.html; }
}
```
The web container is the **single entry point**: it serves the SPA and proxies `/api` to the API Service, so the app is same-origin with no CORS. If the API sits behind this proxy, enable ForwardedHeaders in ASP.NET.

## Manifests (`deploy/k8s`)
```
base/
  kustomization.yaml     namespace: <app>; resources below; labels app.kubernetes.io/part-of: <app>
  namespace.yaml
  db.yaml                StatefulSet (1 replica) + headless Service "db" + volumeClaimTemplate (default storage class)
  api.yaml               Deployment "api" + ClusterIP Service "api" :8080
  web.yaml               Deployment "web" + Service "web" type LoadBalancer port <8080> → 8080
  migrate-job.yaml       Job "db-migrate" running /app/efbundle (same image as api)
  configmap.yaml         non-secret settings (ASPNETCORE_ENVIRONMENT, Seed__Enabled, …)
overlays/local/
  kustomization.yaml     resources: ../../base; images: …; secretGenerator from secrets.env; patches
  secrets.env.example    keys only, with placeholder values (committed)
  secrets.env            real local values (git-ignored)
```
- **Exposure:** Docker Desktop publishes `LoadBalancer` Services on `localhost`, so the app is at `http://localhost:<port>`. Pick a port that doesn't clash with the dev loop (5080, 5173); 8080 is the default. No ingress controller is needed. Only use Ingress if the spec requires host names, and then install the controller as a documented manual step.
- **Secrets:** `secretGenerator` (`envs: [secrets.env]`) creates `app-secrets`, holding the DB password and `ConnectionStrings__Default` (pointing at `db`). The API and Job read them with `envFrom: secretRef`. Never commit real values.
- **Database:** SQL Server (`mcr.microsoft.com/mssql/server:2022-latest`, `ACCEPT_EULA=Y`, `MSSQL_SA_PASSWORD` from the secret) or PostgreSQL, per the spec. Give it a readiness probe (a `sqlcmd` query / `pg_isready`), memory requests suited to a laptop, and a PVC so data survives redeploys.
- **API Deployment:**
  - `replicas: 1` locally (2 if the spec wants rollout testing).
  - Readiness probe `/api/health/ready`, liveness `/api/health/live`, and a startup probe.
  - Small resource requests.
  - `securityContext`: `runAsNonRoot`, `allowPrivilegeEscalation: false`, drop ALL capabilities, `readOnlyRootFilesystem: true` with an `emptyDir` at `/tmp`.
- **Web Deployment:** readiness on `/`, and the same `securityContext` (nginx-unprivileged works as non-root; give it writable `emptyDir`s for its temp paths if the filesystem is read-only).
- **Migration Job:** `backoffLimit: 2`, `ttlSecondsAfterFinished: 600`, and the same image tag as the API. Because Jobs are immutable, the script deletes the previous `db-migrate` Job before applying.

## Scripts (`deploy/scripts`)

**`deploy-local.sh`**:
1. Run the context check.
2. Make sure `secrets.env` exists. If it doesn't, copy the example, generate a strong DB password, and tell the user.
3. Run `docker build` for the API and web images with the SHA tag.
4. Write the image tags into the overlay (`kustomize edit set image`, or a generated patch).
5. Run `$K apply -k deploy/k8s/overlays/local`. Don't use `--prune`, which can delete resources it shouldn't.
6. Wait for the DB: `$K -n <app> rollout status statefulset/db --timeout=180s`.
7. Delete the old Job, re-apply it, then `$K -n <app> wait --for=condition=complete job/db-migrate --timeout=180s`. On failure, print the Job's logs and exit non-zero.
8. Restart the API and web if the tags didn't change, then run `rollout status` for both.
9. Run `smoke.sh`.

**`smoke.sh`**: `curl -fsS http://localhost:<port>/` (the HTML contains the app root) and `curl -fsS http://localhost:<port>/api/health/ready`, with retries for up to about 60 seconds. Exit non-zero with useful output on failure.

**`teardown-local.sh`**: deletes the app resources and keeps the PVC and namespace. Only `--wipe-data` deletes the namespace, and only after a confirmation prompt.

All scripts use `set -euo pipefail`, print each step, and are idempotent.

## Troubleshooting (put these in your report when they happen)
| Symptom | Check |
| --- | --- |
| `ErrImageNeverPull` / `ImagePullBackOff` | Image visibility (kind provisioner?), the tag in the overlay, `imagePullPolicy` |
| API `CrashLoopBackOff` | `$K logs deploy/api`: connection string, missing secret key, port binding |
| Migration Job fails | `$K logs job/db-migrate`: DB not ready, wrong bundle architecture, SQL error |
| DB pod pending | PVC and storage class (`$K get pvc,sc`), the memory Docker Desktop has |
| Nothing on localhost:<port> | `$K get svc web` (does it show `localhost`?), a port clash with another process |

## Offline validation (when there's no cluster)
`kubectl kustomize deploy/k8s/overlays/local | kubeconform -strict -summary -ignore-missing-schemas` and `docker build` both images. Report that the live deploy wasn't verified.

## Checklist
- [ ] Scripts enforce the `docker-desktop` context, are idempotent, and never wipe data without `--wipe-data` and a confirmation.
- [ ] Images are tagged by SHA, non-root, and built multi-stage, with the EF bundle in the API image.
- [ ] Migrations run as a Job before the API rollout.
- [ ] Secrets come from a git-ignored env file, and the example is committed.
- [ ] Probes, resources and securityContext are set, and the web is the single entry point proxying `/api`.
- [ ] `deploy-local.sh` succeeds, and the smoke test passes (or offline validation passes, with the limitation reported).
