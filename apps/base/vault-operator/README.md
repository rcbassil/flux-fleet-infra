# Vault Secrets Operator

Deploys the [HashiCorp Vault Secrets Operator (VSO)](https://developer.hashicorp.com/vault/docs/platform/k8s/vso) via the official Helm chart using FluxCD. VSO bridges Vault and Kubernetes by syncing secrets from Vault into native Kubernetes `Secret` resources.

## Files

- [release.yaml](release.yaml) — `HelmRelease` targeting the `vault-system` namespace. Depends on the `vault` HelmRelease being ready before reconciling.
- [values.yaml](values.yaml) — `ConfigMap` containing Helm values passed to the chart via `valuesFrom`.
- [connection.yaml](connection.yaml) — `VaultConnection` resource pointing to the internal Vault service at `http://vault.vault-system.svc.cluster.local:8200`.
- [rbac-global.yaml](rbac-global.yaml) — `ClusterRole` and `ClusterRoleBinding` granting VSO read access to `VaultConnection` and `VaultAuthGlobal` resources across all namespaces.
- [kustomization.yaml](kustomization.yaml) — Kustomize entry point.

## How It Works

VSO introduces three CRDs that apps use to pull secrets from Vault:

| CRD | Purpose |
|-----|---------|
| `VaultConnection` | Cluster-wide connection config (address, TLS settings) |
| `VaultAuth` | Per-namespace authentication method (Kubernetes service account) |
| `VaultStaticSecret` | Per-app secret definition — what path to read, how often to refresh |

The operator watches these resources and reconciles them into Kubernetes `Secret` objects that pods can consume normally.

## VaultConnection

Defined in [connection.yaml](connection.yaml). Points to the Vault service via in-cluster DNS:

```
http://vault.vault-system.svc.cluster.local:8200
```

TLS verification is skipped (`skipTLSVerify: true`) since Vault runs without TLS in the dev environment.

## RBAC

The `ClusterRole` in [rbac-global.yaml](rbac-global.yaml) allows the VSO service account (`dev-vault-secrets-operator`) to `get`, `list`, and `watch` `VaultConnection` and `VaultAuthGlobal` resources cluster-wide. This is required for cross-namespace secret syncing.

## Dependency

The `HelmRelease` has `dependsOn: vault`, ensuring Vault is running and ready before VSO attempts to connect to it.
