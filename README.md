# flux-fleet-infra

GitOps repository for managing Kubernetes applications using [FluxCD](https://fluxcd.io/). Applications are defined as Helm releases composed with Kustomize, following a base/overlay pattern for environment-specific configuration.

## Repository Structure

```
.
├── clusters/
│   └── my-macos-cluster/         # minikube local cluster
│       ├── flux-system/          # Flux bootstrap manifests (generated)
│       └── dev/
│           ├── infra.yaml        # Syncs apps/infrastructure/dev (1h interval)
│           └── apps.yaml         # Syncs apps/overlays/dev (10m interval), depends on infra
└── apps/
    ├── base/                     # Environment-agnostic Helm release definitions
    │   ├── vault/
    │   ├── vault-operator/
    │   └── nginx/
    ├── infrastructure/
    │   └── dev/                  # Helm sources and namespace definitions
    │       ├── sources/          # HelmRepository resources
    │       └── namespaces/       # Namespace resources
    └── overlays/
        └── dev/                  # Dev-specific patches and values
```

## Applications

| App                                                 | Chart                            | Namespace    | Version |
| --------------------------------------------------- | -------------------------------- | ------------ | ------- |
| [Vault](apps/base/vault/)                           | hashicorp/vault                  | vault-system | 0.29.x  |
| [Vault Secrets Operator](apps/base/vault-operator/) | hashicorp/vault-secrets-operator | vault-system | 0.9.x   |
| [NGINX](apps/base/nginx/)                           | bitnami/nginx                    | web-apps     | 22.x.x  |

## How It Works

### Base/Overlay Pattern

Each app in `apps/base/` defines a `HelmRelease` and a `ConfigMap` with a `defaults.yaml` key containing environment-agnostic Helm values. Environment overlays in `apps/overlays/<env>/` use `configMapGenerator` with `behavior: merge` to add an `overrides.yaml` key on top — so base defaults are always inherited and only env-specific values need to be listed in the overlay.

Each `HelmRelease` reads both keys in order via two `valuesFrom` entries:

```yaml
valuesFrom:
  - kind: ConfigMap
    name: app-values
    valuesKey: defaults.yaml # base defaults, always present
  - kind: ConfigMap
    name: app-values
    valuesKey: overrides.yaml # env-specific overrides, optional
    optional: true
```

### Infrastructure Before Apps

The cluster's dev entrypoint has two Flux `Kustomization` resources:

1. **`infra.yaml`** — reconciles every 1h, ensures namespaces and `HelmRepository` sources exist first.
2. **`apps.yaml`** — reconciles every 10m, explicitly depends on infra via `dependsOn`, and waits for all HelmReleases to be healthy via `wait: true`.

### Vault Integration

NGINX pulls database credentials from Vault using the Vault Secrets Operator:

```
VaultConnection (cluster-wide, skipTLSVerify patched per env)
    └── VaultAuth (per namespace, Kubernetes auth method)
            └── VaultStaticSecret (per app, synced every 1h)
                    └── Kubernetes Secret (consumed by the pod)
```

`skipTLSVerify` defaults to `false` in base. The dev overlay patches it to `true` for minikube. A production overlay would leave it unpatched, keeping TLS verification enabled.

RBAC is defined in [apps/base/vault-operator/rbac-global.yaml](apps/base/vault-operator/rbac-global.yaml) to allow VSO to read `VaultConnection` resources across namespaces.

### Persistent Storage (Vault)

Vault data is stored on a 10Gi `PersistentVolumeClaim`. The `storageClass` defaults to `""` (cluster default) in base. The dev overlay sets it to `standard` (minikube's hostPath provisioner). Data lands inside the minikube VM at `/tmp/hostpath-provisioner/vault-system/`. See [apps/base/vault/README.md](apps/base/vault/README.md) for details.

After any Vault pod restart, Vault comes back sealed and must be manually unsealed:

```sh
kubectl exec -n vault-system vault-0 -- vault operator unseal <key>
# repeat with 3 of your 5 unseal keys
```

## Prerequisites

- [minikube](https://minikube.sigs.k8s.io/)
- [flux CLI](https://fluxcd.io/flux/installation/)
- A GitHub personal access token with repo access

## Bootstrap

```sh
minikube start

flux bootstrap github \
  --owner=<your-github-username> \
  --repository=flux-fleet-infra \
  --branch=main \
  --path=clusters/<your-cluster-name> \
  --personal
```

Once bootstrapped, Flux will reconcile all infrastructure and applications automatically.

## Adding a New Application

1. Create `apps/base/<app>/` with:
   - `release.yaml` — HelmRelease with two `valuesFrom` entries (`defaults.yaml` and `overrides.yaml`)
   - `values.yaml` — ConfigMap with a `defaults.yaml` key containing base Helm values
   - `kustomization.yaml` — listing both files as resources
2. Add `apps/overlays/dev/<app>-values.yaml` with only dev-specific overrides, and wire it into the overlay `kustomization.yaml` using `behavior: merge` and `overrides.yaml=<app>-values.yaml`.
3. Add patches to `apps/overlays/dev/kustomization.yaml` to update both `valuesFrom[0].name` and `valuesFrom[1].name` to the `dev-`-prefixed ConfigMap name.
4. If the app needs a new Helm source, add a `HelmRepository` under `apps/infrastructure/dev/sources/`.
