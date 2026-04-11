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
│           └── apps.yaml         # Syncs apps/overlays/dev (10m interval)
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

| App | Chart | Namespace | Version |
|-----|-------|-----------|---------|
| [Vault](apps/base/vault/) | hashicorp/vault | vault-system | 0.28.x |
| [Vault Secrets Operator](apps/base/vault-operator/) | hashicorp/vault-secrets-operator | vault-system | 0.9.x |
| [NGINX](apps/base/nginx/) | bitnami/nginx | web-apps | 22.x.x |

## How It Works

### Base/Overlay Pattern

Each app in `apps/base/` defines a `HelmRelease` and a `ConfigMap` of default Helm values. Environment overlays in `apps/overlays/<env>/` patch those definitions — applying a `namePrefix`, swapping ConfigMap references, and injecting environment-specific values via `configMapGenerator`.

### Infrastructure Before Apps

The cluster's dev entrypoint has two Flux `Kustomization` resources:

1. **`infra.yaml`** — reconciles every 1h, ensures namespaces and `HelmRepository` sources exist first.
2. **`apps.yaml`** — reconciles every 10m, depends on infra being ready.

### Vault Integration

NGINX pulls database credentials from Vault using the Vault Secrets Operator:

```
VaultConnection (cluster-wide)
    └── VaultAuth (per namespace, Kubernetes auth method)
            └── VaultStaticSecret (per app, synced every 1h)
                    └── Kubernetes Secret (consumed by the pod)
```

RBAC is defined in [apps/base/vault-operator/rbac-global.yaml](apps/base/vault-operator/rbac-global.yaml) to allow VSO to read `VaultConnection` resources across namespaces.

### Persistent Storage (Vault)

Vault data is stored on a 10Gi `PersistentVolumeClaim` using the `standard` StorageClass (minikube's hostPath provisioner). Data lands inside the minikube VM at `/tmp/hostpath-provisioner/vault-system/`. See [apps/base/vault/README.md](apps/base/vault/README.md) for details.

## Prerequisites

- [minikube](https://minikube.sigs.k8s.io/)
- [flux CLI](https://fluxcd.io/flux/installation/)
- A GitHub personal access token with repo access

## Bootstrap

```sh
minikube start

flux bootstrap github \
  --owner=rcbassil \
  --repository=flux-fleet-infra \
  --branch=main \
  --path=clusters/my-macos-cluster \
  --personal
```

Once bootstrapped, Flux will reconcile all infrastructure and applications automatically.

## Adding a New Application

1. Create `apps/base/<app>/` with a `release.yaml` (HelmRelease), `values.yaml` (ConfigMap), and `kustomization.yaml`.
2. Add environment-specific values to `apps/overlays/dev/<app>-values.yaml` and reference them in the overlay `kustomization.yaml`.
3. If the app needs a new Helm source, add a `HelmRepository` under `apps/infrastructure/dev/sources/`.
