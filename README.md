# kube-state

Željeno stanje Kubernetes klastera za **ShopHub** platformu. Ovo je IaC repo iz
specifikacije 5.3: jedini izvor istine o tome *šta* je instalirano u klasteru i
*sa kojim* override-ima. Ko god pročita ovaj repo može da digne ceo klaster od
nule.

Svaka komponenta je opisana sa dva fajla:

- `helm.yaml` — koji chart se instalira (OCI referenca ili `repo/chart` + verzija + namespace).
- `values.yaml` — override vrednosti povrh chart default-a.

## Struktura

```
kube-state
└── clusters/
    └── local/
        ├── cluster.yaml            # k3d klaster (1 server + 2 agenta)
        ├── cnpg/                   # CloudNativePG — PostgreSQL ("standard" baza)
        ├── mongodb-operator/       # MongoDB Community Operator ("light" baza, zamena za Redis)
        ├── kube-prometheus-stack/  # Prometheus + Alertmanager + Grafana + exporteri
        ├── loki/                   # Loki + Promtail (logovi)
        ├── tempo/                  # Tempo (tracing)
        ├── shop-operator/          # naš Shop operator (OCI chart iz helm-charts)
        └── shophub/                # naša ShopHub platforma (OCI chart iz helm-charts)
```

## Komponente

| Komponenta | Chart | Namespace | Svrha |
|-----------|-------|-----------|-------|
| cnpg | `cnpg/cloudnative-pg` | `cnpg-system` | PostgreSQL operator za `standard` bazu |
| mongodb-operator | `mongodb/community-operator` | `mongodb-operator` | MongoDB operator za `light` bazu (zamena za Redis, dozvoljeno spec 1.2) |
| kube-prometheus-stack | `prometheus-community/kube-prometheus-stack` | `monitoring` | Metrike, alarmi, Grafana, prometheus-operator CRD-ovi |
| loki | `grafana/loki-stack` | `monitoring` | Agregacija logova (spec 4.1) |
| tempo | `grafana/tempo` | `monitoring` | Tracing; Shop backend šalje span-ove preko OTLP/HTTP (spec 4.1) |
| shop-operator | `oci://.../urospetraskovic/shop-operator` | `shop-operator-system` | Shop operator + CRD-ovi |
| shophub | `oci://.../urospetraskovic/shophub` | `shophub` | ShopHub platforma (UI + API, na `shophub.localhost:8080`) |

> Tačne verzije su pinovane u `helm.yaml` fajlu svake komponente.

## Podizanje lokalnog klastera

> **Nova mašina / od nule?** Vidi [`SETUP.md`](SETUP.md) — pun runbook koji
> instalira i preduslove (Docker, k3d, kubectl, helm) i vodi kroz demo.
>
> **Hoćeš da obrišeš sve i proveriš svaku funkciju iz čista?** Vidi
> [`TESTING.md`](TESTING.md) — reset Docker-a + checklist provere feature po feature.

```bash
# 1. napravi k3d klaster iz cluster.yaml
k3d cluster create --config clusters/local/cluster.yaml

# 2. dodaj upstream Helm repoe iz helm.yaml fajlova
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 3. Grafana admin secret — lozinka živi SAMO u klasteru, nikad u git-u. Grafana,
# Shop operator i ShopHub je svako čita iz Secret-a grafana-admin u svom
# namespace-u (Secret se ne referencira preko namespace-a), pa ga napravi u sva tri:
GRAFANA_PASS=$(openssl rand -hex 16)
for ns in monitoring shop-operator-system shophub; do
  kubectl create namespace "$ns" --dry-run=client -o yaml | kubectl apply -f -
  kubectl create secret generic grafana-admin -n "$ns" \
    --from-literal=admin-user=admin \
    --from-literal=admin-password="$GRAFANA_PASS"
done
echo "Grafana admin lozinka: $GRAFANA_PASS"   # sačuvaj je za maintainer login

# 4. instaliraj svaku komponentu sa pinovanom verzijom + values, npr:
helm install cnpg cnpg/cloudnative-pg --version 0.28.2 \
  -n cnpg-system --create-namespace -f clusters/local/cnpg/values.yaml
# ...ponovi za kube-prometheus-stack, loki, tempo, mongodb-operator, shop-operator, shophub
```

Klaster izlaže load balancer na host portovima `8080` (→ :80) i `8443` (→ :443),
pa su storefront-ovi dostupni na `http://<shop>.localhost:8080`.

> ⚠️ **Ne brišite ručno `<shop>-app` Secret** (npr. `moja-radnja-app`). Objavljuje
> ga operator baze (CNPG/MongoDB) i sadrži konekcioni string ka bazi prodavnice —
> bez njega prodavnica ne radi. Operator ga posmatra, ali automatskog oporavka za
> ručno brisanje nema.

## GitOps (Argo CD)

Za hands-off, kontinuirano usklađen setup, [`argocd/`](argocd/) daje
**app-of-apps**: jedan `kubectl apply` predaje ceo klaster Argo CD-u, koji
instalira svaku komponentu (iz istih pinovanih verzija i `values.yaml` fajlova) i
sam ispravlja drift. Bootstrap vidi u [`argocd/README.md`](argocd/README.md). Ovo
pokriva opcioni GitOps deo spec 5.3.

## CI

- **commit-lint** — Conventional Commits.
