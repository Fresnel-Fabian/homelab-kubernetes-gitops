# homelab-kubernetes-gitops

GitOps configuration for a 6-node homelab Kubernetes cluster managed by [Flux v2](https://fluxcd.io/). All cluster state is declared here — no manual `kubectl apply` or `helm install` after bootstrap.

## Cluster

| Role | Count | RAM | Nodes |
|---|---|---|---|
| Control Plane | 3 | 2.4–2.5 GB each | k8s-cp-01, k8s-cp-02, k8s-cp-03 |
| Worker | 3 | 10 GB each | k8s-w-01, k8s-w-02, k8s-w-03 |

- **Kubernetes**: 1.32.3 (kubeadm)
- **CNI**: Calico (via Tigera Operator)
- **Load Balancer**: MetalLB — IP pool `192.168.0.240–250`
- **Storage**: Local Path Provisioner
- **External Access**: Tailscale

---

## How Flux Works Here

Flux watches this repository on a 10-minute interval. Any change pushed to `main` is automatically reconciled into the cluster. The sync entrypoint is `cluster/homelab/` — Flux reads the Kustomization CRs there and fans out to the rest of the tree.

```
Git push → source-controller detects new revision
         → kustomize-controller applies infrastructure/
         → (waits for health checks)
         → kustomize-controller applies infrastructure/configs/
         → (waits for dependsOn)
         → helm-controller applies apps/ (HelmReleases)
```

The three Kustomization CRs enforce strict ordering via `dependsOn`:

```
infrastructure  →  infrastructure-configs  →  apps
(operators)         (CRs/config)               (workloads)
```

This ensures cert-manager and MetalLB are fully ready before their CRs are applied, and all infrastructure is stable before workloads start.

---

## Repository Structure

```
homelab-kubernetes-gitops/
│
├── cluster/homelab/                  # Flux entrypoint — sync target
│   ├── flux-system/                  # Flux's own manifests (self-managed)
│   │   ├── gotk-components.yaml      # Controllers: source, kustomize, helm, notification
│   │   ├── gotk-sync.yaml            # GitRepository + root Kustomization CRs
│   │   └── kustomization.yaml
│   ├── infrastructure.yaml           # Kustomization CR → infrastructure/controllers/
│   ├── infrastructure-configs.yaml   # Kustomization CR → infrastructure/configs/
│   └── apps.yaml                     # Kustomization CR → apps/homelab/
│
├── infrastructure/
│   ├── controllers/                  # Operators and system components
│   │   ├── kustomization.yaml
│   │   ├── metallb/                  # MetalLB operator (HelmRelease)
│   │   │   ├── namespace.yaml
│   │   │   ├── helmrepository.yaml
│   │   │   ├── helmrelease.yaml
│   │   │   └── kustomization.yaml
│   │   ├── local-path-provisioner/   # Storage provisioner (manifest)
│   │   │   ├── deploy.yaml
│   │   │   └── kustomization.yaml
│   │   └── metrics-server/           # Resource metrics API (manifest)
│   │       └── kustomization.yaml
│   │
│   └── configs/                      # CRs that depend on controllers being ready
│       └── metallb/
│           ├── ipaddresspool.yaml    # IP pool: 192.168.0.240-250 (L2 mode)
│           └── kustomization.yaml
│
└── apps/
    ├── base/                         # Reusable base manifests (environment-agnostic)
    └── homelab/                      # Homelab-specific overlays and releases
        ├── kustomization.yaml
        ├── monitoring/               # kube-prometheus-stack
        │   ├── namespace.yaml
        │   ├── helmrepository.yaml
        │   ├── prometheus.yaml       # HelmRelease
        │   ├── prometheus-values.yaml
        │   ├── prometheus-values-cm.yaml
        │   └── kustomization.yaml
        ├── logging/                  # Loki + Promtail + Grafana
        │   ├── namespace.yaml
        │   ├── helmrepository-grafana.yaml
        │   ├── helmrepository-grafana-community.yaml
        │   ├── loki.yaml             # HelmRelease (loki-6.x standalone)
        │   ├── loki-values.yaml
        │   ├── promtail.yaml         # HelmRelease
        │   ├── grafana.yaml          # HelmRelease
        │   ├── grafana-values.yaml
        │   └── kustomization.yaml
        └── dbs/                      # PostgreSQL (manifest)
```

---

## Deployed Services

### Infrastructure (Flux-managed, `infrastructure/`)

| Component | Namespace | How Deployed | Purpose |
|---|---|---|---|
| MetalLB | `metallb-system` | HelmRelease | L2 load balancer, IP pool `240–250` |
| Local Path Provisioner | `local-path-storage` | Manifest | Node-local PVC storage |
| Metrics Server | `kube-system` | Manifest | `kubectl top`, HPA signal |

### Infrastructure Configs (`infrastructure/configs/`)

| Resource | Namespace | Description |
|---|---|---|
| IPAddressPool | `metallb-system` | `192.168.0.240–192.168.0.250` |
| L2Advertisement | `metallb-system` | Advertises pool over L2 |

### Applications (`apps/homelab/`)

| Service | Namespace | Chart | External IP | Purpose |
|---|---|---|---|---|
| kube-prometheus-stack | `monitoring` | `prometheus-community/kube-prometheus-stack` | — | Prometheus + Alertmanager + Node Exporter |
| Loki | `logging` | `grafana/loki` 6.x | — | Log aggregation (distributed mode) |
| Promtail | `logging` | `grafana/promtail` | — | Log shipper (DaemonSet) |
| Grafana | `logging` | `grafana/grafana` | `192.168.0.243:80` | Dashboards (Prometheus + Loki datasources) |

### Not Yet Migrated to Flux

| Component | Namespace | Status |
|---|---|---|
| Calico / Tigera Operator | `calico-system`, `tigera-operator` | Manifest-applied directly |
| cert-manager | `cert-manager` | Manifest-applied directly |
| AMD GPU Device Plugin | `kube-system` | Manifest-applied directly |
| PostgreSQL | `dbs` | Manifest |
| MinIO | `ml` | Scaled to 0, pending migration |
| MLflow | `ml` | Scaled to 0, pending migration |
| Jupyter Notebook | `ml` | Scaled to 0, pending migration |

---

## Bootstrap

> Only needed once on a fresh cluster or after a full disaster recovery.

### Prerequisites

```bash
# Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# GitHub token with repo Contents (read/write) + Metadata (read)
export GITHUB_TOKEN=<token>
export GITHUB_USER=<username>
```

### Run Bootstrap

```bash
flux bootstrap github \
  --owner=${GITHUB_USER} \
  --repository=homelab-kubernetes-gitops \
  --branch=main \
  --path=cluster/homelab \
  --personal \
  --token-auth
```

Bootstrap installs the Flux controllers into `flux-system`, creates a deploy key on the repo, pushes its own manifests to `cluster/homelab/flux-system/`, and begins reconciling immediately. After this point, all changes go through Git.

---

## Day-to-Day Operations

### Deploy a Change

```bash
# Edit any manifest or values file
vim apps/homelab/monitoring/prometheus-values.yaml

git add . && git commit -m "feat: adjust prometheus retention to 30d"
git push origin main

# Wait up to 10 minutes, or force immediately:
flux reconcile kustomization apps --with-source
```

### Check Sync Status

```bash
# All Flux-managed resources
flux get all -A

# Watch reconciliation live
flux get kustomizations --watch
flux get helmreleases -A --watch
```

### Force Reconcile

```bash
# Pull latest from Git and reconcile everything
flux reconcile source git flux-system

# Reconcile a specific layer
flux reconcile kustomization infrastructure --with-source
flux reconcile kustomization apps --with-source

# Reconcile a single HelmRelease
flux reconcile helmrelease prometheus -n flux-system --with-source
```

### Debug a Failing Reconciliation

```bash
# Kustomization not syncing
kubectl describe kustomization apps -n flux-system

# HelmRelease stuck or failing
flux logs --kind=HelmRelease --name=loki -n flux-system --since=1h

# See exactly what Flux would change without applying
flux diff kustomization apps
```

### Emergency Manual Intervention

```bash
# Suspend Flux ownership (safe to kubectl edit while suspended)
flux suspend kustomization apps

# ... make manual changes ...

# Hand control back to Flux
flux resume kustomization apps
```

---

## Adding a New Application

1. Create a directory under `apps/homelab/<name>/`
2. Add a `kustomization.yaml` listing your resources
3. If Helm: add a `HelmRepository` + `HelmRelease` + values ConfigMap
4. If manifest: add cleaned YAML (`kubectl neat` to strip runtime fields)
5. Add the new directory to `apps/homelab/kustomization.yaml`
6. Commit and push — Flux picks it up within 10 minutes

---

## Key Design Decisions

**Why `cluster/` not `clusters/`** — personal preference, single-cluster homelab, no ambiguity.

**Why values in ConfigMaps** — `HelmRelease.spec.valuesFrom` references a ConfigMap so values stay in Git as plain YAML, not embedded inside the HelmRelease spec. Easier to diff and review.

**Why `infrastructure → infrastructure-configs → apps` ordering** — MetalLB CRDs must exist before `IPAddressPool` is applied. cert-manager CRDs must exist before `Certificate` objects. `dependsOn` enforces this without manual sequencing.

**Why Local Path Provisioner over Longhorn** — control plane nodes have 2.4–2.5 GB RAM. Longhorn's memory overhead caused pod evictions. Local Path has near-zero overhead; the tradeoff is no replication across nodes.

**Why standalone Loki 6.x over loki-stack** — `loki-stack` is deprecated. The standalone chart supports distributed/microservices mode and is the current maintained path.