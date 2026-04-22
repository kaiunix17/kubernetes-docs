# Kubernetes Services: DevOps Professional Guide

Ushbu qo'llanma Kubernetes dagi Service turlari, ularning real hayotiy misollari va production muhitida qo'llanilishi haqida batafsil ma'lumot beradi.

## 1. Kubernetes Service Turlari

| Service Turi | Doirasi (Scope) | Asosiy Vazifasi | Prodda Qo'llanilishi |
| :--- | :--- | :--- | :--- |
| **ClusterIP** | Ichki (Internal) | Podlar o'rtasidagi muloqot | Eng ko'p ishlatiladigan tur (DB, Backend) |
| **NodePort** | Tashqi (External) | Har bir nodeda ma'lum bir portni ochish | Debugging va ichki toolar uchun |
| **LoadBalancer** | Tashqi (External) | Cloud Provider orqali IP olish | Public API va Frontend uchun |
| **ExternalName** | Tashqi DNS | Klaster ichidan tashqi DNS ga yo'naltirish | Tashqi DB yoki API larni ulashda |

---

## 2. Chuqur tahlil va Real Misollar

### ClusterIP (Default)
Bu servis turi podlarga klaster ichidagi doimiy IP manzilini beradi. 
* **Real Misol:** Sizning `python-backend` podingiz `postgres-db` podiga ulanishi kerak. Pod o'lib qayta tug'ilsa IP o'zgaradi, lekin Service orqali u doim `postgres-service:5432` manzili orqali ulanaveradi.

### NodePort
Har bir nodeda (serverda) 30000-32767 oralig'idagi portni band qiladi.
* **Vazifasi:** User to'g'ridan-to'g'ri `Server_IP:Port` orqali kira oladi. 
* **DevOps maslahati:** Prodda NodePortni ochiq qoldirish xavfli. Agar ishlatsangiz, faqat VPN orqali yoki Firewall (IP whitelist) bilan ishlating.

### LoadBalancer va Cloud API (Vultr/AWS/GCP)
Bu eng qiziqarli qism. Siz `type: LoadBalancer` yozganingizda quyidagilar sodir bo'ladi:
1. **Cloud Controller Manager (CCM)** sizning manifestni ko'radi.
2. CCM Cloud provider API (masalan, Vultr API) ga "Menga yangi IP ber" deb so'rov yuboradi.
3. Cloud provider tashqi IP (External IP) ajratadi va uni klasteringizdagi nodelarga yo'naltiradi.

---

## 3. Production Pattern: Ingress + ClusterIP

Productionda har bir servis uchun alohida LoadBalancer sotib olish qimmat (har bir IP pul turadi). Shuning uchun **Ingress Controller** ishlatiladi.

**Ishlash sxemasi:**
1. Bitta **LoadBalancer** servisi yaratiladi (bu Ingress Controller uchun).
2. Qolgan barcha servislar (Frontend, Backend, Minio) **ClusterIP** bo'lib qoladi.
3. Ingress qoidalar (Rules) orqali trafik yo'naltiriladi:
   - `domain.uz/` -> Frontend Service
   - `domain.uz/api` -> Backend Service

---

## 4. DevOps Arxitektura Sirlari

* **Kube-Proxy:** Har bir nodeda ishlaydigan bu komponent trafikni pod qaysi nodeda turganidan qat'i nazar, to'g'ri manzilga yetkazib beradi (Iptables/IPVS orqali).
* **Quorum (Kvorum):** Prod muhitida doim 3 tadan kam bo'lmagan node bo'lishi shart. Bu "Split Brain" muammosini oldini oladi.
* **Workload turlari:** - **Deployment:** Stateless (Frontend/Backend) uchun.
    - **StatefulSet:** Database va Storage (Minio/Postgres) uchun.
    - **DaemonSet:** Monitoring va Log yig'uvchilar uchun.

---
*Tayyorladi: Gemini (DevOps Assistant)*
