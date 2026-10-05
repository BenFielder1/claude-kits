# Workspace: <product / team name>

<One line: what this workspace delivers.>

- **Layout:** <multi-repo with all repos under one parent folder | monorepo | mixed>
- **Specs:** `specs/<JIRA-KEY>/SPEC.md` (+ `PROGRESS.md` alongside)
- **Jira project(s):** <KEY> · **GitLab group:** <https://gitlab.example.com/group>
- **Container registry:** <registry.gitlab.example.com/group>
- **ArgoCD:** <https://argocd.example.com> (read-only access for agents: yes / no)

## Components

| Name | Type | Path (relative to this file) | GitLab project | Default branch | Gate |
| --- | --- | --- | --- | --- | --- |
| web | frontend | ../web-app | group/web-app | main | `npm ci && npm run typecheck && npm run lint && npm test && npm run build` |
| orders-api | backend | ../orders-api | group/orders-api | main | `dotnet build -warnaserror && dotnet test && dotnet format --verify-no-changes` |
| gitops | gitops | ../platform-gitops | group/platform-gitops | main | `./scripts/validate.sh` (helm lint + kubeconform) |

Each component has its own `CLAUDE.md` with conventions and commands.

## Contracts

| Provider | Contract file (in provider repo) | Consumers | Client generation |
| --- | --- | --- | --- |
| orders-api | `openapi/orders-api.json` (regenerated on build) | web | in web: `npm run gen:api` (reads `../orders-api/openapi/orders-api.json`) |

## Environments

| Env | ArgoCD application(s) | Namespace | Values / overlay path (in gitops) | How images update | Who may change |
| --- | --- | --- | --- | --- | --- |
| dev | orders-api-dev, web-dev | team-dev | `apps/orders-api/values-dev.yaml` | CI bumps image tag on merge to main | agents via MR |
| staging | … | team-staging | … | promotion MR | agents via MR, human approval |
| prod | … | team-prod | … | promotion MR | humans only |

## Local stack (for integration and E2E)

```bash
docker compose -f ../orders-api/docker-compose.yml up -d   # db + dependencies
dotnet run --project ../orders-api/src/Orders.Api           # http://localhost:5080
npm --prefix ../web-app run dev                             # http://localhost:5173 (proxy /api → 5080)
```

## Access notes

- To work across repos in one session, start Claude Code in this folder, or add each component with `--add-dir` or `/add-dir`.
- Secrets live in <Vault / External Secrets / Sealed Secrets / GitLab CI variables>. They're never committed.
