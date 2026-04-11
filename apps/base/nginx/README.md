# NGINX

Deploys [NGINX](https://nginx.org/) via the Bitnami Helm chart using FluxCD. Runs in the `web-apps` namespace and integrates with Vault Secrets Operator to inject database credentials at runtime.

## Files

- [release.yaml](release.yaml) — `HelmRelease` targeting the `web-apps` namespace. Helm release name is `nginx-server`. Retries up to 3 times on install/upgrade failure.
- [values.yaml](values.yaml) — `ConfigMap` with base Helm values (2 replicas, `ClusterIP` service).
- [vault-auth.yaml](vault-auth.yaml) — `VaultAuth` resource configuring Kubernetes authentication for VSO in the `web-apps` namespace.
- [db-secret.yaml](db-secret.yaml) — `VaultStaticSecret` that syncs database credentials from Vault into a Kubernetes `Secret`.
- [kustomization.yaml](kustomization.yaml) — Kustomize entry point.

## Vault Secret Injection

Database credentials are pulled from Vault automatically by the Vault Secrets Operator:

```
VaultAuth (static-auth, web-apps ns)
    └── uses service account: nginx-server
    └── Kubernetes auth role: vso-role
    └── references: vault-system/dev-vault-connection

VaultStaticSecret (nginx-db-secret)
    └── Vault path: secret/nginx/database  (kv-v2)
    └── refreshes every: 1h
    └── creates K8s Secret: nginx-db-secret
```

The resulting `nginx-db-secret` Kubernetes Secret is available in the `web-apps` namespace for NGINX pods to mount.

## Default Values

| Setting | Value |
|---------|-------|
| Replicas | 2 |
| Service type | ClusterIP |

Environment overlays (e.g. dev) override these — see [apps/overlays/dev/](../../overlays/dev/).
