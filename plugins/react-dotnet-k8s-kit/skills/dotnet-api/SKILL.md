---
name: dotnet-api
description: Use when building or changing a C# / ASP.NET Core backend service, covering domain logic, application handlers, HTTP endpoints, validation, errors, auth, configuration, health checks, logging and tests.
---

# .NET service workflow

## 0. Detect before you write
Existing services have conventions, and matching them matters more than any default below. Before writing code, look at:
- the solution layout (`*.sln`/`*.slnx`, `src/`, `tests/`), the architecture (layers such as Domain/Application/Infrastructure/Api, or vertical slices), and the target framework (`global.json`, `<TargetFramework>`);
- the API style (minimal APIs with `MapGroup`, or controllers), the validation (FluentValidation, DataAnnotations or hand-written), the error format, the mapping (manual, Mapperly, …), and any mediator;
- the test stack (xUnit/NUnit/MSTest, the assertion library, `WebApplicationFactory`, Testcontainers);
- package management (`Directory.Packages.props` central management, `Directory.Build.props` shared settings).
Mirror the closest existing feature: copy its structure and naming, not just its idea.

## 1. Defaults for new code (when nothing exists yet)
- **The current .NET LTS**, with `<Nullable>enable</Nullable>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`, `<ImplicitUsings>enable</ImplicitUsings>`.
- **Minimal APIs** grouped by feature: `app.MapGroup("/api/v1/orders").WithTags("Orders").RequireAuthorization()`. Return `TypedResults` (`Results<Ok<OrderDto>, NotFound, ValidationProblem>`) so OpenAPI is accurate.
- **Errors as ProblemDetails** (RFC 9457): register `AddProblemDetails()` and an `IExceptionHandler` for unexpected errors. Domain and business errors return a ProblemDetails with an `errorCode` extension (`"ORDER_LOCKED"`) and the right status code. The spec's error table is the source of the codes.
- **Validation** at the edge (request DTOs). Invariants live in the domain.
- **Configuration**: use `IOptions<T>` with `.ValidateDataAnnotations().ValidateOnStart()`. Read secrets from env vars or mounted files, never from `appsettings*.json`.
- **Auth**: authorisation **policies** (`RequireAuthorization("orders:write")`), and resource checks in the handler or domain. Never trust IDs from the client without an ownership check.
- **Async all the way**: pass a `CancellationToken` through every async call, and never use `.Result` or `.Wait()`.
- **Logging**: `ILogger<T>` with message templates (`"Order {OrderId} locked"`), never string interpolation. No PII or secrets in logs. Use the existing OpenTelemetry setup if there is one.
- **Health**: `/health/live` (process up) and `/health/ready` (dependencies such as the DB), for Kubernetes probes. Check the existing paths in the GitOps values before renaming anything.
- **HTTP clients**: `IHttpClientFactory` with typed clients and the resilience handler (`AddStandardResilienceHandler`) if the repo already uses `Microsoft.Extensions.Http.Resilience`.

## 2. Domain logic
- Put business rules in the domain layer (or the slice's domain types) as plain C#: no EF, no HttpContext, no static clock (`TimeProvider` is injected), no randomness without a seam.
- Enforce invariants through constructors and methods, not public setters. Model expected failures with a result type or domain exceptions, following the repo's convention.
- Unit-test every rule and edge case from the spec's acceptance criteria. Name each test after the FR (e.g. `FR3_rejects_edit_when_order_locked`).

## 3. Contract
Every endpoint change regenerates the OpenAPI document. Follow the `api-contract` skill. When the brief says **contract-first**, add the DTOs and endpoint signatures with handlers that return `TypedResults.StatusCode(501)`, regenerate the contract, and commit it before writing the implementation.

## 4. Tests
- **Unit**: domain and application logic.
- **Integration**: `WebApplicationFactory<Program>` plus a real database through Testcontainers, if Docker is available and the repo already uses it. Cover the happy path, validation (400), auth (401/403), not found (404), and each business error code.
- If you can't run Testcontainers, say so. Don't swap in an in-memory provider for behaviour that depends on the real database.

## 5. Dependencies and licences
- Add packages only when needed. With central package management, versions go in `Directory.Packages.props`, which is serial-only.
- **Don't add packages whose recent versions are commercially licensed** without asking. Some popular libraries changed licence in recent major versions, including MediatR, AutoMapper and FluentAssertions. Check the licence of the version you'd install, and prefer what the repo already uses.

## Gate
`dotnet build -warnaserror && dotnet test && dotnet format --verify-no-changes`, or the component's own gate from its CLAUDE.md.

## Checklist
- [ ] Follows the conventions of the nearest existing feature.
- [ ] Every FR in scope has a test named after it.
- [ ] Errors use ProblemDetails with the spec's error codes and statuses.
- [ ] Authorisation is enforced on every new endpoint, including the resource-ownership check.
- [ ] OpenAPI is regenerated, and the breaking-change check passes or a decision is recorded.
- [ ] No secrets in config files, logs use templates, and there are no new warnings.
