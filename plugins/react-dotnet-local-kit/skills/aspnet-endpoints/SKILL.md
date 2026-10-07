---
name: aspnet-endpoints
description: Use when building or changing the ASP.NET Core backend in backend/, including pure domain rules, minimal API endpoints, validation, ProblemDetails errors, auth, configuration, health checks, logging and backend tests.
---

# ASP.NET Core backend

If the backend already has code, **mirror the nearest existing feature** (structure, naming, test style) before you apply the defaults below.

## Project structure
- `<App>.Domain`: entities' behaviour, value objects and business rules. **No** EF, ASP.NET or I/O. Time comes from `TimeProvider`, and randomness goes through an injected seam.
- `<App>.Infrastructure`: `DbContext`, configurations, migrations, external clients (owned by `ef-data-engineer` for persistence).
- `<App>.Api`: endpoints, DI, auth, ProblemDetails, OpenAPI, health. `Program.cs` stays readable, with feature registration in `Features/<Feature>/<Feature>Endpoints.cs` extension methods.
- Shared settings in `Directory.Build.props`: `Nullable` enabled, `TreatWarningsAsErrors`, `ImplicitUsings`, analyzers on. Package versions go in `Directory.Packages.props` (central package management).

## Endpoints
```csharp
public static class OrdersEndpoints
{
    public static IEndpointRouteBuilder MapOrders(this IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/api/orders").WithTags("Orders").RequireAuthorization();
        g.MapGet("/{id:guid}", GetOrder).WithName("GetOrder");
        g.MapPost("/{id:guid}/lock", LockOrder).WithName("LockOrder");
        return app;
    }

    static async Task<Results<Ok<OrderDto>, NotFound>> GetOrder(Guid id, AppDbContext db, CancellationToken ct) =>
        await db.Orders.AsNoTracking().Where(o => o.Id == id).Select(OrderDto.Projection).FirstOrDefaultAsync(ct)
            is { } dto ? TypedResults.Ok(dto) : TypedResults.NotFound();
}
```
- Use `TypedResults` with the `Results<…>` return types, so the OpenAPI document lists every status.
- Give each endpoint a stable `.WithName("OperationId")`, which becomes the generated client's method name.
- Pass a `CancellationToken` to every async call. Never use `.Result` or `.Wait()`.
- Keep endpoints thin: validate, then load, then call the domain, then save, then map to a DTO. DTOs are `record`s in the Api project, and entities are never returned.

## Validation and errors
- Validate requests at the edge, using the repo's chosen approach: DataAnnotations with the built-in minimal-API validation where the .NET version supports it, or FluentValidation. Return `TypedResults.ValidationProblem(errors)`.
- Register `AddProblemDetails()` and an `IExceptionHandler` that maps unexpected exceptions to a 500 ProblemDetails without stack traces outside Development.
- Business errors return a ProblemDetails with the right status code and an `errorCode` extension taken from the spec's error table:
  ```csharp
  TypedResults.Problem(statusCode: 409, title: "Order is locked",
      extensions: new Dictionary<string, object?> { ["errorCode"] = "ORDER_LOCKED" });
  ```
  Keep the codes in one static class (`ErrorCodes`), so the backend and the spec stay aligned.

## Auth (when the spec needs accounts)
- The default if the spec doesn't choose: **ASP.NET Core Identity with cookie auth** (`AddIdentityApiEndpoints<AppUser>()` plus `MapIdentityApi<AppUser>()`, or custom endpoints if the spec needs other fields). The frontend is served same-origin through `/api`, so cookies work without CORS. Use secure cookie settings and SameSite=Lax/Strict.
- Authorise with **policies** (`RequireAuthorization("CanEditOrder")`), plus a resource check in the handler or domain for ownership. Never trust client-supplied user IDs.
- If the spec names an external identity provider, use JWT bearer validation against it instead, and document the config keys.

## Configuration, health and logging
- Use `IOptions<T>` with `.ValidateDataAnnotations().ValidateOnStart()`. The connection string comes from `ConnectionStrings__Default` (an env var in Kubernetes, user-secrets in development). There are no secrets in `appsettings*.json`.
- Health checks: `/api/health/live` (no dependency checks) and `/api/health/ready` (including `AddDbContextCheck<AppDbContext>()`). Kubernetes probes use these.
- Logging: `ILogger<T>` with message templates, never interpolation. No secrets or PII in logs.
- Bind Kestrel to `http://+:8080` in containers (the default for recent ASP.NET images), and `5080` for the dev loop via `launchSettings.json`.

## Tests
- **Unit** (`<App>.UnitTests`): every domain rule and the spec's worked examples. Name tests after the requirement, e.g. `FR3_locking_a_shipped_order_fails`.
- **Integration** (`<App>.IntegrationTests`): `WebApplicationFactory<Program>` plus a PostgreSQL Testcontainer (`Testcontainers.PostgreSql`), shared per test collection, with migrations applied (not `EnsureCreated`). Cover success, 400, 401, 403 (another user's resource), 404, and each error code. Add `public partial class Program;` if needed.
- If Docker isn't available, say so. Don't substitute an in-memory provider for behaviour that depends on the database.

## Packages
Prefer what's already referenced. **Don't add packages whose recent versions are commercially licensed** without asking. Some popular libraries changed licence in recent major versions, such as MediatR, AutoMapper and FluentAssertions. Check the licence of the version you'd install.

## Gate
`cd backend && dotnet build -warnaserror && dotnet test && dotnet format --verify-no-changes`

## Checklist
- [ ] Domain rules are pure and unit-tested by requirement ID.
- [ ] Endpoints use TypedResults, have stable operation IDs, are thin, and pass CancellationTokens.
- [ ] Errors are ProblemDetails with the spec's `errorCode`s.
- [ ] Policies and ownership checks are on every protected endpoint.
- [ ] Config is validated on start, with no secrets in appsettings.
- [ ] OpenAPI is regenerated (`openapi-contract`).
