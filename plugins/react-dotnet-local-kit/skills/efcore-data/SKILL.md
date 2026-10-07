---
name: efcore-data
description: Use when changing the backend's persistence with Microsoft EF Core and the dotnet-ef tool, covering entities, DbContext, configurations, migrations, seed data, queries, the dev database and how migrations run on local Kubernetes.
---

# EF Core data and migrations

## Setup (once, done by the scaffolder)
- Install the tool locally: `dotnet new tool-manifest` (in `backend/`), then `dotnet tool install dotnet-ef`. Run it as `dotnet ef …` (or `dotnet dotnet-ef …`), at the version that matches the EF Core packages.
- Use one provider package, following the spec: `Microsoft.EntityFrameworkCore.SqlServer` (the default) or `Npgsql.EntityFrameworkCore.PostgreSQL`. Add `Microsoft.EntityFrameworkCore.Design` to the startup project (`<App>.Api`).
- Register `AddDbContext<AppDbContext>(o => o.Use<Provider>(config.GetConnectionString("Default")))`.
- **Dev database:** the `db` service in `docker-compose.yml`, with the connection string in user-secrets (`dotnet user-secrets set ConnectionStrings:Default "…" --project src/<App>.Api`).
  - SQL Server: `mcr.microsoft.com/mssql/server:2022-latest` (or the current GA tag), with `ACCEPT_EULA=Y`, an `MSSQL_SA_PASSWORD` from `.env`, and a health check. On Apple Silicon, it runs under Docker Desktop's x86 emulation, and start-up is slower.
  - PostgreSQL: `postgres:<major>`, with `POSTGRES_PASSWORD` from `.env`.

## Model
- Configure entities with `IEntityTypeConfiguration<T>`, applied by `modelBuilder.ApplyConfigurationsFromAssembly(...)`.
- Be explicit: `HasMaxLength`, required/optional, `HasPrecision` for decimals, indexes for foreign keys and query predicates, unique indexes for business uniqueness (including filtered or case-insensitive ones, e.g. a case-insensitive collation on SQL Server or `citext` / `lower()` on PostgreSQL), and `OnDelete` behaviour.
- Add a concurrency token on aggregates that are edited concurrently (`IsRowVersion()` on SQL Server, `xmin` / a version column on PostgreSQL). It maps to 409 `CONFLICT` in the API.
- Keep **derived values derived**. Compute them in queries or the domain rather than storing them, unless the spec says otherwise.
- Queries: use `AsNoTracking()` for reads, project with `Select` to DTOs, paginate lists, and avoid N+1 (project rather than `Include` chains). Raw SQL only through `FromSql($"…")`, which is parameterised.

## Migrations
```bash
cd backend
dotnet ef migrations add <PascalCaseName> --project src/<App>.Infrastructure --startup-project src/<App>.Api
dotnet ef migrations script <Previous> <New> --idempotent --project src/<App>.Infrastructure --startup-project src/<App>.Api   # review
dotnet ef database update --project src/<App>.Infrastructure --startup-project src/<App>.Api                                  # dev DB
```
- Give migrations descriptive names, and **review the SQL** for drops, table rebuilds and data loss. Summarise anything risky in your report.
- Never edit a committed migration, and never hand-edit the model snapshot. Migrations are **serial-only**.
- **Rolling-update safety**: Kubernetes runs old and new API pods side by side during a rollout. Prefer additive changes (nullable or defaulted columns, new tables). For renames, drops and type changes, use expand/contract across two phases or releases, and note the follow-up.

## Seed data
- Use `HasData` only for true reference data (lookup tables).
- For demo and dev data, use an idempotent seeder (`IHostedService` or a `--seed` command) that runs **only** when `Seed:Enabled=true`. The local Kubernetes overlay enables it when the spec asks for demo data.

## Migrations on local Kubernetes
- Build a **migration bundle** in the API image's build stage: `dotnet ef migrations bundle --self-contained -r linux-x64 -o /app/efbundle --project … --startup-project …`. Use `linux-arm64` on Apple Silicon, or build for the Docker Desktop node's architecture as detected by the deploy script.
- The `db-migrate` Job (in `deploy/k8s/base`) runs `/app/efbundle --connection "$ConnectionStrings__Default"`. The deploy script runs it and waits for it to complete **before** rolling out the API. The API does **not** migrate at start-up.
- Coordinate any change to this mechanism with `local-k8s-engineer`.

## Tests
Integration tests start a Testcontainers database of the same engine and apply the real migrations (`Database.MigrateAsync()`), so the migrations themselves are tested.

## Checklist
- [ ] Explicit configuration (lengths, precision, indexes, delete behaviour, concurrency).
- [ ] One well-named migration, with its SQL reviewed and safe for rolling updates (or the follow-up noted).
- [ ] The dev DB was updated, and the integration tests run against migrations.
- [ ] Seed data is idempotent and gated by config.
- [ ] The migration bundle and Job still work (or `local-k8s-engineer` has been told).
