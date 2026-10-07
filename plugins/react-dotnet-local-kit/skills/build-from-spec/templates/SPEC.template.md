# <App name> — Specification

<as-of date> · <author>

## 1. Overview
<What the app does, for whom, and what success looks like. 3–5 sentences.>

**In scope (v1):** …
**Out of scope:** …

## 2. Users and permissions
| Role | Can | Cannot |
| --- | --- | --- |
| … | … | … |

Auth: <none | ASP.NET Core Identity (email + password, cookie) | external OIDC: …>

## 3. Functional requirements
<Group by feature area. One ID per testable requirement.>

### <Area>
1. **FR-1** <requirement>
   - Given … when … then …
2. **FR-2** …

## 4. Domain rules
<Formulas, limits, state machines, tie-breaks, and worked examples. These become unit tests.>

## 5. Data model
| Entity | Key fields | Constraints / indexes | Notes |
| --- | --- | --- | --- |
| … | … | … | … |

Database: <SQL Server | PostgreSQL>

## 6. API
| Method | Route | Body / query | Response | Errors (code → status) | Auth |
| --- | --- | --- | --- | --- | --- |
| … | /api/… | … | … | `NAME_TAKEN → 409` | … |

## 7. Pages and UI
| Route | Page | Key content / states |
| --- | --- | --- |
| … | … | loading · empty · error · forbidden |

## 8. Non-functional requirements
Performance, accessibility, browsers and devices, logging.

## 9. Local deployment
URL/port on Docker Desktop, seed data, anything environment-specific.

## 10. Decisions
- …

## 11. Open questions
- [ ] …

## 12. Milestones
1. **M1** …
