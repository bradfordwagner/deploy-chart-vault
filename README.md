# deploy-chart-vault

Helm chart that deploys [HashiCorp Vault](https://www.vaultproject.io/) on Kubernetes, wrapping the
official [`vault`](https://github.com/hashicorp/vault-helm) chart with this organization's
configuration (image mirrors, TLS, Azure Key Vault auto-unseal, and injected storage config).

## Chart

| | |
|---|---|
| Chart name | `bradfordwagner-deploy-vault` |
| Chart version | see [`Chart.yaml`](Chart.yaml) |
| Subchart | `vault` from `https://helm.releases.hashicorp.com`, pinned in [`Chart.yaml`](Chart.yaml) |

## Configuration highlights

See [`values.yaml`](values.yaml) for the full configuration. Notable points:

- Vault server and the Agent Injector run from mirrored images at
  `ghcr.io/bradfordwagner/vault-mirror` rather than the upstream HashiCorp images.
- Storage backend configuration is **not** committed to this repo. It's supplied at deploy time via
  `-config=/vault/userconfig/storage/config.hcl`, mounted from a Kubernetes secret named `storage`
  (provisioned by Terraform/CI), to keep credentials out of version control. See
  [Protecting Sensitive Vault Configurations](https://www.vaultproject.io/docs/platform/k8s/helm/run#protecting-sensitive-vault-configurations).
- Auto-unseal uses Azure Key Vault (`seal "azurekeyvault" {}`), with credentials sourced from a
  Kubernetes secret named `keyvault`.
- TLS is enabled by default (`global.tlsDisable: false`), with the certificate/key mounted from a
  `tls` secret.
- The Vault service is exposed as a `NodePort` on port `30003`.
- Persistent volume storage is disabled (`dataStorage.enabled: false`) since data is backed by
  Azure Blob storage rather than a PVC.

## Deploying

```sh
helm dependency update
helm upgrade --install vault . -f values.yaml
```

## CI/CD

- [`.github/workflows/helm_branches.yml`](.github/workflows/helm_branches.yml) runs on every push,
  checking out this repo alongside `bradfordwagner/taskfiles` and running
  `task -t ./taskfiles/tasks/helm.yml` to lint/package the chart.
- [`workflow.yaml`](workflow.yaml) is an Argo Workflow definition that deploys this chart via a
  shared `flavor-helm-template`, for a given branch/tag of this repo.
