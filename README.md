# kube-state

Desired state of the Kubernetes cluster(s) for the **ShopHub** platform. This is
the IaC repository required by spec section 5.3: it is the single source of truth
for *what* is installed in the cluster and *with which* overrides. Anyone reading
this repo can bring the whole cluster up from scratch.

Each component is described by two files:

- `helm.yaml` — which chart to install (OCI reference or `repo/chart` + version + namespace).
- `values.yaml` — value overrides applied on top of the chart defaults.

## Structure

```
kube-state
├── README.md
└── clusters/
    └── local/
        ├── cluster.yaml            # k3d cluster definition (1 server + 2 agents)
        ├── cnpg/                   # CloudNativePG — PostgreSQL ("standard" DB)
        ├── mongodb-operator/       # MongoDB Community Operator ("light" DB, Redis substitute)
        ├── kube-prometheus-stack/  # Prometheus + Alertmanager + Grafana + exporters
        ├── loki/                   # Loki + Promtail (log aggregation)
        ├── tempo/                  # Tempo (distributed tracing)
        ├── shop-operator/          # our Shop operator (OCI chart from helm-charts)
        └── shophub/                # our ShopHub platform (OCI chart from helm-charts)
```

## Components

| Component | Chart | Version | Namespace | Purpose |
|-----------|-------|---------|-----------|---------|
| cnpg | `cnpg/cloudnative-pg` | 0.28.2 | `cnpg-system` | PostgreSQL operator for the `standard` database option |
| mongodb-operator | `mongodb/community-operator` | 0.13.0 | `mongodb-operator` | MongoDB operator for the `light` database option (Redis substitute, allowed by spec 1.2) |
| kube-prometheus-stack | `prometheus-community/kube-prometheus-stack` | 85.3.3 | `monitoring` | Metrics, alerting, Grafana, and the prometheus-operator CRDs (ServiceMonitor, PrometheusRule) |
| loki | `grafana/loki-stack` | 2.10.2 | `monitoring` | Log aggregation (spec 4.1 "logging") |
| tempo | `grafana/tempo` | 1.10.3 | `monitoring` | Distributed tracing; Shop backend pushes spans over OTLP/HTTP (spec 4.1 "tracing") |
| shop-operator | `oci://.../urospetraskovic/shop-operator` | 0.1.4 | `shop-operator-system` | The Shop operator + its CRDs |
| shophub | `oci://.../urospetraskovic/shophub` | 0.1.1 | `shophub` | The ShopHub platform backend |

## Bringing up the local cluster

```bash
# 1. create the k3d cluster from cluster.yaml
k3d cluster create --config clusters/local/cluster.yaml

# 2. add the upstream Helm repos referenced by the helm.yaml files
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 3. install each component with its pinned version + values, e.g.:
helm install cnpg cnpg/cloudnative-pg --version 0.28.2 \
  -n cnpg-system --create-namespace -f clusters/local/cnpg/values.yaml

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 85.3.3 \
  -n monitoring --create-namespace -f clusters/local/kube-prometheus-stack/values.yaml

# ...repeat for loki, tempo, mongodb-operator, shop-operator, shophub
```

The cluster exposes the load balancer on host ports `8080` (→ :80) and
`8443` (→ :443), so Shop storefronts are reachable at
`http://<shop>.localhost:8080`.

## GitOps (Argo CD)

The manual flow above is the simplest way to stand the cluster up. For a
hands-off, continuously-reconciled setup, [`argocd/`](argocd/) provides an
**app-of-apps**: one `kubectl apply` hands the whole cluster to Argo CD, which
installs every component (from the same pinned versions and `values.yaml` files
via multi-source Applications) and self-heals drift. See
[`argocd/README.md`](argocd/README.md) for the bootstrap. This satisfies the
optional GitOps part of spec 5.3.

## CI

- **commit-lint** — enforces Conventional Commits on PR titles/commits.
