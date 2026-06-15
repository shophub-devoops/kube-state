# ShopHub — vodič za instalaciju i pokretanje od nule

Ovaj dokument vodi **bilo koga sa praznim Windows laptopom** (samo Windows 10/11,
bez ijednog DevOps alata) kroz instalaciju svega potrebnog i pokretanje cele
ShopHub platforme lokalno, do tačke gde se može demonstrirati kreiranje
prodavnice, kupovina kriptovalutom, metrike i alarmi.

> **Bitno:** za **pokretanje** projekta ne treba ni Go, ni Node.js, ni Python,
> ni kubebuilder. Sve aplikacije su unapred zapakovane u Docker image-e koji se
> povlače sa DockerHub-a. Treba ti samo: Docker, k3d, kubectl, helm, Git i
> pregledač sa MetaMask ekstenzijom. (Go/Node/kubebuilder su potrebni samo ako
> bi neko **menjao i ponovo gradio** kod — za odbranu nisu.)

Ceo proces traje ~15–30 min na svežem laptopu (najviše vremena ode na povlačenje
Docker image-a). Ako se kasnije ponovo pokrene na istom laptopu, traje par minuta
jer su image-i već keširani.

---

## 0. Šta ćemo dobiti

Posle ovog vodiča imaćeš lokalni Kubernetes klaster (k3d) sa:

- **ShopHub** platformom na `http://shophub.localhost:8080` (korisnik kreira/briše prodavnice),
- **Shop operatorom** koji od `Shop` CRD-a pravi pravu prodavnicu (deployment + baza + ingress + dashboard + alarmi),
- bazama preko operatora (**CNPG** za PostgreSQL, **MongoDB** community operator za „light"),
- **observability** stekom: Prometheus + Grafana + Alertmanager + Loki (logovi) + Tempo (tracing),
- svaka prodavnica dobija svoj **Grafana dashboard** i **Discord alarme**,
- plaćanje **USDT na Sepolia testnetu** preko MetaMask-a.

---

## 1. Instalacija alata (jednom po laptopu)

Sve komande u ovom poglavlju kucaš u **PowerShell-u pokrenutom kao Administrator**
(Start → ukucaj „PowerShell" → desni klik → „Run as administrator").

### 1.1 Chocolatey (paket-menadžer, da ostalo instaliramo jednom komandom)

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

Provera da je sve instalirano (u **novom** PowerShell prozoru, ne mora Administrator):

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
  (Podešavanje naloga/mreže je u poglavlju 8.)

---

## 2. Preuzimanje koda (klonirање repoa)

ShopHub je podeljen na 5 repozitorijuma (spec 5.1: svaki mikroservis svoj repo).
Za **pokretanje** ti u suštini treba samo `kube-state` (on opisuje ceo klaster),
ali kloniraj sve da imaš celu sliku.

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

Ovo pravi klaster `local` (1 server + 2 agenta) i mapira load balancer:
host **:8080 → :80** i **:8443 → :443**. Zato su prodavnice dostupne na
`http://<ime-prodavnice>.localhost:8080`.

Provera:

```powershell
kubectl get nodes
```

Treba da vidiš 3 noda u stanju `Ready`.

---

## 4. Tajne koje se NIKAD ne čuvaju u git-u

Dve tajne moraš da napraviš ručno u klasteru (lozinke/tokeni ne idu u repo).

### 4.1 Grafana admin lozinka (obavezno)

Grafana, Shop operator i ShopHub je čitaju iz Secret-a `grafana-admin` u **svom**
namespace-u (Secret se ne deli između namespace-ova), pa pravimo isti u sva tri:

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

### 4.2 Discord bot token (opciono — za alarme na Discord)

Potrebno samo ako želiš da demonstriraš alarme koji stižu na Discord (spec D10).
Ako preskočiš, sve ostalo radi; prodavnice se prave bez Discord kanala.

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

> JWT secret za ShopHub se **ne pravi ručno** — helm chart ga sam generiše i čuva.

---

## 5. Instalacija svih komponenti

Imaš dve opcije. **Opcija A (ArgoCD)** je preporučena i bliža spec-u (GitOps,
self-healing). **Opcija B** je ručna ako ti ArgoCD pravi problem.

### Opcija A — ArgoCD (app-of-apps, preporučeno)

```powershell
# 1. instaliraj ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# sačekaj da ArgoCD podigne pod-ove (1-2 min)
kubectl -n argocd rollout status deploy/argocd-repo-server

# 2. jednokratna konfiguracija: projekat + registracija OCI registra
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/repositories.yaml

# 3. predaj ceo klaster GitOps-u (app-of-apps)
kubectl apply -f argocd/root.yaml
```

ArgoCD sad sam instalira sve komponente pravilnim redosledom (sync waves). Prati:

```powershell
kubectl -n argocd get applications -w
```

Čekaj dok sve aplikacije ne pređu u `Synced` / `Healthy` (prvi put 10–20 min zbog
povlačenja image-a). `Ctrl+C` da prekineš praćenje.

ArgoCD UI (opciono):

```powershell
kubectl -n argocd port-forward svc/argocd-server 8081:443
# otvori https://localhost:8081 (admin lozinka:)
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

(Tačne verzije i namespace-ovi su izvor istine u `clusters/local/*/helm.yaml`.)

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
(povlačenje image-a). Ako stoji duže, vidi poglavlje 10.

---

## 7. Pristup aplikacijama

| Šta | URL | Prijava |
|-----|-----|---------|
| ShopHub (kreiranje prodavnica) | <http://shophub.localhost:8080> | registruj nalog (email + lozinka) ili Web3 wallet |
| Pojedinačna prodavnica | `http://<ime-prodavnice>.localhost:8080` | kupci bez prijave; admin lozinka iz ShopHub-a |
| Grafana (maintainer) | port-forward, vidi dole | admin / lozinka iz koraka 4.1 |

Grafana port-forward:

```powershell
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
# otvori http://localhost:3000
```

> `*.localhost` radi automatski u Chrome/Edge. Ako koristiš `curl` ili neki drugi
> alat koji ne razrešava `*.localhost`, dodaj liniju `127.0.0.1 shophub.localhost`
> (i za svaku prodavnicu) u `C:\Windows\System32\drivers\etc\hosts`.

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

## 9. Demo scenario (end-to-end)

1. Otvori <http://shophub.localhost:8080>, **registruj** nalog.
2. **New shop**: ime (npr. `odeca`), availability `high` (3 replike) ili `standard`
   (2), baza `postgres` ili `mongodb`, wallet adresa (ili dugme **Generate**),
   čekiraj Discord ako želiš alarme. Sačekaj da prodavnica postane `Ready`.
3. Klikni **Open** → otvara se `http://odeca.localhost:8080`.
4. U prodavnici uđi kao **admin** (lozinka: u ShopHub-u klikni ikonicu ključa na
   kartici prodavnice) → dodaj artikle (naziv, cena, količina).
5. Kao kupac: pretraži artikle → dodaj u korpu → **Pay with USDT** → potvrdi u
   MetaMask-u → sačekaj on-chain potvrdu → porudžbina prelazi u `confirmed`.
6. **Grafana**: prijavi se kao admin, otvori dashboard te prodavnice (HTTP saobraćaj,
   404-ke, jedinstveni posetioci, CPU/RAM, itd.).
7. **Alarmi**: izazovi greške (npr. više puta otvori nepostojeći URL prodavnice da
   nakupiš 404/5xx) → alarm se okida → notifikacija stiže na Discord kanal te
   prodavnice (ako si podesio Discord u koraku 4.2).

---

## 10. Troubleshooting

**Pod u `ImagePullBackOff` / `ErrImagePull`.**
Najčešće traženi tag image-a nije objavljen na DockerHub-u. Proveri koji image fali:

```powershell
kubectl describe pod -n shophub -l app.kubernetes.io/name=shophub | Select-String -Pattern "Image|Failed"
```

Rešenja:
- Uveri se da su DockerHub repozitorijumi `urospetraskovic/shophub-backend`,
  `urospetraskovic/shop-backend` i `urospetraskovic/shop-operator-controller` **public**.
- Ako fali tačan verzionisani tag (npr. `:0.2.1`), a postoji `:main`, privremeno
  pregazi tag: za ShopHub `--set image.tag=main` pri `helm install`/u values, za
  operator `--set shopImage=docker.io/urospetraskovic/shop-backend:main`. (Trajno
  rešenje je pushovati odgovarajući git tag pa CI objavi verzionisani image.)

**`shophub.localhost` se ne otvara.** Koristi Chrome/Edge, ili dodaj host u
`hosts` fajl (vidi napomenu u poglavlju 7). Proveri i da k3d radi: `k3d cluster list`.

**Sve visi u `Pending` jako dugo.** Proveri da Docker Desktop ima dovoljno
resursa (Settings → Resources: bar 4 GB RAM, idealno 6–8 GB).

**ArgoCD app `OutOfSync`/`Degraded`.** Otvori UI (poglavlje 5A) i pogledaj poruku;
najčešće je u pitanju nedostajuća tajna iz koraka 4 ili još traje povlačenje image-a.

**Baza prodavnice se ne diže.** Proveri da su CNPG / MongoDB operatori `Running`
pre kreiranja prodavnice (`kubectl get pods -n cnpg-system -n mongodb-operator`).

---

## 11. Gašenje / čišćenje

```powershell
# obriši ceo klaster (sve nestaje)
k3d cluster delete local
```

Sledeći put kreni ponovo od poglavlja 3 (alati iz poglavlja 1 ostaju instalirani,
Docker image-i ostaju keširani pa je drugo pokretanje znatno brže).
