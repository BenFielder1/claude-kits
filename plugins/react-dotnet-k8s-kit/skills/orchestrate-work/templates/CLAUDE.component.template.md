# <Component name> (<frontend | backend | gitops>)

<2 sentences: what this component does, who calls it or uses it, and what it depends on.>

Part of workspace: `<path to WORKSPACE.md>`. Specs live in `<specsDir>`.

## Commands
```bash
<install>        # e.g. npm ci | dotnet restore
<dev/run>        # e.g. npm run dev | dotnet run --project src/X.Api
<test>           # e.g. npm test | dotnet test
<lint/format>    # e.g. npm run lint | dotnet format --verify-no-changes
<build>          # e.g. npm run build | dotnet build -warnaserror
<generate>       # e.g. npm run gen:api | dotnet build (exports openapi/x.json)
```
**Gate:** `<single command chain that must pass before commit>`

## Layout
```
<short tree of the important folders, with one-line purposes>
```

## Conventions (detected; follow them)
<!-- Fill from the code, not from preference. Delete the blocks that don't apply. -->
<!-- Backend --> API style: <minimal APIs | controllers> · Architecture: <layers / vertical slices> · Validation: <…> · Errors: <ProblemDetails + errorCode> · Data: <EF Core / Dapper> + <provider> · Tests: <xUnit/NUnit> + <assertion lib> + <Testcontainers?> · Mapping: <manual / Mapperly / …>
<!-- Frontend --> Router: <…> · Server state: <TanStack Query / RTK Query> · Forms: <…> · Styling: <Tailwind / CSS Modules / design system> · API client: <generated via …> · Tests: <Vitest + Testing Library + MSW> · E2E: <Playwright>
<!-- GitOps --> Tool: <Helm / Kustomize> · App definition: <Application / ApplicationSet> · Secrets: <External Secrets / Sealed Secrets> · Validation: <script>

## Rules that must not be broken
1. <e.g. Never commit secrets; config comes from env/ConfigMap, secrets from <mechanism>.>
2. <e.g. Public API changes are additive only unless the spec records a breaking-change decision.>
3. <e.g. Migrations follow expand/contract; never drop/rename in the same release.>
4. <e.g. Generated files (openapi/*.json, src/api/generated/**) are never hand-edited.>
<!-- Add component-specific invariants. -->

## Definition of done
- Gate passes, and new behaviour has tests.
- Contract regenerated and checked for breaking changes (if the API changed).
- No new warnings, and no TODOs without a Jira key.
