# Changelog

## [Unreleased]

## 2026-04-23

### Added
- Kuma service mesh (chart `kuma` v2.x.x) deployed as infrastructure controller under `apps/infrastructure/dev/controllers/kuma.yaml` targeting `kuma-system`.
- Kyverno policy engine (chart `kyverno` v3.x.x) deployed as infrastructure controller under `apps/infrastructure/dev/controllers/kyverno.yaml` targeting `kyverno`.
- HelmRepository sources for Kuma (`kumahq.github.io/charts`) and Kyverno (`kyverno.github.io/kyverno/`).
- `kuma-system` and `kyverno` namespace definitions.
- `kuma.io/sidecar-injection: enabled` label on `web-apps` namespace to enable automatic Kuma sidecar injection for application pods.
- `automountServiceAccountToken: true` added to nginx base values so the `kuma-sidecar` container can read the ServiceAccount token to authenticate with the Kuma control plane.

### Changed
- Moved Vault and Vault Secrets Operator from `apps/base/` to `apps/infrastructure/dev/controllers/` — they are platform services, not application workloads.
- Moved `VaultConnection` and RBAC manifests alongside the operator in `apps/infrastructure/dev/controllers/`.
- Inlined Helm values directly in controller HelmReleases, removing the ConfigMap + `valuesFrom` indirection for infrastructure components.
- `VaultConnection.skipTLSVerify` is now set to `true` directly in the dev controller manifest rather than via overlay patch.
- VSO ClusterRoleBinding service account corrected from `dev-vault-secrets-operator` to `vault-secrets-operator` (no longer prefixed by the overlay `namePrefix`).
- Dev overlay (`apps/overlays/dev/`) now manages only nginx; vault-related bases, ConfigMap generators, and patches removed.

### Fixed
- `kuma.io/sidecar-injection` was set as an annotation instead of a label — Kuma's mutating webhook `namespaceSelector` matches namespace labels, not annotations.

## 2026-04-11

### Changed
- Refactored base/overlay ConfigMap pattern: base ConfigMaps now use a `defaults.yaml` key with real environment-agnostic defaults; overlays add an `overrides.yaml` key via `behavior: merge`, so base values are genuinely inherited rather than replaced.
- All three HelmReleases (vault, vault-operator, nginx) updated to read `defaults.yaml` then `overrides.yaml` (`optional: true`) via two `valuesFrom` entries.
- Overlay values files trimmed to env-specific overrides only — removed duplication of base defaults.
- `skipTLSVerify` moved from hardcoded `true` in base `VaultConnection` to `false` in base, patched to `true` by the dev overlay only.
- Vault-operator base `values.yaml` was empty — now defines `defaultVaultConnection.enabled: true` as a real default.
- Added `remediation.retries: 3` on install and upgrade to the Vault HelmRelease, consistent with NGINX.
- Added `wait: true` and `dependsOn: dev-infra-sync` to `apps.yaml` to enforce correct sync ordering and health gating.
- Fixed `metadata.name` / `releaseName` mismatch on the NGINX HelmRelease (both are now `nginx-server`).
- Fixed non-standard path prefixes in `infra.yaml` and `apps.yaml` (`/../` → `./`).
- Removed stale and dead-code comments across release files.

### Added
- Vault chart upgraded from `0.28.x` to `0.29.x` to address 27 HIGH/CRITICAL CVEs (including CVE-2024-41110 CRITICAL in docker/docker).
- Persistent storage enabled for Vault: 10Gi PVC on `standard` StorageClass (minikube), with `storageClass` defaulting to `""` in base for portability.
- `affinity: ""` added to Vault server and injector in dev overlay to allow scheduling on single-node minikube clusters.

## 2026-04-10

### Fixed
- Corrected service account name and `VaultAuth` reference for the `web-apps` namespace so NGINX can authenticate with Vault via the Vault Secrets Operator (`apps/base/nginx/release.yaml`, `apps/base/nginx/vault-auth.yaml`).

## 2026-03-08

### Added
- `VaultAuth` and `VaultStaticSecret` resources for NGINX to pull database credentials from Vault (`apps/base/nginx/vault-auth.yaml`, `apps/base/nginx/db-secret.yaml`).
- `VaultConnection` pointing to the internal Vault service (`apps/base/vault-operator/connection.yaml`).
- Global `ClusterRole` / `ClusterRoleBinding` so the Vault Secrets Operator can read `VaultConnection` and `VaultAuthGlobal` resources across namespaces (`apps/base/vault-operator/rbac-global.yaml`).
- Dev overlay patches to wire `VaultAuth` and `VaultStaticSecret` to the `dev-vault-connection`.

## 2026-03-07

### Added
- HashiCorp Vault Helm release and base values (`apps/base/vault/`).
- Vault Secrets Operator Helm release and base values (`apps/base/vault-operator/`).
- `vault-system` namespace definition (`apps/infrastructure/dev/namespaces/vault-system.yaml`).
- HashiCorp Helm repository source (`apps/infrastructure/dev/sources/hashicorp-repo.yaml`).
- Dev overlay values for Vault (dev mode) and the Vault Secrets Operator.

### Changed
- Restructured repository layout: moved cluster sync entrypoints under `clusters/my-local-cluster/dev/` and split infrastructure sources and namespaces into separate subdirectories under `apps/infrastructure/dev/`.
- Separated per-app Helm values into dedicated `ConfigMap` manifests (`values.yaml`) and switched HelmReleases to use `valuesFrom` instead of inline `values:` blocks.
- NGINX dev overlay updated to apply `namePrefix` patches for ConfigMap references.

## 2026-03-04

### Added
- Initial NGINX Helm release targeting the `web-apps` namespace using the Bitnami OCI chart (`apps/base/nginx/`).
- `web-apps` namespace definition.
- Bitnami Helm repository source.
- Dev overlay for NGINX (1 replica, `ClusterIP` service).
- Cluster-level Flux `Kustomization` resources for infrastructure and apps sync (`clusters/my-local-cluster/dev/infra.yaml`, `apps.yaml`).

### Added (bootstrap)
- Flux v2.8.1 component manifests generated by `flux bootstrap` (`clusters/my-local-cluster/flux-system/gotk-components.yaml`).
- Flux `GitRepository` and root `Kustomization` sync manifests (`clusters/my-local-cluster/flux-system/gotk-sync.yaml`).
