---
name: api-contract
description: Use whenever an HTTP API between a .NET service and a React frontend changes. Covers generating and committing the OpenAPI document, checking for breaking changes, regenerating the typed TypeScript client, and contract-first work across folders/repos.
---

# API contract (OpenAPI) between repos

The committed OpenAPI file is the **handshake** between backend and frontend, even when they live in different repos. Frontend and backend agents coordinate through this file, not through each other.

## Provider side (.NET)
- Generate the document from code, using the repo's mechanism: `Microsoft.AspNetCore.OpenApi` (built into recent .NET), or Swashbuckle/NSwag in older services.
- **Commit the generated file** (e.g. `openapi/<service>.json`) so contract changes appear in MR diffs. Prefer build-time generation if it's set up (e.g. `Microsoft.Extensions.ApiDescription.Server` with `OpenApiDocumentsDirectory`). Otherwise use a documented script that runs the app's document endpoint.
- Make the document precise: typed results for every status, `required` and nullability matching the C# types, enums as strings if the repo does that, ProblemDetails for errors, operation IDs that are stable and readable (`GetOrder`, `LockOrder`), and tags per feature.
- **Contract-first:** commit DTOs and endpoint signatures with `501` stubs and regenerate, before implementing. Consumers can start straight away.

## Breaking-change rules
Without a recorded spec decision, these are **not allowed**:
- removing or renaming an endpoint, parameter, or response field
- making an optional request field required, or adding a new required request field
- changing a field's type or format, or narrowing an enum the client sends
- changing the status codes or error codes clients rely on

These are **allowed**: new endpoints, new optional request fields, new response fields, new enum values in responses (if clients tolerate unknown values), and deprecations (`deprecated: true` plus a note).

**Check:** compare the base branch's contract file with the new one. Use `oasdiff breaking <base.json> <new.json>` if available (installable as a Go binary or Docker image). Otherwise, do a careful manual diff of paths, required fields and schemas. Report the result in your summary. If a breaking change is truly needed, follow the spec's decision: usually a new versioned route (`/v2/...`), or a parallel field with a deprecation window.

## Consumer side (React)
- Generate the typed client with the repo's tool (openapi-typescript + openapi-fetch, Orval, or NSwag TS), using a script such as `npm run gen:api` that reads the provider's contract. Take the path or URL from WORKSPACE.md; it's often a sibling repo path or a published CI artifact.
- **Generated files are committed or generated in CI**, following repo convention, and are **never hand-edited**.
- After regenerating, the typecheck tells you every call site affected. Fix them all.
- **Mocks:** MSW handlers for new or changed endpoints return data that matches the generated types, so TypeScript catches any drift in the mocks.

## Errors contract
- Errors are ProblemDetails, carrying `errorCode` (a stable, UPPER_SNAKE value), `title`, `status`, and `errors` for validation.
- The spec's error table is the source of truth. The backend emits exactly those codes, and the frontend maps exactly those codes to copy.

## Checklist
- [ ] The contract file was regenerated and committed in the provider.
- [ ] The breaking-change check was run, with the result recorded.
- [ ] The consumer client was regenerated, and the typecheck is clean.
- [ ] Error codes match the spec on both sides.
- [ ] Every consumer outside the active scope is listed as a cross-component follow-up.
