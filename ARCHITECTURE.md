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

    subgraph infra["Infrastructure layer"]
        NS1["Namespace\nvault-system"]
        NS2["Namespace\nweb-apps"]
        REPO1["HelmRepository\nhashicorp-repo"]
        REPO2["HelmRepository\nbitnami-repo"]
    end

    subgraph releases["HelmReleases (flux-system)"]
        HR_V["HelmRelease\ndev-vault"]
        HR_VSO["HelmRelease\ndev-vault-secrets-operator\ndependsOn: dev-vault"]
        HR_N["HelmRelease\ndev-nginx-server"]
    end

    GH -->|"fetch every 1m"| GR
    GR --> INFRA
    GR --> APPS
    APPS -->|"dependsOn"| INFRA
    INFRA --> NS1 & NS2 & REPO1 & REPO2
    APPS --> HR_V & HR_VSO & HR_N
    REPO1 -->|"chart source"| HR_V & HR_VSO
    REPO2 -->|"chart source"| HR_N
```

---

## 2. In-Cluster Resources

What gets deployed into each namespace.

```mermaid
flowchart TD
    subgraph vault-system["vault-system"]
        VAULT["Vault\nStatefulSet\nvault-0\nv1.18.x"]
        PVC["PVC\ndata-vault-0\n10Gi · standard StorageClass"]
        VSO["Vault Secrets Operator\nDeployment"]
        VC["VaultConnection\ndev-vault-connection\nhttp://vault.vault-system:8200\nskipTLSVerify: true ①"]
    end

    subgraph web-apps["web-apps"]
        NGINX["NGINX\nDeployment\n1 replica (dev)"]
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

    note1["① dev overlay patches\nskipTLSVerify to true.\nBase default is false."]
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

## 4. Base/Overlay Values Pattern

How Helm values are layered across environments.

```mermaid
flowchart LR
    subgraph base["apps/base/vault/"]
        BV["values.yaml\nConfigMap\nkey: defaults.yaml\n─────────────────\ndataStorage:\n  enabled: true\n  size: 10Gi\n  storageClass: ''\n  ..."]
    end

    subgraph overlay["apps/overlays/dev/"]
        OV["vault-values.yaml\n─────────────────\nserver:\n  dev.enabled: false\n  affinity: ''\ninjector:\n  affinity: ''"]
        KZ["kustomization.yaml\nconfigMapGenerator\nbehavior: merge\noverrides.yaml=vault-values.yaml"]
    end

    subgraph merged["Merged ConfigMap in cluster\ndev-vault-values"]
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

---

## 5. Repository Layout

```
flux-fleet-infra/
├── clusters/
│   └── my-local-cluster/
│       ├── flux-system/          ← Flux bootstrap (generated, do not edit)
│       └── dev/
│           ├── infra.yaml        ← Kustomization → apps/infrastructure/dev
│           └── apps.yaml         ← Kustomization → apps/overlays/dev
│
└── apps/
    ├── infrastructure/
    │   └── dev/
    │       ├── namespaces/       ← vault-system, web-apps
    │       └── sources/          ← HelmRepository (hashicorp, bitnami)
    │
    ├── base/
    │   ├── vault/                ← HelmRelease + defaults ConfigMap
    │   ├── vault-operator/       ← HelmRelease + defaults ConfigMap + VaultConnection + RBAC
    │   └── nginx/                ← HelmRelease + defaults ConfigMap + VaultAuth + VaultStaticSecret
    │
    └── overlays/
        └── dev/
            ├── kustomization.yaml          ← namePrefix: dev-, configMapGenerator, patches
            ├── vault-values.yaml           ← dev overrides (affinity, dev mode)
            ├── vault-operator-values.yaml  ← dev overrides (Vault address)
            └── nginx-values.yaml           ← dev overrides (1 replica)
```
