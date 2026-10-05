---
name: github-vercel-delivery
description: Use when setting up or changing GitHub workflow (branches, PRs, Actions CI), Vercel deployment config and env vars, or Supabase migration deployment for a Next.js + Supabase project.
---

# GitHub + Vercel + Supabase delivery

**Default posture:** make everything ready and documented, but don't push, link, deploy or touch remote databases unless the user explicitly asks.

## Git and GitHub
- Work on a branch (`build/<name>` or `feat/<name>`), never directly on `main`.
- Write commits as `<scope>: <summary> (<requirement IDs>)`, one logical change each.
- PRs (only when asked): `gh pr create --draft --title "<summary>" --body-file <file>`. The body lists the change, the requirement IDs covered, how it was tested, and known issues.
- Add `.github/pull_request_template.md` with sections for Summary, Requirements, Testing and Screenshots.

## CI: `.github/workflows/ci.yml`

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }
jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc, cache: npm }
      - run: npm ci
      - run: npm run typecheck
      - run: npm run lint
      - run: npm test
      - run: npm run build
        env:
          NEXT_PUBLIC_SUPABASE_URL: http://127.0.0.1:54321
          NEXT_PUBLIC_SUPABASE_ANON_KEY: ci-placeholder
  e2e:
    runs-on: ubuntu-latest
    needs: checks
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc, cache: npm }
      - uses: supabase/setup-cli@v1
        with: { version: latest }
      - run: npm ci
      - run: supabase start
      - name: Export local Supabase env
        run: |
          supabase status -o env | sed -n 's/^API_URL=/NEXT_PUBLIC_SUPABASE_URL=/p; s/^ANON_KEY=/NEXT_PUBLIC_SUPABASE_ANON_KEY=/p' | tr -d '"' >> "$GITHUB_ENV"
      - run: npx playwright install --with-deps chromium
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: playwright-report, path: playwright-report }
```
Also add a `db-lint` step if useful (`supabase db lint`). Keep the checks job under about 5 minutes.

Before relying on the CI file, check the versions of the actions and the `supabase status` output format against current docs. If you can't verify them, say so in your summary.

## Supabase migrations to remote (only when asked)
- **Option A** (simplest): the Supabase GitHub integration, where migrations in `supabase/migrations` apply on merge to the production branch, with branching for PR previews.
- **Option B**: an Actions job on push to `main` running `supabase link --project-ref $SUPABASE_PROJECT_REF` and then `supabase db push`, with secrets `SUPABASE_ACCESS_TOKEN` and `SUPABASE_DB_PASSWORD`.
- Always review the migration diff before pushing, and never push `seed.sql` to production.

## Vercel
- Connect through Vercel's GitHub integration: previews on PRs, production on `main`. Only add `vercel.json` if you need config (regions, crons, headers).
- Env vars: set them in the Vercel project (Production, Preview and Development) or with `vercel env add`. List every variable in `.env.example` with a comment giving its source and scope.
  - `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` are public.
  - Server-only secrets (only if the spec needs them, e.g. webhook secrets) never get the `NEXT_PUBLIC_` prefix.
- Supabase Auth: add the Vercel production URL and the preview URL pattern (`https://*-<team>.vercel.app/**`) to Auth → URL Configuration → Redirect URLs.
- Cron (if the spec needs it): `vercel.json` `crons`, hitting a route protected by `CRON_SECRET`.

## Security hygiene
- `.gitignore` covers `.env*.local`, `.vercel`, `supabase/.temp` and test reports.
- Run `git grep -nE "(service_role|sk_live|SUPABASE_SERVICE)"`. It should return nothing in app code.
- Enable Dependabot (`.github/dependabot.yml`) for npm and GitHub Actions, weekly.

## Deploy-readiness checklist
- [ ] CI is green on the branch (or would be; if not run, say why).
- [ ] `.env.example` is complete, with sources and scopes.
- [ ] The README has deploy steps: Supabase project, migrations, auth URLs, Vercel env, and the first deploy.
- [ ] No secrets are in the repo, and no service-role key is in app code.
- [ ] Production auth settings are listed (email confirmation, anonymous sign-in and CAPTCHA if used, rate limits).
