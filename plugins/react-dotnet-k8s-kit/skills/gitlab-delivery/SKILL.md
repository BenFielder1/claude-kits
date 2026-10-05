---
name: gitlab-delivery
description: Use for Git/GitLab workflow and CI in React or .NET app repos, covering branches, commits, merge requests (glab), .gitlab-ci.yml pipelines, test reports, container image builds, and handing image versions to the GitOps repo.
---

# Git, GitLab and CI

**Default posture:** prepare and validate everything locally. Push, create MRs, run pipelines, or change CI/CD variables **only when the user asks**.

## Branches, commits and MRs
- Branch names: `feature/<JIRA-KEY>-<slug>` (or the repo's convention, e.g. `bugfix/…`). Never commit to protected branches.
- Commit messages: `<JIRA-KEY> <imperative summary>` with FR IDs in the body when relevant. The Jira key links commits through the GitLab–Jira integration.
- MRs (only when asked):
  ```bash
  glab mr create --draft --title "<KEY> <summary>" --description "$(cat mr.md)" \
    --target-branch <default> --source-branch <branch> --label "<labels>"
  ```
  - The description covers the summary, Jira link, FRs covered, how it was tested, contract changes (with the breaking-change check result), migrations (whether expand or contract), config or secrets changes, and **related MRs with the merge order**.
  - Use the repo's `.gitlab/merge_request_templates/` if it exists.

## CI pipelines (`.gitlab-ci.yml`)
**First, check for shared templates.** Many organisations `include:` central CI templates (`include: project: group/ci-templates`). If they do, extend those rather than writing jobs from scratch. Check `include:`, `extends:` and the group's template project.

Typical shape:
```yaml
stages: [verify, build, publish, deploy-update]

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

# .NET
dotnet-verify:
  stage: verify
  image: mcr.microsoft.com/dotnet/sdk:<version>   # match global.json
  variables: { NUGET_PACKAGES: "$CI_PROJECT_DIR/.nuget/packages" }
  cache: { key: { files: ["**/packages.lock.json", "Directory.Packages.props"] }, paths: [.nuget/packages] }
  script:
    - dotnet restore
    - dotnet format --verify-no-changes --no-restore
    - dotnet build -c Release -warnaserror --no-restore
    - dotnet test -c Release --no-build --logger "junit;LogFilePath=TestResults/{assembly}.xml" --collect:"XPlat Code Coverage"
  artifacts:
    when: always
    reports:
      junit: "**/TestResults/*.xml"
      coverage_report: { coverage_format: cobertura, path: "**/coverage.cobertura.xml" }

# Frontend
web-verify:
  stage: verify
  image: node:<lts>
  cache: { key: { files: [package-lock.json] }, paths: [.npm/] }
  script:
    - npm ci --cache .npm --prefer-offline
    - npm run typecheck && npm run lint
    - npm test -- --reporter=default --reporter=junit --outputFile=junit.xml
    - npm run build
  artifacts: { when: always, reports: { junit: junit.xml } }

# Image (use the org's builder: buildah, a Kaniko fork, or docker buildx with dind)
image:
  stage: publish
  rules: [{ if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH }, { if: $CI_COMMIT_TAG }]
  script:
    - <build and push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA (and :$CI_COMMIT_TAG)>
```
- The JUnit logger for `dotnet test` requires the `JunitXml.TestLogger` package. Check what the test projects reference, or use the TRX logger plus a converter, following convention.
- **Tag images immutably** with the commit SHA and/or release tag. Never deploy `latest`.
- **Handing off to GitOps:** follow the organisation's existing pattern:
  - **(a)** a CI job that commits the new image tag to the GitOps repo's **dev** values, or opens an MR for higher environments, using a project or group access token stored as a masked, protected CI variable;
  - **(b)** ArgoCD Image Updater watching the registry;
  - **(c)** a manual promotion MR.
  Don't introduce a new mechanism without a spec decision.
- **Secrets** live in masked, protected CI/CD variables or the organisation's secret manager. Never put them in YAML.

## Validate locally
- `glab ci lint` (needs glab auth and network access to the GitLab instance), if available.
- Otherwise, check the YAML syntax and run each job's script commands locally in the component.
- Report what was validated and what can only be verified by a real pipeline.

## Checklist
- [ ] Branch and commit naming include the Jira key.
- [ ] Pipeline jobs mirror the component's gate, and test and coverage reports are wired up.
- [ ] Shared CI templates are reused, not duplicated.
- [ ] Images are tagged immutably, and the GitOps handoff matches the existing pattern.
- [ ] No secrets in the repo or YAML.
- [ ] The MR description lists related MRs and the merge order (if MRs were requested).
