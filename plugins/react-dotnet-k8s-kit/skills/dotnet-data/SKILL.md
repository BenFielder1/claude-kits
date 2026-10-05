---
name: dotnet-data
description: Use when changing a .NET service's persistence (EF Core entities, configurations, DbContext, migrations, queries, seed data), or when planning schema changes that must deploy safely to Kubernetes with rolling updates.
---

# .NET data and migrations

## 0. Detect
Find the data access approach (EF Core, Dapper or both), the provider (SQL Server, PostgreSQL via Npgsql, …), where the `DbContext` and entity configurations live, the migrations project and folder, and **how migrations run in each environment**. Common setups are a migration bundle run as a Kubernetes Job or ArgoCD PreSync hook, a CI step, or the app applying them at startup. Check the GitOps repo and CI before assuming.

## 1. Model changes (EF Core)
- Configure entities with `IEntityTypeConfiguration<T>` classes, not attributes, unless the repo uses attributes.
- Be explicit: max lengths, required/optional, precision for decimals, indexes for foreign keys and query predicates, unique constraints that match business rules, and delete behaviour.
- Use a concurrency token (`rowversion`/`xmin`, or a version column) on aggregates that users edit concurrently.
- No lazy loading. Use `AsNoTracking()` for reads, project to DTOs with `Select`, and paginate every list endpoint.
- Never put raw SQL built from string interpolation into `FromSqlRaw`. Use `FromSql($"...")` (parameterised) or Dapper parameters.

## 2. Migrations
```bash
dotnet ef migrations add <PascalCaseName> --project <Infrastructure/Data project> --startup-project <Api project>
dotnet ef migrations script <PreviousMigration> <NewMigration> --idempotent --project … --startup-project …   # review the SQL
```
- Give migrations descriptive names (`AddOrderLockedAt`, not `Update3`).
- **Review the generated SQL** for unintended drops, table rebuilds or long locks. Attach the script summary to your report.
- Migrations and the model snapshot are **serial-only**. Never edit a migration that's been merged, and never hand-edit the snapshot.

## 3. Rolling-deploy safety (expand/contract)
In Kubernetes, old and new pods run at the same time during a rollout, and a rollback runs old code against the new schema. So every migration must be compatible with **both** the previous and the next app version.

| Change | Safe approach |
| --- | --- |
| Add column | Make it nullable or give it a default. Backfill. Make it required in a **later** release. |
| Rename column/table | Add the new one, write to both, backfill, switch reads, then drop the old one in a later release. |
| Drop column | Stop using it in release N, and drop it in release N+1. |
| Change type | Add a new column and migrate the data, as with a rename. |
| Add index on a large table | Use the provider's online/concurrent option, often through raw SQL in the migration, outside a transaction if required. |
| Backfill large data | Batch it in a job or a separate migration, not one giant `UPDATE` inside the schema migration. |

State in your report which release step this is (expand or contract), and what the follow-up is.

## 4. Seed and test data
- Dev seed data follows the repo's mechanism (`HasData`, a seeding service, or SQL scripts), never production data.
- Integration tests run migrations against a Testcontainers database, rather than `EnsureCreated`, so the migrations themselves get tested.

## Checklist
- [ ] Entity configuration is explicit (lengths, precision, indexes, delete behaviour).
- [ ] The migration is named well, its SQL has been reviewed, and it's safe for old and new versions to run side by side.
- [ ] Any backfill and follow-up "contract" step is documented.
- [ ] Queries are no-tracking, projected and paginated, and raw SQL is parameterised.
- [ ] Integration tests run against real migrations.
- [ ] The way migrations run in Kubernetes is unchanged, or the change has been coordinated with `platform-engineer`.
