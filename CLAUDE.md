# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is a thin Helm "umbrella chart" that wraps HashiCorp's official `vault` chart
(`https://helm.releases.hashicorp.com`, pinned in `Chart.yaml`) with this organization's
specific configuration in `values.yaml`. There is no application source code here —
changes are almost entirely version bumps in `Chart.yaml`/`values.yaml` and config tweaks.

## Repo layout

- `Chart.yaml` — chart metadata and the pinned `vault` subchart version.
- `values.yaml` — all deployment-specific overrides (image tags, TLS, storage, injector config).
- `workflow.yaml` — an Argo Workflow that deploys this chart via a shared `flavor-helm-template`
  (from another repo), targeting a given `git_ref_type`/`git_version` (branch/tag) of this repo.
- `.github/workflows/helm_branches.yml` — CI entrypoint. It checks out this repo plus a sibling
  `bradfordwagner/taskfiles` repo, then runs `task -t ./taskfiles/tasks/helm.yml` (using
  `go-task`, not Make) to lint/package the chart on every push. The actual task definitions live
  in that external `taskfiles` repo, not here.

## Key configuration details (values.yaml)

- Vault and the injector run from custom mirrored images at `ghcr.io/bradfordwagner/vault-mirror`
  (not the upstream HashiCorp images) — when bumping versions, update the image `tag` to match
  the new upstream Vault/injector version using this mirror's tag convention
  (`<mirror-version>-vault_<vault-version>` / `<mirror-version>-vault-webhook-injector_<injector-version>`).
- Storage backend config is intentionally *not* defined in `values.yaml`. It's injected at deploy
  time via `extraArgs: -config=/vault/userconfig/storage/config.hcl` and an `extraSecret` named
  `storage`, sourced from Terraform/CI, specifically to keep credentials out of this repo (see the
  comment in `values.yaml` referencing the Vault k8s docs on protecting sensitive configuration).
- Auto-unseal uses Azure Key Vault (`seal "azurekeyvault" {}`); the required Azure credentials are
  supplied via `extraSecretEnvironmentVars` pulling from a Kubernetes secret named `keyvault`.
- `dataStorage.enabled` is `false` because storage is backed by Azure Blob (handled outside this
  chart), not a PVC.
- `vault.server.service.type` is `NodePort` with a fixed `nodePort: 30003`.

## Making changes

There is no local build/lint/test tooling in this repo — validation happens in CI via the
external `taskfiles` repo's `helm.yml` task, or by running `helm lint` / `helm template` manually
against this chart (remember to `helm dependency update` first to pull the pinned `vault` subchart).
Version bumps to the chart itself should update the `version` field in `Chart.yaml`.
