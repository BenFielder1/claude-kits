# Build progress

Spec: `SPEC.md` · Branch: `build/<name>` · Started: <date>

## Requirements
<!-- Use the spec's own IDs if it has them; otherwise assign REQ-n here. -->
| ID | Summary | Spec section | Phase | Status |
| --- | --- | --- | --- | --- |
| REQ-1 | … | § … | 1 | todo |

## Phases
- [ ] Phase 0 – Scaffold (base: <sha>)
  - [ ] project-scaffolder: app, tooling, Supabase init, CLAUDE.md
  - [ ] devops-engineer: CI skeleton
  - [ ] gate · review · commit
- [ ] Phase 1 – Foundations (base: <sha>) — IDs: …
  - [ ] 1a db-engineer: …
  - [ ] 1b domain-logic-engineer: …
  - [ ] 1c api-engineer / ui-engineer: auth …
  - [ ] gate · review · commit
- [ ] Phase N – <feature> (base: <sha>) — IDs: …
- [ ] Final – Harden & accept
  - [ ] ui polish ‖ test-engineer E2E
  - [ ] spec-auditor → fixes → re-audit
  - [ ] devops-engineer: CI/deploy readiness
  - [ ] docs-writer: README + SPEC sync

## Decisions
- <date> — <decision> — <reason> — SPEC updated: no

## Known issues
- <issue> — <what was tried> — <next step> — <owner>

## Environment
- Docker / local Supabase: <available | not available>
- GitHub remote: <url | none> · gh auth: <yes | no>
- Vercel project linked: <yes | no>
