# Kubernetes Infratuzilma Dokumentatsiyasi

> **Stack:** K3s · Helm · NGINX Ingress · Cert-Manager · External DNS · ArgoCD  
> **Server:** DigitalOcean | Ubuntu 24.04 | 2 vCPU / 4GB RAM  
> **Domain:** kyiv.uz (Cloudflare)  
> **IP:** 46.101.225.221

---

## Mundarija

1. [Umumiy Arxitektura](#1-umumiy-arxitektura)
2. [K3s — Kubernetes O'rnatish](#2-k3s--kubernetes-ornatish)
3. [Helm — Package Manager](#3-helm--package-manager)
4. [NGINX Ingress Controller](#4-nginx-ingress-controller)
5. [Cert-Manager — SSL Sertifikat](#5-cert-manager--ssl-sertifikat)
6. [External DNS — Avtomatik DNS](#6-external-dns--avtomatik-dns)
7. [ArgoCD — GitOps Platformasi](#7-argocd--gitops-platformasi)
8. [Firewall Sozlash](#8-firewall-sozlash)
9. [Natija va Tekshirish](#9-natija-va-tekshirish)
10. [Keyingi Qadamlar](#10-keyingi-qadamlar)

---

## 1. Umumiy Arxitektura

Quyida barcha komponentlarning qanday birgalikda ishlashi ko'rsatilgan:

```
  Internet (Foydalanuvchi)
         |
         v
  Cloudflare DNS (kyiv.uz)  <──  External DNS (avtomatik A-record yozadi)
         |
         v
  DigitalOcean Server: 46.101.225.221
         |
         v
  NGINX Ingress Controller  (port 80 / 443)
         |  SSL/TLS (Cert-Manager + Let's Encrypt)
         v
  ArgoCD Server  →  argocd.kyiv.uz
         |
         v
  GitHub Repo  <──  GitOps: YAML fayllar avtomatik deploy
```

**Server ma'lumotlari:**

| Parametr | Qiymat |
|---|---|
| Provider | DigitalOcean |
| OS | Ubuntu 24.04 LTS |
| CPU | 2 vCPU |
| RAM | 4 GB |
| Disk | 80 GB SSD |
| IP | 46.101.225.221 |
| Domain | kyiv.uz (Cloudflare) |

---

## 2. K3s — Kubernetes O'rnatish

### Nima bu?

K3s — Kubernetes'ning yengillashtirilgan versiyasi. To'liq Kubernetes (K8s) o'rnatish juda murakkab va kamida 3 ta server talab qiladi. K3s esa bitta buyruq bilan, bitta serverda o'rnatiladi. O'rganish uchun ideal variant. Funksional jihatdan to'liq Kubernetes bilan bir xil.

### Nimaga kerak?

K3s — bu barcha qolgan komponentlarning asosi. NGINX Ingress, Cert-Manager, ArgoCD — bularning hammasi K3s ustida ishlaydi.

### O'rnatish

```bash
curl -sfL https://get.k3s.io | sh -
```

### KUBECONFIG sozlash

K3s o'z kubeconfig faylini `/etc/rancher/k3s/k3s.yaml` da saqlaydi. Helm va kubectl bu faylni topishi uchun environment variable sozladik:

```bash
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
source ~/.bashrc
```

### Tekshirish

```bash
kubectl get nodes

# Natija:
# NAME   STATUS   ROLES           AGE   VERSION
# kai    Ready    control-plane   21s   v1.34.6+k3s1
```

---

## 3. Helm — Package Manager

### Nima bu?

Helm — Kubernetes uchun package manager. Xuddi Ubuntu'da `apt` yoki Mac'da `brew` nima bo'lsa, Kubernetes uchun Helm o'sha. Helm'siz har bir dasturni o'rnatish uchun 10-20 ta YAML fayl yozish kerak. Helm bilan esa bitta buyruq yetarli.

### Nimaga kerak?

NGINX Ingress, Cert-Manager, External DNS, ArgoCD — bularning barchasini Helm orqali o'rnatdik.

### O'rnatish

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Tekshirish

```bash
helm version
# version.BuildInfo{Version:"v3.20.2", ...}
```

### Asosiy buyruqlar

| Buyruq | Vazifasi |
|---|---|
| `helm repo add <nom> <url>` | Yangi chart repository qo'shish |
| `helm repo update` | Repositorylarni yangilash |
| `helm install <nom> <chart>` | Dastur o'rnatish |
| `helm upgrade <nom> <chart>` | Dasturni yangilash |
| `helm list -A` | Barcha namespace'lardagi dasturlar |
| `helm uninstall <nom>` | Dasturni o'chirish |

---

## 4. NGINX Ingress Controller

### Nima bu?

Ingress Controller — Kubernetes'da tashqi internetdan kelgan so'rovlarni to'g'ri Pod'larga yo'naltiruvchi "eshik qo'riqchisi". NGINX Ingress eng mashhur Ingress Controller hisoblanadi.

### Nimaga kerak?

Bizda bitta server va bitta IP bor (`46.101.225.221`). Lekin bu IP orqali ko'plab domenlar ishlashi kerak. Masalan `argocd.kyiv.uz`, `app.kyiv.uz` va boshqalar — barchasi bitta IP orqali. NGINX Ingress qaysi so'rov qaysi domenga tegishli ekanini ajratib, to'g'ri joyga yo'naltiradi.

### O'rnatish

```bash
# Repo qo'shish
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# O'rnatish
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### External IP sozlash

K3s'da LoadBalancer avtomatik IP bermaydi. Serverning real IP'sini qo'lda berdik:

```bash
kubectl patch svc ingress-nginx-controller \
  -n ingress-nginx \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/externalIPs","value":["46.101.225.222"]}]'
```

### Tekshirish

```bash
kubectl get svc -n ingress-nginx

# Natija:
# NAME                       TYPE           EXTERNAL-IP      PORT(S)
# ingress-nginx-controller   LoadBalancer   46.101.225.221   80:32702,443:31245
```

---

## 5. Cert-Manager — SSL Sertifikat

### Nima bu?

Cert-Manager — Kubernetes'da SSL/TLS sertifikatlarini avtomatik olish va yangilash uchun tool. Let's Encrypt bilan ishlaydi — bu bepul SSL sertifikat beruvchi xizmat. Cert-Manager'siz har 90 kunda qo'lda sertifikat yangilash kerak bo'lardi.

### Nimaga kerak?

`argocd.kyiv.uz` saytiga HTTPS bilan kirish uchun SSL sertifikat zarur. Cert-Manager bu sertifikatni avtomatik oladi, saqlaydi va 90 kundan keyin o'zi yangilaydi.

### DNS01 Metod qanday ishlaydi?

```
Cert-Manager → Cloudflare API'ga kiradi
             → _acme-challenge.kyiv.uz TXT record qo'shadi
             → Let's Encrypt DNS'ni tekshiradi
             → Tasdiqlangach sertifikat beradi
             → TXT record o'chiriladi
```

### O'rnatish

```bash
helm repo add cert-manager https://charts.jetstack.io
helm repo update

helm install cert-manager cert-manager/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

### Cloudflare Token Secret yaratish

Cert-Manager Cloudflare DNS'ni boshqarishi uchun API token kerak:

```bash
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=<CLOUDFLARE_TOKEN> \
  --namespace cert-manager
```

### ClusterIssuer yaratish

ClusterIssuer — Cert-Manager'ga qayerdan va qanday sertifikat olishni ko'rsatuvchi resurs:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cloudflare
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: shahri2403@gmail.com
    privateKeySecretRef:
      name: letsencrypt-cloudflare-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
```

```bash
kubectl apply -f clusterissuer.yaml
```

### Tekshirish

```bash
kubectl get clusterissuer

# Natija:
# NAME                     READY
# letsencrypt-cloudflare   True   <-- tayyor!
```

---

## 6. External DNS — Avtomatik DNS

### Nima bu?

External DNS — Kubernetes'dagi Ingress va Service resurslarini kuzatib, ularning domenlarini avtomatik ravishda DNS provayderiga (bizda Cloudflare) qo'shib va o'chirib turadigan tool.

### Nimaga kerak?

Har yangi domen qo'shganda Cloudflare panelga kirib qo'lda A-record yozish kerak bo'lardi. External DNS buni avtomatlashtiradi:
- Ingress yaratilsa → DNS record qo'shiladi
- Ingress o'chirilsa → DNS record ham o'chiriladi

### O'rnatish

```bash
# values fayl
cat <<EOF > external-dns-values.yaml
provider:
  name: cloudflare
env:
  - name: CF_API_TOKEN
    value: "<CLOUDFLARE_TOKEN>"
txtOwnerId: "k8s-kyiv-uz"
domainFilters:
  - kyiv.uz
EOF

# Repo va o'rnatish
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --values external-dns-values.yaml
```

### Ishlash natijasi

`argocd.kyiv.uz` Ingress yaratilganda External DNS avtomatik Cloudflare'ga A-record qo'shdi:

```
action=CREATE  record=argocd.kyiv.uz  type=A  →  46.101.225.221
```

---

## 7. ArgoCD — GitOps Platformasi

### Nima bu?

ArgoCD — GitOps asosidagi Continuous Deployment (CD) platformasi. GitHub'dagi YAML fayllarni doimiy kuzatib, Kubernetes cluster bilan sinxronlashtiradi.

**Qanday ishlaydi:**
```
Siz → git push → GitHub
                    ↑
              ArgoCD kuzatadi (har 3 daqiqada)
                    ↓
              Kubernetes'ga avtomatik deploy
```

### GitOps nima?

GitOps — "Git haqiqat manbai bo'lsin" degan metodologiya. Kubernetes'dagi barcha o'zgarishlar faqat Git orqali amalga oshiriladi. Kimdir qo'lda `kubectl` bilan o'zgartirsa, ArgoCD uni sezadi va Git'dagi holatga qaytaradi.

### ArgoCD Komponentlari

| Pod | Vazifasi |
|---|---|
| `argocd-server` | UI va API. Biz kiradigan veb-interfeys |
| `argocd-application-controller` | Kubernetes cluster'ni doimiy kuzatadi |
| `argocd-repo-server` | GitHub repo'ni o'qiydi, YAML tahlil qiladi |
| `argocd-dex-server` | Login va autentifikatsiyani boshqaradi |
| `argocd-redis` | Cache. Tez ishlashi uchun |
| `argocd-notifications` | Slack/email xabar yuboradi |

### O'rnatish

```bash
# values fayl
cat <<EOF > argocd-values.yaml
global:
  domain: argocd.kyiv.uz

configs:
  params:
    server.insecure: true

server:
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
      nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    hostname: argocd.kyiv.uz
    tls: true
EOF

# Repo va o'rnatish
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values argocd-values.yaml
```

### values.yaml tushuntirish

| Sozlama | Tushuntirish |
|---|---|
| `global.domain: argocd.kyiv.uz` | ArgoCD'ning asosiy domeni |
| `server.insecure: true` | ArgoCD HTTP'da ishlaydi, HTTPS'ni NGINX hal qiladi |
| `force-ssl-redirect: true` | HTTP so'rovni HTTPS'ga yo'naltir |
| `backend-protocol: HTTP` | NGINX → ArgoCD orasidagi protokol |
| `cert-manager.io/cluster-issuer` | Cloudflare DNS01 orqali sertifikat ol |

### Traffic oqimi

```
Foydalanuvchi (HTTPS)
        ↓
NGINX Ingress (TLS terminatsiya — sertifikatni hal qiladi)
        ↓  (HTTP)
ArgoCD Server (server.insecure: true)
```

### Kirish paroli

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Username: admin
# Password: (yuqoridagi buyruq natijasi)
```

### Tekshirish

```bash
kubectl get pods -n argocd

# Hammasi Running bo'lishi kerak:
# argocd-application-controller-0    1/1   Running
# argocd-server-xxx                  1/1   Running
# argocd-repo-server-xxx             1/1   Running
# argocd-redis-xxx                   1/1   Running
# argocd-dex-server-xxx              1/1   Running
```

---

## 8. Firewall Sozlash

### Nimaga kerak?

Ubuntu'da UFW (Uncomplicated Firewall) o'rnatilgan. 80 va 443 portlarini ochmasak, tashqaridan serverga kira olmaymiz.

### Buyruqlar

```bash
ufw allow 22      # SSH — BU BIRINCHI! Yo'qsa serverdan uzilasiz
ufw enable        # Firewallni yoqish
ufw allow 80      # HTTP
ufw allow 443     # HTTPS
```

> ⚠️ **Diqqat:** 22-portni (SSH) avval ochmasdan `ufw enable` qilmang! Aks holda serverga SSH orqali kira olmay qolasiz.

---

## 9. Natija va Tekshirish

### O'rnatilgan komponentlar

```bash
helm list -A

# NAMESPACE       NAME            CHART
# ingress-nginx   ingress-nginx   ingress-nginx-4.x.x
# cert-manager    cert-manager    cert-manager-v1.20.x
# external-dns    external-dns    external-dns-1.20.x
# argocd          argocd          argo-cd-9.x.x
```

### SSL Sertifikat

```bash
kubectl get certificate -n argocd

# NAME                READY
# argocd-server-tls   True    <-- sertifikat tayyor!
```

### DNS tekshirish

```bash
nslookup argocd.kyiv.uz 8.8.8.8

# Name:    argocd.kyiv.uz
# Address: 46.101.225.221
```

### ArgoCD UI

Brauzerda **https://argocd.kyiv.uz** manzilini oching.  
"Let's get stuff deployed!" sahifasi ko'rinsa — hammasi muvaffaqiyatli! ✅

---

## 10. Keyingi Qadamlar

Infratuzilma tayyor. Keyingi amaliyotlar:

- [ ] GitHub'da `gitops-repo` ochish
- [ ] ArgoCD'ga GitHub repo'ni ulash
- [ ] Birinchi `Application` yaratish (YAML → GitHub → ArgoCD → K8s)
- [ ] Avtomatik deploy'ni sinash: `git push` → K8s'da o'zgarish
- [ ] Helm chart yozish va ArgoCD orqali deploy qilish

> **Eslatma:** DNS to'liq tarqalgunga qadar (1-2 soat) quyidagini ishlating:
> ```bash
> sudo sh -c 'echo "46.101.225.221 argocd.kyiv.uz" >> /etc/hosts'
> ```
