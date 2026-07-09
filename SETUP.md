## 1. Instalacija alata 

Sve komande u ovom poglavlju kucaš u **PowerShell-u pokrenutom kao Administrator**

### 1.1 Chocolatey 

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

Zatvori i ponovo otvori PowerShell (kao Administrator) da `choco` postane dostupan.

### 1.2 Docker Desktop, k3d, kubectl, helm, git

```powershell
choco install docker-desktop k3d kubernetes-cli kubernetes-helm git -y
```

- **Restartuj računar** posle instalacije Docker Desktop-a (traži WSL2).
- Pokreni **Docker Desktop** i sačekaj da u donjem levom uglu piše „Engine running".
  Docker mora da radi pre svega ostalog.

Provera da je sve instalirano (u novom PowerShell prozoru, ne mora Administrator):

```powershell
docker version
k3d version
kubectl version --client
helm version
git --version
```

Ako svaka komanda ispiše verziju — spreman si.

### 1.3 Pregledač + MetaMask (za demo plaćanja)

- Koristi **Chrome** ili **Edge** (oni automatski razrešavaju `*.localhost` na 127.0.0.1).
- Instaliraj **MetaMask** ekstenziju: <https://metamask.io/> → „Add to browser".

---

## 2. Preuzimanje koda (klonirање repoa)

ShopHub je podeljen na 5 repozitorijuma (spec 5.1: svaki mikroservis svoj repo).
Za **pokretanje** ti u suštini treba samo `kube-state` 

```powershell
# napravi radni folder, npr. u Dokumentima
mkdir $HOME\Documents\GitHub ; cd $HOME\Documents\GitHub

git clone https://github.com/shophub-devoops/kube-state.git
git clone https://github.com/shophub-devoops/helm-charts.git
git clone https://github.com/shophub-devoops/shop-operator.git
git clone https://github.com/shophub-devoops/shophub.git
git clone https://github.com/shophub-devoops/shop.git
```

Sve dalje komande pokrećeš iz `kube-state` foldera:

```powershell
cd $HOME\Documents\GitHub\kube-state
```

---

## 3. Kreiranje Kubernetes klastera (k3d)

```powershell
k3d cluster create --config clusters/local/cluster.yaml
```

Provera:

```powershell
kubectl get nodes
```

---

## 4. Tajne koje se NIKAD ne čuvaju u git-u

Dve tajne moraš da napraviš ručno u klasteru (lozinke/tokeni ne idu u repo).

### 4.1 Grafana admin lozinka (obavezno)

```powershell
$GRAFANA_PASS = -join (1..32 | ForEach-Object { '{0:x}' -f (Get-Random -Maximum 16) })
foreach ($ns in 'monitoring','shop-operator-system','shophub') {
  kubectl create namespace $ns --dry-run=client -o yaml | kubectl apply -f -
  kubectl create secret generic grafana-admin -n $ns `
    --from-literal=admin-user=admin `
    --from-literal=admin-password=$GRAFANA_PASS
}
Write-Host "ZAPAMTI Grafana admin lozinku: $GRAFANA_PASS"
```

Sačuvaj ispisanu lozinku — njom se prijavljuješ u Grafanu kao maintainer.

### 4.2 Discord bot token 

1. Napravi Discord aplikaciju + bota: <https://discord.com/developers/applications>
   → New Application → Bot → Reset Token (kopiraj token).
2. Bot mora imati dozvole **Manage Channels** + **Manage Webhooks** i mora biti
   pozvan na server (guild) čiji je ID već podešen u
   `clusters/local/shophub/values.yaml` (`discord.guildId`). Ako koristiš svoj
   server, promeni taj `guildId` na ID svog servera.
3. Ubaci token u klaster:

```powershell
kubectl create namespace shophub --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic discord-bot-token -n shophub --from-literal=token=PASTE_BOT_TOKEN_OVDE
```

---

## 5. Instalacija svih komponenti

Imaš dve opcije. **Opcija A (ArgoCD)** je preporučena i bliža spec-u (GitOps,
self-healing). **Opcija B** je ručna ako ti ArgoCD pravi problem.

### Opcija A — ArgoCD (app-of-apps, preporučeno)

```powershell
# 1. instaliraj ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# sačekaj da ArgoCD podigne pod-ove (1-2 min)
kubectl -n argocd rollout status deploy/argocd-repo-server

# 2. jednokratna konfiguracija: projekat + registracija OCI registra
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/repositories.yaml

# 3. predaj ceo klaster GitOps-u (app-of-apps)
kubectl apply -f argocd/root.yaml
```

ArgoCD sad sam instalira sve komponente pravilnim redosledom (sync waves). 
Prati:

```powershell
kubectl -n argocd get applications -w
```

Čekaj dok sve aplikacije ne pređu u `Synced` / `Healthy` (prvi put 10–20 min zbog
povlačenja image-a). `Ctrl+C` da prekineš praćenje.

ArgoCD UI (opciono). Username je `admin`. `port-forward` blokira terminal, pa lozinku izvuci u drugom terminalu:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8081:443
# otvori https://localhost:8081 — username: admin, lozinka iz komande ispod 

#(DRUGI TERMINAL):

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | % { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```


### Opcija B — ručni `helm install`

```powershell
# upstream helm repozitorijumi
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# wave 0 — operatori koji daju CRD-ove
helm install cnpg cnpg/cloudnative-pg --version 0.28.2 -n cnpg-system --create-namespace -f clusters/local/cnpg/values.yaml
helm install mongodb-operator mongodb/community-operator --version 0.13.0 -n mongodb-operator --create-namespace -f clusters/local/mongodb-operator/values.yaml
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 85.3.3 -n monitoring --create-namespace -f clusters/local/kube-prometheus-stack/values.yaml

# wave 1 — logovi, tracing, naš operator
helm install loki grafana/loki-stack --version 2.10.2 -n monitoring -f clusters/local/loki/values.yaml
helm install tempo grafana/tempo --version 1.10.3 -n monitoring -f clusters/local/tempo/values.yaml
helm install shop-operator oci://registry-1.docker.io/urospetraskovic/shop-operator --version 0.1.7 -n shop-operator-system --create-namespace -f clusters/local/shop-operator/values.yaml

# wave 2 — ShopHub platforma
helm install shophub oci://registry-1.docker.io/urospetraskovic/shophub --version 0.2.1 -n shophub --create-namespace -f clusters/local/shophub/values.yaml
```


---

## 6. Provera da je sve podignuto

```powershell
kubectl get pods -A
```

Sačekaj dok svi pod-ovi ne budu `Running` (ili `Completed` za job-ove). Ključni:

```powershell
kubectl get pods -n shophub
kubectl get pods -n shop-operator-system
kubectl get pods -n monitoring
```

Ako neki pod visi u `Pending`/`ContainerCreating` prvih par minuta — to je normalno

---

## 7. Pristup aplikacijama

ShopHub (kreiranje prodavnica) | <http://shophub.localhost:8080> | registruj nalog (email + lozinka) ili Web3 wallet |

Pojedinačna prodavnica | `http://<ime-prodavnice>.localhost:8080` | kupci bez prijave; admin lozinka iz ShopHub-a |

Grafana port-forward. Username je `admin`, lozinka je ona iz koraka 4.1 (izvuci je komandom ispod):

```powershell
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
# otvori http://localhost:3000 — username: admin, lozinku vidi komandom (drugi terminal):
kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | % { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

---

## 8. Priprema MetaMask-a (Sepolia + TestUSDT)

Plaćanje radi na **Sepolia testnetu** sa našim test tokenom (nije pravi novac).

1. U MetaMask-u uključi test mreže: Settings → Advanced → „Show test networks", pa
   izaberi **Sepolia** iz liste mreža.
2. **Sepolia ETH** (za gas) — uzmi sa nekog faucet-a, npr.
   <https://sepoliafaucet.com/> ili <https://www.alchemy.com/faucets/ethereum-sepolia>
   (nalepi svoju adresu).
3. **TestUSDT token** — naš self-deployed ERC-20:
   - Ugovor (contract): `0x74b0ef872a9f1a4bbb07a01a6b4376379737ff6f`
   - Decimale: **6**
   - Faucet/mint: token ima otvoren `mint`, pa možeš sebi da dodeliš tokene
     (preko Etherscan-a „Write Contract" → `mint`, ili kako je opisano u `shop` README-u).
   - Dodaj token u MetaMask: Import tokens → nalepi adresu ugovora.

---

## 9. Gašenje / čišćenje

**Pauza (preporuka, npr. pred odbranu):** zamrzni klaster umesto brisanja — sve preživi
(slike, korisnici, prodavnice), budi se za ~2-3 min:
```powershell
k3d cluster stop local
k3d cluster start local
```

**Potpuno brisanje:**
```powershell
# obriši ceo klaster (sve nestaje)
k3d cluster delete local
```

Sledeći put kreni ponovo od poglavlja 3 (alati iz poglavlja 1 ostaju instalirani). Pažnja:
slike koje klaster povlači žive UNUTAR k3d nodova, pa se brisanjem klastera gube — novo
podizanje ih povlači ponovo (~15-20 min).
