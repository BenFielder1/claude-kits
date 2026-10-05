---
jira: <KEY-123>
title: <short title>
issue_type: <Story | Bug | Task | Epic>
jira_status_at_capture: <status>
captured: <YYYY-MM-DD>
spec_version: 1
components: [<component names from WORKSPACE.md>]
---

# <KEY-123> — <Title>

Jira: <link> · Parent: <EPIC-KEY — title | none>

## 1. Summary
<2–3 sentences: the problem, who has it, and the outcome when this is done.>

## 2. Scope
**In scope**
- …

**Out of scope**
- …

## 3. Users and permissions
| Role | Can | Cannot / behaviour when unauthorised |
| --- | --- | --- |
| … | … | … |

## 4. Functional requirements

| ID | Requirement | Component | Source | Priority |
| --- | --- | --- | --- | --- |
| FR-1 | … | <component> | AC-1 | Must |

**Jira AC → FR mapping:** AC-1 → FR-1, FR-2 · AC-2 → FR-3 · …

### FR-1 — <name>
**Component:** <component> · **Source:** <AC-1 / comment by X on date / user answer Q2>

<Behaviour in 1–4 sentences. Current behaviour, if this changes it: …>

**Acceptance**
- Given <state>, when <action>, then <outcome>.
- Given <edge case>, when <action>, then <outcome>.

### FR-2 — …

## 5. API contract changes
| Change | Method | Path | Request | Response | Errors (code → status) | Auth | Compatible? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| new | POST | /api/v1/… | `{ … }` | `201 { … }` | `ORDER_LOCKED → 409` | policy `…` | yes |

Notes: <versioning, deprecations, consumers affected>. None if no API change.

## 6. Data changes
| Entity / table | Change | Migration | Backfill | Old-and-new-version safe? |
| --- | --- | --- | --- | --- |
| … | add column `x` (nullable) | yes | yes, from … | yes (expand/contract) |

Retention, privacy and audit notes: …

## 7. UI changes
| Screen / route | Change | States | Notes |
| --- | --- | --- | --- |
| … | … | loading · empty · error · forbidden | validation, copy, a11y |

## 8. Non-functional requirements
- Performance: …
- Security: …
- Observability (logs, metrics, traces, alerts): …
- Audit: …
- Accessibility / localisation: …

## 9. Configuration and deployment
| Item | Type (config / secret / k8s resource / feature flag) | dev | test/staging | prod |
| --- | --- | --- | --- | --- |
| … | … | … | … | … |

Rollout: <order of environments, flag strategy, migration timing>.

## 10. Testing approach
- Unit: …
- Integration (API + DB): …
- E2E: <journeys>
- Test data: …

## 11. Delivery plan hints
- Order: <e.g. contract → backend (data + API) → frontend → gitops>
- Can run in parallel: <e.g. frontend with mocked contract once the contract is agreed>
- Components that can be skipped or are unaffected: …
- Dependencies on other tickets or teams: …

## 12. Decisions and assumptions
| # | Decision / assumption | Source | Status |
| --- | --- | --- | --- |
| D1 | … | user answer / ticket / code (`path`) | agreed / assumed |

## 13. Open questions
| # | Question | Who can answer | Blocks |
| --- | --- | --- | --- |
| Q1 | … | <role/person> | FR-n / none |

## 14. Sources
- Jira: <KEY> (+ linked issues consulted)
- Confluence: <pages>
- Code consulted: `<component>/<path>` — <what was learned>
