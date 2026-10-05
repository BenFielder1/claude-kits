# nextjs-supabase-kit

A reusable, spec-driven build kit for Claude Code on this stack: **Next.js (App Router) · TypeScript · Tailwind CSS · Next.js route handlers · Supabase · GitHub · Vercel**.

You write a `SPEC.md`. The orchestrator plans phases from it and delegates to specialised subagents. It gates every phase with checks and an independent review, then audits the result against every requirement.

## What's inside

| Type | Name | Purpose |
| --- | --- | --- |
| Skill | `orchestrate-build` | Plans phases from the spec, briefs agents, runs gates, commits, and resumes from `PROGRESS.md` |
| Skill | `supabase-migration` | Migrations, RLS, security definer RPCs, seed, types |
| Skill | `nextjs-api-route` | Route handler shape, Zod, error mapping, tests |
| Skill | `nextjs-ui` | Server/client split, Tailwind, forms, permissions, Realtime, accessibility |
| Skill | `domain-logic` | Pure, deterministic, test-first business rules |
| Skill | `github-vercel-delivery` | Branches/PRs, Actions CI, Vercel env, migration deploy |
| Agent | `project-scaffolder` | Phase 0 repo bootstrap and CLAUDE.md |
| Agent | `db-engineer` | Schema and permissions (serial) |
| Agent | `domain-logic-engineer` | `lib/<domain>` rules + tests |
| Agent | `api-engineer` | `app/api/**`, `lib/api/**` |
| Agent | `ui-engineer` | Pages and components |
| Agent | `test-engineer` | Playwright E2E, coverage gaps |
| Agent | `devops-engineer` | CI/CD and deploy readiness (doesn't deploy unless asked) |
| Agent | `code-reviewer` | Read-only phase review |
| Agent | `spec-auditor` | Read-only acceptance audit |
| Agent | `docs-writer` | README and SPEC sync |

Templates: `skills/orchestrate-build/templates/CLAUDE.template.md` and `PROGRESS.template.md`.

## Install

**As a plugin** (recommended; one copy, every project):
Put this folder in a Git repo or a local marketplace and install it with `/plugin`. The `.claude-plugin/plugin.json` manifest is included. Plugin skills and agents may appear with the `nextjs-supabase-kit:` prefix.

**For all your projects, without a plugin:**
```bash
cp -r agents/* ~/.claude/agents/
cp -r skills/* ~/.claude/skills/
```

**For one project:**
```bash
mkdir -p .claude && cp -r agents skills .claude/
```
Project-level copies override user-level ones with the same name, so you can customise per project.

## Use

1. Put `SPEC.md` at the project root. Give requirements IDs (FR-1…) if you can; otherwise the orchestrator assigns them.
2. In Claude Code: **"Use the orchestrate-build skill to build this app from SPEC.md."**
3. To resume later, send the same message. It picks up from `PROGRESS.md`.

It works on a `build/<name>` branch and commits at each phase gate. It won't push, open PRs, link Vercel or Supabase, or deploy unless you ask.

## Tailoring per project

- Project-specific rules (permissions, limits, formulas) belong in the project's `CLAUDE.md`, which the scaffolder writes from the spec. Keep the kit generic.
- For a project with a distinctive domain, add a project-level skill (e.g. `.claude/skills/pricing-rules/`), and name it in the orchestrator's briefs.
