---
name: github-actions-ci
description: Use for Git/GitHub workflow and CI in a single repo with backend/ (.NET) and frontend/ (Vite) and deploy/ (Kustomize), covering branches, commits, PRs, GitHub Actions jobs, contract-freshness checks, image build checks, manifest validation and Dependabot.
---

# Git, GitHub and GitHub Actions

**Default posture:** prepare and validate locally. Push, open PRs, or change repository settings and secrets **only when the user asks**.

## Git and PRs
- Branch names: `build/<name>` or `feat/<id>-<slug>`. Never commit directly to `main`.
- Commits: `<area>: <summary> (<IDs>)`, where the area is `backend`, `frontend`, `deploy`, `ci` or `docs`. One logical change per commit.
- PRs (only when asked): `gh pr create --draft --title "…" --body-file pr.md`. The body covers the summary, IDs covered, how it was tested (gate, local deploy, E2E), contract changes and the breaking-check result, and migrations.
- Add `.github/pull_request_template.md` with those sections.

## CI: `.github/workflows/ci.yml`
Use path filters, so a docs-only change doesn't run everything.

```yaml
name: CI
on:
  pull_request:
  push: { branches: [main] }
concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }

jobs:
  backend:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: backend } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { global-json-file: backend/global.json }
      - run: dotnet tool restore
      - run: dotnet restore
      - run: dotnet format --verify-no-changes --no-restore
      - run: dotnet build -c Release -warnaserror --no-restore
      - run: dotnet test -c Release --no-build --logger trx --results-directory TestResults
      - name: Contract is up to date
        run: git diff --exit-code openapi/api.json
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: backend-tests, path: backend/TestResults }

  frontend:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: frontend } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: frontend/.nvmrc, cache: npm, cache-dependency-path: frontend/package-lock.json }
      - run: npm ci
      - name: Client is up to date
        run: npm run gen:api && git diff --exit-code src/api/generated
      - run: npm run typecheck
      - run: npm run lint
      - run: npm test
      - run: npm run build

  deploy-manifests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Render and validate
        run: |
          kubectl kustomize deploy/k8s/overlays/local > rendered.yaml
          docker run --rm -v "$PWD":/w ghcr.io/yannh/kubeconform:latest -strict -summary -ignore-missing-schemas /w/rendered.yaml
      - name: Images build
        run: |
          docker build -f backend/Dockerfile backend
          docker build -f frontend/Dockerfile frontend
```
- Integration tests use Testcontainers, and Docker is available on `ubuntu-latest`.
- The `secrets.env` the overlay needs is git-ignored. For rendering in CI, the overlay must still build. Either copy `secrets.env.example` to `secrets.env` in a step first, or make the secret generator tolerate the example file. Pick one and document it.
- E2E in CI (optional, only if the spec asks): spin up a `kind` cluster, or run `docker compose` with the images, then Playwright. Don't add it by default.
- Before relying on the workflow, check the versions of the actions and the tool image tags against current docs, and pin them if the repo pins.

## Repository hygiene
- `.gitignore`: `bin/`, `obj/`, `node_modules/`, `dist/`, `TestResults/`, `playwright-report/`, `.env`, `deploy/k8s/overlays/local/secrets.env`, and `*.user`.
- `.github/dependabot.yml`: `nuget` (`/backend`), `npm` (`/frontend`), `github-actions` (`/`), and `docker` (`/backend`, `/frontend`). Weekly.
- Secret scan before commit: `git grep -nE "(Password=|pwd=|MSSQL_SA_PASSWORD=.+|BEGIN (RSA|OPENSSH) PRIVATE KEY)"` should only match `*.example` files.

## Validate locally
- Run each job's commands locally in the same order.
- Check the YAML syntax, with `actionlint` if available.
- Report what only a real GitHub run can confirm.

## Checklist
- [ ] CI mirrors the CLAUDE.md gate, plus contract and client freshness checks.
- [ ] Manifests render and validate, and images build.
- [ ] Dependabot and the PR template are present, with no secrets in the repo.
- [ ] Nothing was pushed, and no settings changed, unless the user asked.
