## 1. Čist prazan Docker 

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

Prati dok se sve ne podigne (prvi put **15–20 min** zbog povlačenja slika):

```powershell
kubectl get applications -n argocd -w     # čekaj Synced/Healthy, pa Ctrl+C
kubectl get pods -A                        # sve treba Running
```

---

## 3. Provera da je infrastruktura gore

```powershell
kubectl get pods -A
```
ShopHub UI: <http://shophub.localhost:8080> (koristi **Chrome/Edge**).

---

## 4. Testiranje funkcionalnosti

### 4.1 Registracija i prijava
- **Kako:** otvori <http://shophub.localhost:8080> → registruj se (email + lozinka ≥8 karaktera).
- **Provera u klasteru:** `kubectl get namespaces` → vidiš novi `tenant-…`.

### 4.2 Web3 prijava 
- **Kako:** na login stranici izaberi „Sign in with wallet" → MetaMask potpiše poruku.
- **Očekuj:** prijava bez lozinke; adresa novčanika je tvoj identitet.

### 4.3 Kreiranje prodavnice
- **Provera:**
  ```powershell
  kubectl get shops -A -o custom-columns="NS:.metadata.namespace,NAME:.metadata.name,AVAIL:.spec.availability,DB:.spec.database,READY:.status.readyReplicas"
  ```

### 4.4 Operator je napravio celu prodavnicu
`<ns>` je tenant namespace, `<ime>` ime prodavnice — obe vrednosti čitaš iz prve dve kolone
`kubectl get shops -A`. Pažnja: namespace je granica KORISNIKA, pa ako korisnik ima više
prodavnica, prva komanda ih pokazuje sve zajedno.
```powershell
kubectl get deployment,svc,ingress -n <ns>   # sve prodavnice tog korisnika (deployment 2/3 replike, service, ingress; + mongo "<ime>-svc" ako je mongodb)
kubectl get pods -n <ns> -l app=<ime>        # samo APP podovi jedne prodavnice
kubectl get pods -n <ns>                     # sve, uključujući DB podove: "<ime>-1" (postgres) / "<ime>-0" (mongo, 2/2)
```

### 4.5 Otvaranje prodavnice (storefront)
- **Kako:** klikni **Open** na kartici prodavnice (ili idi na `http://<ime>.localhost:8080`).

### 4.6 Admin prodavnice + artikli 
- **Kako:** u ShopHub-u klikni ikonicu **ključa** na kartici → kopiraj admin lozinku. Otvori `http://<ime>.localhost:8080/admin/login`, prijavi se, dodaj artikle (naziv, cena, količina).

### 4.7 Kupovina kriptom 
- **Kako:** kao kupac → pretraga → dodaj u korpu → **Pay with USDT** → potvrdi u MetaMask-u.

### 4.8 Pregled porudžbina 
- **Kako:** u admin panelu prodavnice otvori listu porudžbina.
- **Očekuj:** vidiš kreiranu porudžbinu sa iznosom i statusom.

### 4.9 Grafana — dashboard po prodavnici 
- **Kako:**
  ```powershell
  kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
  ```
  Otvori <http://localhost:3000>, prijavi se kao `admin` / (lozinka iz koraka 2B). 
 
### 4.10 Per-tenant Grafana izolacija
- **Kako:** u ShopHub dashboard-u klikni **Metrics** → dobiješ login za **svoju** Grafana organizaciju.
- **Očekuj:** prijavljen kao taj nalog vidiš **samo svoje** dashboarde, ne tuđe.

### 4.11 Discord alarmi (spec D10)
- **Provera da je kanal napravljen:** na Discord serveru se pojavi kanal sa imenom prodavnice.
- **Kako da okineš alarm:**:
  ```powershell
  $end = (Get-Date).AddSeconds(60)
  while ((Get-Date) -lt $end) { curl.exe -s -o NUL -H "Host: <ime>.localhost" http://localhost:8080/api/nema-ovoga }
  ```

**Test `ShopDown` (obaranje prodavnice na 0 replika):**
```powershell
kubectl scale shop <ime> -n <tenant-ns> --replicas=0   # → za ~1-2 min FIRING:1 u kanal prodavnice
kubectl scale shop <ime> -n <tenant-ns> --replicas=2   # → resolved za par minuta
```

### 4.12 Metrike i alarmi celog klastera

**Provera da su cluster alarmi učitani (Prometheus UI):**
```powershell
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```
Otvori <http://localhost:9090/alerts> → nađi `NodeHighMemory` i `NodeHighCPU`. Stanje `Inactive` je normalno dok nod nije preopterećen; `Pending`/`Firing` kad pređu prag.

**Da OKINEŠ `NodeHighCPU` 
```powershell
kubectl run cpu-burn --image=busybox --restart=Never -- sh -c 'for i in $(seq $(nproc)); do while :; do :; done & done; wait'
```
**Obavezno počisti posle:**
```powershell
kubectl delete pod cpu-burn
```

### 4.13 Logovi i tracing 
- **Logovi (Loki):** u Grafani → Explore → izvor `Loki` → upit `{app="<ime>"}` → vidiš logove prodavnice.
- **Tracing (Tempo):** Explore → izvor `Tempo` → tab `Search` → `Run query` → vidiš trace-ove (backend ih šalje preko OTLP).
  - **Da izvučeš baš API trace:** podesi vreme tako da obuhvati kad si kupovao (npr. `Last 3 hours`), pa tab **`TraceQL`** i upit (sintaksa: vitičaste zagrade + atribut):
    ```
    { name =~ ".*orders.*" }
    ```
    (ili `{ name =~ ".*items.*" }`).
  - **Šta gledaš:** svaki red u tabeli je JEDAN stvarni HTTP zahtev (trace); `Service` = ime
    prodavnice (operator injektuje `OTEL_SERVICE_NAME`). Otvoren trace pokazuje rutu, status,
    trajanje i vremensku liniju. Naši trace-ovi imaju **1 span** po zahtevu (instrumentiran je
    HTTP sloj, ne i upiti ka bazi) — zato nema "vodopada" pod-koraka; poenta demo-a je da lanac
    backend → OTLP → Tempo → Grafana radi end-to-end.

### 4.14 Izmena i brisanje prodavnice (spec 1.2)
- **Izmena:** olovka na kartici → promeni availability (npr. standard→high) ili wallet → sačuvaj.
  - **Očekuj:** `kubectl get pods -n <ns>` pokaže da broj replika prati promenu (2→3).
- **Brisanje:** kanta na kartici → potvrdi.
  - **Očekuj:** ceo `tenant`-resurs (deployment, baza, ingress, dashboard, Discord kanal) nestaje (operator počisti preko owner-referenci i finalizera).

## 5. Kad završiš — ponovo čisto (opciono)

```powershell
k3d cluster delete local

docker system prune -a --volumes -f
```

