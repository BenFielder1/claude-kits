# Progress — <KEY> <title>

Spec: `<path>/SPEC.md` (v<n>) · Started: <date> · Active components: <list> (requested scope: <all | "backend only" | FR-x–y>)

## Components
| Component | Path | Branch | Baseline gate | Base SHA | Latest SHA |
| --- | --- | --- | --- | --- | --- |
| orders-api | ../orders-api | feature/<KEY>-<slug> | ✓ / ✗ (pre-existing: …) | … | … |

## Requirements
| FR | Component | Phase | Status | Evidence |
| --- | --- | --- | --- | --- |
| FR-1 | orders-api | 1 | todo / in progress / done / deferred | test or file |

## Phases
- [ ] 0 Discover — CLAUDE.md present per component, baseline gates recorded
- [ ] 1 Contract & data
  - [ ] dotnet-engineer: contract stubs + OpenAPI regenerated
  - [ ] dotnet-data-engineer: entities + migration
  - [ ] gate · review · commit
- [ ] 2 Slice: <name> (FR-…)
  - [ ] dotnet-engineer ‖ react-engineer (mocked)
  - [ ] react-engineer: regenerate client, wire real API
  - [ ] gate · review · commit
- [ ] Integrate — qa-engineer
- [ ] Delivery — gitlab-ci-engineer → platform-engineer
- [ ] Final — acceptance-auditor → fixes → tech-writer

## Cross-component dependencies (outside active scope)
- <component> needs <change> for FR-n — owner/team — status

## Decisions
- <date> — <decision> — <reason> — spec synced: no

## Known issues / blockers
- <issue> — <tried> — <next step> — <owner>

## Environment
- .NET SDK: <ver> · Node: <ver> · Docker (Testcontainers): <yes/no> · glab auth: <yes/no> · cluster read access: <no/yes>
