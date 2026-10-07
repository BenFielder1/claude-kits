# Build progress

Spec: `SPEC.md` · Branch: `build/<name>` · Started: <date>

## Environment
- .NET SDK: <ver> · Node: <ver> · Docker: <ver / not available>
- Kubernetes: `docker-desktop` context <present + reachable | missing | unreachable>
- Local app URL: <http://localhost:8080> · Dev loop: API :5080, Vite :5173

## Requirements
| ID | Summary | Spec § | Area | Phase | Status | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| REQ-1 | … | … | backend / frontend / deploy / both | 1 | todo | — |

## Phases
- [ ] 0 Walking skeleton (base: <sha>)
  - [ ] repo-scaffolder
  - [ ] github-ci-engineer ‖ local-k8s-engineer (skeleton deployed + smoke ✓)
  - [ ] gate · review · commit
- [ ] 1 Foundations (base: <sha>) — IDs: …
  - [ ] ef-data-engineer ‖ aspnet-engineer (domain)
  - [ ] aspnet-engineer (auth, errors, contract)
  - [ ] vite-react-engineer (shell, auth, client)
  - [ ] gate · review · redeploy · commit
- [ ] N <feature> (base: <sha>) — IDs: …
  - [ ] contract stubs (+ migration)
  - [ ] aspnet-engineer ‖ vite-react-engineer (mocks)
  - [ ] vite-react-engineer (real API)
  - [ ] gate · review · redeploy · commit
- [ ] Integrate & deploy — e2e-qa-engineer → local-k8s-engineer → E2E on k8s
- [ ] Final — spec-verifier → fixes → github-ci-engineer → readme-writer

## Decisions
- <date> — <decision> — <reason> — SPEC updated: no

## Known issues
- <issue> — <tried> — <next step> — <owner>
