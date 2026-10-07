# <Project name>

<2–3 sentences from the spec: what the app does and for whom.>

**`SPEC.md` is the source of truth.** Read the relevant section before building a feature. If code and spec disagree, flag it rather than silently picking one. Requirement IDs are referenced in commits and PRs.

## Stack

- **Frontend** (`frontend/`): TypeScript (strict) + React + Vite, <router>, <server-state lib>, <styling>, Vitest + Testing Library + MSW, Playwright
- **Backend** (`backend/`): C# / ASP.NET Core (.NET <LTS version>), minimal APIs, EF Core + <SQL Server | PostgreSQL>, xUnit, Testcontainers
- **Contract**: `backend/openapi/api.json` (generated, committed) → `frontend/src/api/generated/` (generated)
- **Deploy**: Docker images and Kustomize manifests (`deploy/k8s`) on the Docker Desktop Kubernetes cluster (context `docker-desktop`)
- **VCS/CI**: Git, GitHub, GitHub Actions

## Layout

```
backend/
  <App>.sln
  src/<App>.Api/            endpoints, DI, auth, ProblemDetails, OpenAPI
  src/<App>.Domain/         pure business rules (no EF, no ASP.NET)
  src/<App>.Infrastructure/ DbContext, entity configs, Migrations/, external clients
  tests/<App>.UnitTests/    domain + application tests
  tests/<App>.IntegrationTests/ WebApplicationFactory + Testcontainers
  openapi/api.json          GENERATED contract — commit, never hand-edit
  Directory.Build.props · Directory.Packages.props · global.json
frontend/
  src/api/generated/        GENERATED client — never hand-edit
  src/features/<feature>/   pages, components, hooks, tests
  src/lib/                  errors, permissions, config
  e2e/                      Playwright
deploy/
  docker/                   (or Dockerfiles in backend/ and frontend/)
  k8s/base/                 namespace, db, api, web, migration job
  k8s/overlays/local/       docker-desktop overlay, secrets.env (git-ignored), secrets.env.example
  scripts/                  deploy-local.sh, teardown-local.sh, smoke.sh
docker-compose.yml          dev-loop database only
.github/workflows/ci.yml
SPEC.md · PROGRESS.md · CLAUDE.md
```

## Commands

```bash
# Dev loop
docker compose up -d db                                   # dev database
dotnet run --project backend/src/<App>.Api                # http://localhost:5080
npm --prefix frontend run dev                             # http://localhost:5173 (proxies /api → 5080)

# Backend
cd backend && dotnet build -warnaserror && dotnet test && dotnet format --verify-no-changes
dotnet ef migrations add <Name> --project src/<App>.Infrastructure --startup-project src/<App>.Api
dotnet build src/<App>.Api                                # regenerates openapi/api.json

# Frontend
cd frontend && npm run typecheck && npm run lint && npm test && npm run build
npm run gen:api                                           # regenerate client from ../backend/openapi/api.json
npm run test:e2e                                          # Playwright (BASE_URL=http://localhost:5173 or the k8s URL)

# Local Kubernetes (Docker Desktop)
./deploy/scripts/deploy-local.sh                          # build images, apply, migrate, roll out, smoke test
./deploy/scripts/teardown-local.sh                        # remove app (keeps DB data unless --wipe-data)
```

**Gate:**
```bash
(cd backend && dotnet build -warnaserror && dotnet test && dotnet format --verify-no-changes) && (cd frontend && npm run typecheck && npm run lint && npm test && npm run build) && kubectl kustomize deploy/k8s/overlays/local > /dev/null
```

## Rules that must not be broken

1. `openapi/api.json` and `frontend/src/api/generated/` are generated. Regenerate them, and never hand-edit them. API changes are additive unless SPEC.md records a breaking-change decision.
2. The frontend calls the backend only through the generated client, at same-origin `/api`. No hard-coded hosts.
3. Business rules live in `<App>.Domain`, pure and unit-tested. Endpoints stay thin.
4. Errors are ProblemDetails with an `errorCode` from the spec's error table. The frontend maps those codes to copy in one place.
5. Every schema change is an EF Core migration. Never edit a committed migration, and never use `EnsureCreated` outside tests.
6. Secrets are never committed: user-secrets for the dev loop, `deploy/k8s/overlays/local/secrets.env` (git-ignored) for Kubernetes.
7. Deploy scripts target only the `docker-desktop` Kubernetes context.
<!-- Add project-specific invariants from the spec: permissions, limits, formulas, derived vs stored data. -->

## Conventions

- C#: nullable on, warnings as errors, async with a `CancellationToken`, `TypedResults`, `ILogger` message templates, `TimeProvider` injected.
- TS: strict, no `any`. Server state lives in the query library, and environment values come from same-origin config, not `VITE_*`, unless they're build-time constants.
- Names: C# PascalCase, TS files kebab-case, components PascalCase, DB tables and columns per the EF conventions configured.

## Workflow

Use the `build-from-spec` skill for multi-phase work. Stack skills: `aspnet-endpoints`, `efcore-data`, `vite-react-ui`, `openapi-contract`, `docker-desktop-k8s`, `github-actions-ci`.

## Definition of done

- The gate passes, and new behaviour has tests named after the requirement IDs.
- Contract and client are regenerated (if the API changed).
- Migration added and reviewed (if the schema changed).
- Redeployed to Docker Desktop, and the smoke test passes (if Kubernetes is available).
- SPEC.md is updated if behaviour differs from it.
