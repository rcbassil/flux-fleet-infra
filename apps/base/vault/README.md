# Vault

Deploys [HashiCorp Vault](https://www.vaultproject.io/) via the official Helm chart using FluxCD.

## Files

- [release.yaml](release.yaml) — `HelmRelease` targeting the `vault-system` namespace, sourced from the `hashicorp-repo` HelmRepository.
- [values.yaml](values.yaml) — `ConfigMap` containing Helm values passed to the chart via `valuesFrom`.
- [kustomization.yaml](kustomization.yaml) — Kustomize entry point.

## Persistent Storage

Vault data is stored on a `10Gi` PersistentVolumeClaim mounted at `/vault/data` inside the container. The StorageClass is set to `standard` (minikube's default hostPath provisioner).

The actual data resides inside the minikube VM under `/tmp/hostpath-provisioner/vault-system/`. To inspect it:

```sh
minikube ssh
ls /tmp/hostpath-provisioner/vault-system/
```

## Upgrading the Chart

The chart version is pinned to `0.28.x` in [release.yaml](release.yaml). To upgrade, update the `version` field and let Flux reconcile.
