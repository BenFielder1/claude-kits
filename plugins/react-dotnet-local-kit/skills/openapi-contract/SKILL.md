---
name: openapi-contract
description: Use whenever the HTTP API between backend/ and frontend/ changes. Covers generating and committing backend/openapi/api.json from ASP.NET Core, contract-first stubs, breaking-change checks, and regenerating the typed client in frontend/src/api/generated.
---

# The OpenAPI contract (single repo)

`backend/openapi/api.json` is the handshake between the two halves of the repo. The backend generates it, and the frontend generates its client from it. Both generated outputs are committed, so every API change shows up in the PR diff.

## Backend: generate the document
- Use `Microsoft.AspNetCore.OpenApi` (built into recent .NET): `builder.Services.AddOpenApi()` and `app.MapOpenApi()` (in Development only, if the spec prefers).
- **Generate at build** into the repo with `Microsoft.Extensions.ApiDescription.Server`. In `<App>.Api.csproj`:
  ```xml
  <PropertyGroup>
    <OpenApiDocumentsDirectory>$(MSBuildProjectDirectory)/../../openapi</OpenApiDocumentsDirectory>
    <OpenApiGenerateDocumentsOptions>--file-name api</OpenApiGenerateDocumentsOptions>
  </PropertyGroup>
  ```
  Check these property names against the docs for the installed .NET version. If build-time generation isn't workable, add a script that runs the app and saves `/openapi/v1.json` to `backend/openapi/api.json`, and document it in CLAUDE.md.
- Make the document precise:
  - `TypedResults` for every status, and stable `.WithName()` operation IDs.
  - Nullability and `required` that match the C# types.
  - Enums as strings, if configured.
  - ProblemDetails schemas for error responses.

## Contract-first stubs
To unblock the frontend, add the DTO records and endpoint signatures with handlers that return `TypedResults.StatusCode(StatusCodes.Status501NotImplemented)`. Build, which regenerates `api.json`, and commit. The implementation follows in a later task.

## Breaking-change rules
These are **not allowed** without a recorded spec decision:
- removing or renaming an endpoint, parameter or response field
- making an optional request field required
- changing a type or format
- removing an enum value the client sends
- changing documented status or error codes

These are **allowed**: new endpoints, new optional request fields, new response fields, and deprecations (with a note).

**Check:** `git show <base>:backend/openapi/api.json > /tmp/base.json`, then `oasdiff breaking /tmp/base.json backend/openapi/api.json` (if installed, or `docker run --rm -v "$PWD":/w tufin/oasdiff breaking /w/…`). Otherwise, diff paths, required fields and schemas carefully by hand. Report the result.

## Frontend: generate the client
- Use one tool, recorded in CLAUDE.md. The default is `openapi-typescript` (types) plus `openapi-fetch` (a tiny typed client), with `openapi-react-query` if you want query hooks; Orval is a good alternative. The script lives in `frontend/package.json`:
  ```json
  "gen:api": "openapi-typescript ../backend/openapi/api.json -o src/api/generated/schema.ts"
  ```
- After regenerating, `npm run typecheck` lists every affected call site, and you fix them all.
- MSW handlers are typed with the generated types, so mock drift becomes a type error.

## Errors contract
Errors are ProblemDetails `{ type, title, status, detail, errorCode, errors? }`. `errorCode` is UPPER_SNAKE and comes from the spec's error table. The backend's `ErrorCodes` and the frontend's `errors.ts` must cover the same set.

## Up-to-date check (CI and gate)
```bash
(cd backend && dotnet build src/<App>.Api) && git diff --exit-code backend/openapi/api.json
(cd frontend && npm run gen:api) && git diff --exit-code frontend/src/api/generated
```

## Checklist
- [ ] `api.json` was regenerated and committed, with stable operation IDs.
- [ ] The breaking-change check was run, with the result recorded.
- [ ] The client was regenerated, and the typecheck is clean.
- [ ] Error codes match the spec on both sides.
