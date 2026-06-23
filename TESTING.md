# ShopHub — čišćenje, pokretanje i testiranje svega (vodič za početnike)

Ovaj dokument te vodi kroz: **(1)** brisanje svega do praznog Docker-a, **(2)**
pokretanje celog projekta iz nule, i **(3)** proveru **svake** funkcionalnosti
— od kreiranja prodavnice i plaćanja kriptom do Grafane i Discord alarma.

Pisan je za nekoga ko **ne zna mnogo o DevOps-u**. Komande se kucaju u
**PowerShell**-u na Windows-u. Instalaciju alata (Docker, k3d, kubectl, helm)
pokriva [`SETUP.md`](SETUP.md) poglavlje 1 — ovde podrazumevamo da su već tu.

---

## 0. Pojmovi na 2 minuta (da razumeš šta radiš)

| Pojam | Šta je, prostim rečima |
|-------|------------------------|
| **Docker image** | „Slika" — zamrznut paket aplikacije sa svim što joj treba. Kao instalacioni fajl. |
| **Docker container** | Pokrenuta instanca slike — živ proces. Slika je recept, kontejner je skuvano jelo. |
| **Kubernetes (k8s)** | Sistem koji pokreće i održava kontejnere — sam ih restartuje, skalira, povezuje. |
| **k3d** | Alat koji pravi mali Kubernetes klaster **unutar Docker-a** na tvom računaru (za lokalni rad). |
| **node** | Jedan „računar" u klasteru. Tvoj klaster ima 3 (1 server + 2 agenta), svi su Docker kontejneri. |
| **pod** | Najmanja jedinica u k8s — omotač oko jednog (ili više) kontejnera. Aplikacija se vrti u podovima. |
| **namespace** | „Fascikla" unutar klastera koja grupiše resurse. Svaki korisnik ShopHub-a dobija svoj (`tenant-…`). |
| **kubectl** | Komandni alat kojim pričaš sa klasterom (`kubectl get pods` = „prikaži podove"). |
| **helm** | „Instaler" za Kubernetes — instalira gomilu resursa odjednom iz paketa zvanog **chart**. |
| **operator** | Program u klasteru koji automatski pravi/održava stvari. Tvoj Shop operator od jednog „Shop" zahteva napravi celu prodavnicu. |
| **CRD** | Custom Resource Definition — novi tip objekta koji operator razume (npr. `Shop`, `Wallet`, `DiscordChannel`). |
| **ArgoCD** | Alat koji čita git repo i sam dovodi klaster u to stanje (GitOps). |
| **ShopHub vs Shop** | ShopHub = platforma gde praviš prodavnice. Shop = jedna pojedinačna prodavnica koju operator deployuje. |

---

## 1. Čist prazan Docker (kako da obrišeš sve)

Radi ovo kad hoćeš da kreneš od nule (npr. na novom laptopu, ili da proveriš da je sve ponovljivo).

```powershell
# 1. obriši k3d klaster (uklanja 4 kontejnera klastera + njegove volumene)
k3d cluster delete local

# 2. obriši SVE Docker slike, zaustavljene kontejnere, mreže, volumene i build cache
docker system prune -a --volumes -f
```

Provera da je prazno (sve treba da bude 0):

```powershell
docker system df
k3d cluster list
```

> ⚠️ `docker system prune -a --volumes -f` briše **sve** Docker slike na računaru,
> ne samo ovog projekta. Sve se ponovo povuče/izgradi pri sledećem pokretanju.

---

## 2. Pokretanje iz nule

Detaljno je u [`SETUP.md`](SETUP.md) (poglavlja 2–6). Ukratko, iz `kube-state` foldera:

```powershell
# A. napravi klaster, mora da budemo u kube-state folderu u terminalu
k3d cluster create --config clusters/local/cluster.yaml

# B. tajne (Grafana admin lozinka u 3 namespace-a)
$GRAFANA_PASS = -join (1..32 | ForEach-Object { '{0:x}' -f (Get-Random -Maximum 16) })
foreach ($ns in 'monitoring','shop-operator-system','shophub') {
  kubectl create namespace $ns --dry-run=client -o yaml | kubectl apply -f -
  kubectl create secret generic grafana-admin -n $ns `
    --from-literal=admin-user=admin --from-literal=admin-password=$GRAFANA_PASS
}
Write-Host "ZAPAMTI Grafana lozinku: $GRAFANA_PASS"

# C. (opciono, za Discord) bot token
kubectl create namespace shophub --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic discord-bot-token -n shophub --from-literal=token=PASTE_BOT_TOKEN

# D. instaliraj sve — ArgoCD način (preporuka)
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-repo-server
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/repositories.yaml
kubectl apply -f argocd/root.yaml
```

Prati dok se sve ne podigne (prvi put 15–20 min zbog povlačenja slika):

```powershell
kubectl get applications -n argocd -w     # čekaj Synced/Healthy, pa Ctrl+C
kubectl get pods -A                        # sve treba Running
```

> Ovo je odlična prilika da otvoriš Docker Desktop → Images i **gledaš kako se
> slike pojavljuju jedna po jedna** — to su komponente koje se povlače.

---

## 3. Provera da je infrastruktura gore

```powershell
kubectl get pods -A
```

Treba da vidiš `Running` podove u: `cnpg-system`, `mongodb-operator`, `monitoring`
(prometheus, grafana, alertmanager, loki, tempo), `shop-operator-system`, `shophub`.

ShopHub UI: <http://shophub.localhost:8080> (koristi **Chrome/Edge**).

---

## 4. Testiranje SVAKE funkcionalnosti

Za svaku stavku: **šta** testiraš, **kako**, i **šta da očekuješ**. Štikliraj kako prolaziš.

### 4.1 Registracija i prijava (spec 1.1)
- **Kako:** otvori <http://shophub.localhost:8080> → registruj se (email + lozinka ≥8 karaktera).
- **Očekuj:** ulaziš u dashboard. Iza scene: napravljen je tvoj `tenant-…` namespace.
- **Provera u klasteru:** `kubectl get namespaces` → vidiš novi `tenant-…`.

### 4.2 Web3 prijava (spec 1.1 opciono)
- **Kako:** na login stranici izaberi „Sign in with wallet" → MetaMask potpiše poruku.
- **Očekuj:** prijava bez lozinke; adresa novčanika je tvoj identitet.

### 4.3 Kreiranje prodavnice (spec 1.2)
- **Kako:** „New shop" → ime (npr. `odeca`), **availability** (`standard`=2 replike, `high`=3), **baza** (`postgres` ili `mongodb`), wallet adresa (ili dugme **Generate**), čekiraj **Discord** ako testiraš alarme.
- **Očekuj:** prodavnica se pojavi sa bedžom availability-ja. Operator je primio `Shop` CRD i kreće da pravi sve.
- **Provera:**
  ```powershell
  kubectl get shops -A -o custom-columns="NS:.metadata.namespace,NAME:.metadata.name,AVAIL:.spec.availability,DB:.spec.database,READY:.status.readyReplicas"
  ```

### 4.4 Operator je napravio celu prodavnicu (spec 3.1)
Zameni `<ns>` i `<ime>` svojom prodavnicom (`kubectl get shops -A`):
```powershell
kubectl get deployment,svc,ingress -n <ns>     # deployment (2 ili 3 replike), service, ingress
kubectl get pods -n <ns>                        # app podovi + baza (-1)
```
- **Očekuj:** `high` → 3 app poda, `standard` → 2. Baza (CNPG ili MongoDB) ima svoj pod.

### 4.5 Otvaranje prodavnice (storefront)
- **Kako:** klikni **Open** na kartici prodavnice (ili idi na `http://<ime>.localhost:8080`).
- **Očekuj:** vidiš storefront (prazan dok ne dodaš artikle).

### 4.6 Admin prodavnice + artikli (spec 2.1)
- **Kako:** u ShopHub-u klikni ikonicu **ključa** na kartici → kopiraj admin lozinku. Otvori `http://<ime>.localhost:8080/admin/login`, prijavi se, dodaj artikle (naziv, cena, količina).
- **Očekuj:** artikli se pojave u storefront-u. (Cene drži na ≤ par decimala.)

### 4.7 Kupovina kriptom (spec 2.4)
- **Priprema MetaMask-a:** mreža **Sepolia**, imaš Sepolia ETH (gas) i mintovan **TestUSDT** (`0x74b0ef…7ff6f`, 6 decimala) — vidi [`SETUP.md`](SETUP.md) poglavlje 8.
- **Kako:** kao kupac → pretraga → dodaj u korpu → **Pay with USDT** → potvrdi u MetaMask-u.
- **Očekuj:** status ide `pending` → `confirmed` (backend verifikuje uplatu na lancu). Stanje artikla se umanji.

### 4.8 Pregled porudžbina (spec 2.2)
- **Kako:** u admin panelu prodavnice otvori listu porudžbina.
- **Očekuj:** vidiš kreiranu porudžbinu sa iznosom i statusom.

### 4.9 Grafana — dashboard po prodavnici (spec 4.1)
- **Kako:**
  ```powershell
  kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
  ```
  Otvori <http://localhost:3000>, prijavi se kao `admin` / (lozinka iz koraka 2B). Nađi dashboard te prodavnice.
- **Očekuj panele:** ukupno HTTP zahteva (24h), uspešni (2xx/3xx), neuspešni (4xx/5xx), **404 sa endpoint-ima**, **jedinstveni posetioci**, GB saobraćaja, CPU/RAM/fajl-sistem/mreža.
- **Da nakupiš podatke:** par puta osveži storefront i pogodi nepostojeći **API** put (npr. `http://<ime>.localhost:8080/api/nema-ovoga`). Napomena: ne-API putevi (npr. `/nema-ovoga`) vraćaju `index.html` sa statusom 200 (SPA fallback), pa se NE broje kao 404 — za pravi 404 put mora počinjati sa `/api`, `/metrics` ili `/probe`.

### 4.10 Per-tenant Grafana izolacija (spec 4.1 opciono)
- **Kako:** u ShopHub dashboard-u klikni **Metrics** → dobiješ login za **svoju** Grafana organizaciju.
- **Očekuj:** prijavljen kao taj nalog vidiš **samo svoje** dashboarde, ne tuđe.

### 4.11 Discord alarmi (spec D10)
- **Preduslov:** prodavnica kreirana sa čekiranim Discord-om (korak 4.3) + `discord-bot-token` secret (korak 2C).
- **Provera da je kanal napravljen:** na Discord serveru se pojavi kanal sa imenom prodavnice.
- **Kako da okineš alarm:** alarm ima `for: 2m`, pa udeo grešaka mora da bude visok i da **traje neprekidno ~2 min** — kratka petlja (50 zahteva) NE okida. Pusti **trajnu** petlju koja gađa nepostojeći **API** put ~4 minuta:
  ```powershell
  $end = (Get-Date).AddMinutes(4)
  while ((Get-Date) -lt $end) { curl.exe -s -o NUL -H "Host: <ime>.localhost" http://localhost:8080/api/nema-ovoga }
  ```
- **Očekuj:** posle ~2-3 min (alarm ima `for: 2m`) stigne **notifikacija na Discord kanal** te prodavnice.
- **Provera pravila:** `kubectl get prometheusrule -A` i `kubectl get alertmanagerconfig -A`.

### 4.12 Metrike i alarmi celog klastera (spec 4.1)
- **Kako (dashboardi):** u Grafani otvori Kubernetes/Node dashboarde (dolaze sa kube-prometheus-stack-om) — npr. „Node Exporter / Nodes".
- **Očekuj:** CPU/RAM/FS/mreža po nodu. Cluster alarmi: `NodeHighMemory`, `NodeHighCPU` (definisani u operator chart-u, grupa `cluster.rules`).

**Šta je Prometheus:** baza vremenskih serija + alarm-engine. On **skuplja (scrape) metrike** sa svih komponenti, čuva ih i **evaluira pravila alarma**. Grafana ga samo crta — kad gledaš dashboard, Grafana iza scene pita Prometheus. Zato „nismo posebno palili Prometheus" tokom testa: koristili smo ga **indirektno preko Grafane** sve vreme.

**Provera da su cluster alarmi učitani (Prometheus UI):**
```powershell
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```
Otvori <http://localhost:9090/alerts> → nađi `NodeHighMemory` i `NodeHighCPU`. Stanje `Inactive` je normalno dok nod nije preopterećen; `Pending`/`Firing` kad pređu prag.

> Pragovi (operator chart, grupa `cluster.rules`): **RAM > 90%** i **CPU > 90%**, oba sa `for: 10s` (skraćeno sa 5m radi bržeg demoa). Per-shop alarmi (`ShopHighErrorRate`, `ShopHighLatency`) su takođe `for: 10s`.

**Da OKINEŠ `NodeHighCPU` (opciono, opterećuje laptop):** pusti par „CPU-burner" podova pa sačekaj ~1-2 min:
```powershell
1..3 | ForEach-Object { kubectl run cpu-burner-$_ --image=busybox --restart=Never -- sh -c "while true; do :; done" }
```
Prati na <http://localhost:9090/alerts> da `NodeHighCPU` pređe u `Firing`. **Obavezno počisti posle:**
```powershell
1..3 | ForEach-Object { kubectl delete pod cpu-burner-$_ }
```

### 4.13 Logovi i tracing (spec 4.1)
- **Logovi (Loki):** u Grafani → Explore → izvor `Loki` → upit `{app="<ime>"}` → vidiš logove prodavnice.
  - Braon upozorenje **„Failed to load log volume … parse error … unexpected IDENTIFIER"** je samo histogram na vrhu (Grafana↔Loki kozmetička začkoljica) — **logovi se svejedno učitavaju**, ignoriši ga.
- **Tracing (Tempo):** Explore → izvor `Tempo` → tab `Search` → `Run query` → vidiš trace-ove (backend ih šalje preko OTLP).
  - Lista je puna `GET /probe/liveness`, `/probe/readiness`, `/metrics` jer se ti gađaju svakih ~15s i brojčano dominiraju. Tvoji `/api/...` pozivi (kupovina, artikli) su tu, samo retki.
  - **Da izvučeš baš API trace:** podesi vreme tako da obuhvati kad si kupovao (npr. `Last 3 hours`), pa tab **`TraceQL`** i upit (sintaksa: vitičaste zagrade + atribut):
    ```
    { name =~ ".*orders.*" }
    ```
    (ili `{ name =~ ".*items.*" }`). Alternativa: u `Search Options` podigni `Limit` na ~200 pa skroluj.

### 4.14 Izmena i brisanje prodavnice (spec 1.2)
- **Izmena:** olovka na kartici → promeni availability (npr. standard→high) ili wallet → sačuvaj.
  - **Očekuj:** `kubectl get pods -n <ns>` pokaže da broj replika prati promenu (2→3).
- **Brisanje:** kanta na kartici → potvrdi.
  - **Očekuj:** ceo `tenant`-resurs (deployment, baza, ingress, dashboard, Discord kanal) nestaje (operator počisti preko owner-referenci i finalizera).

### 4.15 DevOps (spec 5.x) — na GitHub-u, ne u klasteru
- **PR pipeline:** otvori bilo koji PR → vidi da se pokreću `Tests`, `Lint`, `Docker Build`, `commitlint` i da blokiraju merge ako padnu.
- **Conventional commits + linear history:** u Settings → Rules.
- **CI/CD:** na merge u `main` rade `docker-publish` (slika sa SemVer tagom) i `helm-publish` (chart na OCI).
- **IaC (eliminacioni 5.3):** repoi `helm-charts` (tvoji chart-ovi) i `kube-state` (stanje klastera) + opcioni ArgoCD.

---

## 5. Kad završiš — ponovo čisto (opciono)

```powershell
k3d cluster delete local
docker system prune -a --volumes -f
```

Sledeći put je brže jer... zapravo nije, ako si obrisao i slike — opet se sve
povlači. Ako želiš **brzo** sledeće pokretanje, obriši **samo klaster**
(`k3d cluster delete local`) a **ostavi slike** (preskoči `docker system prune`) —
tada sledeći `k3d cluster create` + instalacija koriste već keširane slike.
