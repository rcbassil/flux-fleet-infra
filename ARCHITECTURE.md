# Architecture

## 1. GitOps Sync Flow

How Flux pulls from Git and reconciles the cluster.

```mermaid
flowchart TD
    GH["GitHub\nrcbassil/flux-fleet-infra\nmain branch"]

    subgraph flux-system["flux-system (Flux controllers)"]
        GR["GitRepository\nflux-system"]
        INFRA["Kustomization\ndev-infra-sync\ninterval: 1h"]
        APPS["Kustomization\ndev-apps-sync\ninterval: 10m · wait: true"]
    end

    subgraph infra["Infrastructure layer (dev-infra-sync)"]
        NS1["Namespace\nvault-system"]
        NS2["Namespace\nweb-apps\n(kuma injection: enabled)"]
        NS3["Namespace\nkuma-system"]
        NS4["Namespace\nkyverno"]
        REPO1["HelmRepository\nhashicorp-repo"]
        REPO2["HelmRepository\nbitnami-repo"]
        REPO3["HelmRepository\nkuma-repo"]
        REPO4["HelmRepository\nkyverno-repo"]
        HR_V["HelmRelease\nvault"]
        HR_VSO["HelmRelease\nvault-secrets-operator\ndependsOn: vault"]
        HR_K["HelmRelease\nkuma"]
        HR_KY["HelmRelease\nkyverno"]
    end

    subgraph apps["Application layer (dev-apps-sync)"]
        HR_N["HelmRelease\ndev-nginx-server"]
    end

    GH -->|"fetch every 1m"| GR
    GR --> INFRA
    GR --> APPS
    APPS -->|"dependsOn"| INFRA
    INFRA --> NS1 & NS2 & NS3 & NS4
    INFRA --> REPO1 & REPO2 & REPO3 & REPO4
    INFRA --> HR_V & HR_VSO & HR_K & HR_KY
    APPS --> HR_N
    REPO1 -->|"chart source"| HR_V & HR_VSO
    REPO2 -->|"chart source"| HR_N
    REPO3 -->|"chart source"| HR_K
    REPO4 -->|"chart source"| HR_KY
```

---

## 2. In-Cluster Resources

What gets deployed into each namespace.

```mermaid
flowchart TD
    subgraph kuma-system["kuma-system"]
        KUMA["Kuma Control Plane\nDeployment\nv2.13.x"]
        KWEB["Mutating Webhook\nnmspaceSelector:\nkuma.io/sidecar-injection=enabled"]
    end

    subgraph kyverno["kyverno"]
        KYV["Kyverno Admission Controller\nDeployment\n1 replica (dev)"]
    end

    subgraph vault-system["vault-system"]
        VAULT["Vault\nStatefulSet\nvault-0\nv1.18.x"]
        PVC["PVC\ndata-vault-0\n10Gi · standard StorageClass"]
        VSO["Vault Secrets Operator\nDeployment"]
        VC["VaultConnection\nvault-connection\nhttp://vault.vault-system:8200\nskipTLSVerify: true"]
    end

    subgraph web-apps["web-apps  (kuma.io/sidecar-injection: enabled)"]
        NGINX["NGINX\nDeployment\n1 replica (dev)"]
        SIDECAR["kuma-sidecar\nEnvoy proxy"]
        VA["VaultAuth\ndev-static-auth\nKubernetes auth · role: vso-role"]
        VSS["VaultStaticSecret\ndev-nginx-db-secret\npath: secret/nginx/database\nrefresh: 1h"]
        SEC["Secret\nnginx-db-secret"]
    end

    VAULT <-->|"hostPath volume"| PVC
    VSO -->|"reads"| VC
    VSO -->|"reads"| VA
    VSO -->|"reads"| VSS
    VA -->|"vaultConnectionRef"| VC
    VSS -->|"fetches credentials"| VAULT
    VSS -->|"creates & refreshes"| SEC
    SEC -->|"mounted into"| NGINX
    KUMA -->|"injects"| SIDECAR
    SIDECAR -.->|"mTLS proxy"| NGINX
    KWEB -->|"intercepts pod create\nin labeled namespaces"| SIDECAR
```

---

## 3. Vault Secret Injection

How a Kubernetes Secret gets created from a Vault path.

```mermaid
sequenceDiagram
    participant VSO as Vault Secrets Operator
    participant VC as VaultConnection
    participant VA as VaultAuth
    participant VSS as VaultStaticSecret
    participant Vault as Vault (vault-0)
    participant K8s as Kubernetes API

    VSO->>VC: resolve Vault address
    VSO->>VA: get Kubernetes auth config
    VA-->>VSO: mount=kubernetes, role=vso-role, SA=nginx-server
    VSO->>Vault: authenticate (ServiceAccount token)
    Vault-->>VSO: Vault token
    VSO->>VSS: read secret spec
    VSS-->>VSO: path=secret/nginx/database
    VSO->>Vault: read secret/nginx/database
    Vault-->>VSO: { username, password, ... }
    VSO->>K8s: create/update Secret nginx-db-secret
    Note over VSO,K8s: Repeats every refreshAfter: 1h
```

---

## 4. Base/Overlay Values Pattern (Applications)

How Helm values are layered across environments for application workloads.

```mermaid
flowchart LR
    subgraph base["apps/base/nginx/"]
        BV["values.yaml\nConfigMap\nkey: defaults.yaml\n─────────────────\nreplicaCount: 2\nautomountServiceAccountToken: true\nservice.type: ClusterIP"]
    end

    subgraph overlay["apps/overlays/dev/"]
        OV["nginx-values.yaml\n─────────────────\nreplicaCount: 1"]
        KZ["kustomization.yaml\nconfigMapGenerator\nbehavior: merge\noverrides.yaml=nginx-values.yaml"]
    end

    subgraph merged["Merged ConfigMap in cluster\ndev-nginx-values"]
        M1["key: defaults.yaml → base content"]
        M2["key: overrides.yaml → dev content"]
    end

    subgraph hr["HelmRelease valuesFrom"]
        F1["1. defaults.yaml (base)"]
        F2["2. overrides.yaml (dev wins on conflict)"]
    end

    BV -->|"behavior: merge"| merged
    OV --> KZ --> merged
    merged --> F1 & F2
    F1 & F2 -->|"merged by Helm"| HELM["Helm chart values"]
```

Infrastructure controllers (Vault, Kuma, Kyverno) use inline `values:` in their HelmRelease instead of this pattern — no ConfigMap indirection needed since they are single-environment.

---

## 5. Repository Layout

```
flux-fleet-infra/
├── clusters/
│   └── my-local-cluster/
│       ├── flux-system/          ← Flux bootstrap (generated, do not edit)
│       └── dev/
│           ├── infra.yaml        ← Kustomization → apps/infrastructure/dev (1h)
│           └── apps.yaml         ← Kustomization → apps/overlays/dev (10m, dependsOn infra)
│
└── apps/
    ├── infrastructure/
    │   └── dev/
    │       ├── sources/          ← HelmRepository (hashicorp, bitnami, kuma, kyverno)
    │       ├── namespaces/       ← vault-system, web-apps, kuma-system, kyverno
    │       └── controllers/      ← HelmReleases with inline values
    │           ├── vault.yaml
    │           ├── vault-operator.yaml
    │           ├── vault-connection.yaml
    │           ├── vault-rbac.yaml
    │           ├── kuma.yaml
    │           └── kyverno.yaml
    │
    ├── base/
    │   └── nginx/                ← HelmRelease + defaults ConfigMap + VaultAuth + VaultStaticSecret
    │
    └── overlays/
        └── dev/
            ├── kustomization.yaml   ← namePrefix: dev-, configMapGenerator, patches
            └── nginx-values.yaml    ← dev overrides (1 replica)
```
