# quantman888/.github

This repository hosts organization-level workflow templates for `quantman888`.

Control-plane reusable workflows stay in `quantman888/workflow-reusable`.
This repository only provides bootstrap templates that new repositories can pick from the GitHub Actions UI.

## Design split

- `quantman888/workflow-reusable`
  - Runtime control plane
  - GitHub App token minting
  - Runner probe and fallback
  - Docker publish/promote
  - Branch sync PR
  - Reusable workflow ref update PR
  - GitHub App secret sync control plane
- `quantman888/.github`
  - Organization workflow templates
  - New repository bootstrap entrypoints
  - Shared onboarding guidance

## Current pinned control-plane ref

All templates in this repository pin to:

- `quantman888/workflow-reusable@a7e1aa3beebbcab777534fa4319f1362eb93d2de`

Do not switch templates to floating branches.
When `workflow-reusable` changes, use the organization update workflow to open bump PRs.

## Template catalog

- `docker-publish-governed.yml`
  - Docker build/publish workflow with runner probe and central reusable workflow
  - Includes an explicit `TODO` test step that must be replaced per repository
- `docker-promote-governed.yml`
  - Digest-based promote workflow for release tags
  - Supports release body override via `OCI_SOURCE_TAG: ...`
- `reusable-workflow-update-pr.yml`
  - Opens PRs to bump pinned reusable workflow refs in caller repositories
- `workflow-ref-policy.yml`
  - Enforces full SHA or controlled `v*` tag pinning for `uses:` references
- `branch-sync-main-to-docker-pr.yml`
  - Opens or reuses PRs from default branch into `docker`
- `fork-sync-upstream-main.yml`
  - Fast-forward sync from upstream repository into the current default branch

## Recommended bootstrap order

For a new Dockerized service repository:

1. Add `workflow-ref-policy.yml`
2. Add `reusable-workflow-update-pr.yml`
3. Add `docker-publish-governed.yml`
4. Add `docker-promote-governed.yml`
5. Replace the placeholder test steps with repository-specific commands

For a fork-maintained repository:

1. Add `fork-sync-upstream-main.yml`
2. Optionally add `workflow-ref-policy.yml`
3. Optionally add `reusable-workflow-update-pr.yml`

For a branch-split repository with a long-lived `docker` branch:

1. Add `branch-sync-main-to-docker-pr.yml`
2. Add `workflow-ref-policy.yml`
3. Add `reusable-workflow-update-pr.yml`

## Required organization secrets and variables

Expected across repositories:

- `GH_APP_ID`
- `GH_APP_PRIVATE_KEY`
- `RUNNER_PROBE_TOKEN`
- `RUNNER_SELF_HOSTED_LABELS` (variable, optional)
- `RUNNER_GITHUB_HOSTED_LABEL` (variable, optional)

Docker repositories also typically need:

- `NEXUS_REGISTRY` (variable)
- `IMAGE_NAME` (variable)
- `NEXUS_USERNAME` (secret)
- `NEXUS_PASSWORD` (secret)
- `DOCKERFILE_PATH` (variable, optional)
- `BUILD_CONTEXT` (variable, optional)
- `TARGET_PLATFORMS` (variable, optional)
- `BUILD_ARGS` (variable, optional)

## Private repository onboarding

Private repositories should receive `GH_APP_ID` and `GH_APP_PRIVATE_KEY` through the control-plane workflow in `quantman888/workflow-reusable`:

- `.github/workflows/github-app-secret-sync.controlplane.yml`

That workflow scans organization repositories through the GitHub API and syncs the GitHub App secrets without hardcoded repository lists.

## Operational rule

Templates are starters, not the source of truth.
Once a repository is bootstrapped, the canonical implementation remains the workflow file inside that repository plus the reusable workflows in `quantman888/workflow-reusable`.
