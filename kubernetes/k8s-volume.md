# Kubernetes Storage — To'liq Qo'llanma

> **Maqsad:** Volume, PV, PVC, ConfigMap, Secret — hammasini amalda tushunish  
> **Server:** Vultr (K3s) | **Namespace:** amaliyot  
> **Amaliyot sanasi:** April 2026

---

## Mundarija

1. [Nima uchun storage kerak?](#1-nima-uchun-storage-kerak)
2. [Volume turlari](#2-volume-turlari)
3. [emptyDir](#3-emptydir)
4. [ConfigMap](#4-configmap)
5. [Secret](#5-secret)
6. [hostPath + PV + PVC](#6-hostpath--pv--pvc)
7. [PV parametrlari tushuntirish](#7-pv-parametrlari-tushuntirish)
8. [Hammasi taqqoslab](#8-hammasi-taqqoslab)
9. [Foydali commandlar](#9-foydali-commandlar)

---

## 1. Nima uchun storage kerak?

Pod o'chsa — **ichidagi hamma ma'lumot yo'qoladi**.

```
Pod restart → nginx log yo'qoldi      ❌
Pod restart → database ma'lumot yo'q  ❌
Pod restart → upload fayllar yo'q     ❌
```

Storage bilan:

```
Pod restart → ma'lumot qoladi ✅
```

### Docker bilan taqqoslash

Docker Swarm da sen shunday qilasan:

```yaml
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

K8s da xuddi shu narsa — lekin **3 qismga bo'lingan:**

```
Docker named volume  →  K8s da:  PV + PVC + Pod
```

---

## 2. Volume turlari

K8s da ko'p volume turi bor. Amalda ishlatiladiganlar:

| Tur | O'chsa | Qachon ishlatiladi |
|-----|--------|-------------------|
| `emptyDir` | Yo'qoladi ❌ | 2 container ma'lumot ulashsin |
| `configMap` | Qoladi ✅ | Config fayllarni inject qilish |
| `secret` | Qoladi ✅ | Parol, token inject qilish |
| `hostPath` | Qoladi ✅ | Test/o'rganish uchun |
| `persistentVolumeClaim` | Qoladi ✅ | Production database, fayllar |
| `nfs` | Qoladi ✅ | Ko'p pod bitta storage ko'rsin |

Intervyuda bilish **shart emas** bo'lganlar:
```
fc (Fibre Channel)     → bank, telekom darajasi
iscsi                  → enterprise SAN storage
gcePersistentDisk      → deprecated, CSI ga ko'chdi
gitRepo                → disabled, xavfsizlik sababi
portworxVolume         → deprecated
```

---

## 3. emptyDir

Pod yaratilganda **bo'sh papka** yaratiladi. Pod o'chsa — papka ham o'chadi.

### Qachon ishlatiladi?

Bir pod ichidagi **2 ta container** ma'lumot ulashishi kerak bo'lganda:

```
Pod
├── writer container  → /logs/app.log ga yozadi
└── reader container  → /logs/app.log ni o'qiydi
        ↑
    ikkalasi bir xil emptyDir volumeni ko'radi
```

### YAML

```yaml
# emptydir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
  namespace: amaliyot
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}              # bo'sh papka
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do echo 'log yozildi' >> /logs/app.log; sleep 2; done"]
    volumeMounts:
    - mountPath: /logs        # writer /logs papkasini ko'radi
      name: shared-logs
  - name: reader
    image: busybox
    command: ["sh", "-c", "tail -f /logs/app.log"]
    volumeMounts:
    - mountPath: /logs        # reader ham xuddi shu papkani ko'radi
      name: shared-logs
```

### Amaliyot

```bash
kubectl apply -f emptydir-pod.yaml
kubectl get pods -n amaliyot

# reader container writer yozgan loglarni o'qiydi
kubectl logs emptydir-pod -c reader -n amaliyot
# log yozildi
# log yozildi
# log yozildi
```

### emptyDir ning asosiy xususiyati — pod o'chsa ma'lumot yo'qoladi

```bash
kubectl delete pod emptydir-pod -n amaliyot
kubectl apply -f emptydir-pod.yaml

# Log soni 0 dan boshlanadi — eski loglar yo'qoldi ❌
kubectl logs emptydir-pod -c reader -n amaliyot
```

### Docker ekvivalenti

```
Docker tmpfs  ≈  K8s emptyDir
```

---

## 4. ConfigMap

Config ma'lumotlarini (`.env`, `nginx.conf`, sozlamalar) pod ichiga **fayl yoki env variable** sifatida berish.

### Nima uchun kerak?

K8s da pod **istalgan nodeda** ishga tushishi mumkin. 3 ta server bo'lsa:

```
Node 1 → /etc/nginx/nginx.conf  ✅ bor
Node 2 → /etc/nginx/nginx.conf  ❌ yo'q!
Node 3 → /etc/nginx/nginx.conf  ❌ yo'q!
```

ConfigMap bilan config **K8s ichida** saqlanadi — pod qaysi nodeda bo'lsa ham config keladi ✅

### Docker ekvivalenti

```
Docker → docker config create nginx-config nginx.conf
K8s   → ConfigMap
```

### YAML

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: amaliyot
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        location / {
          return 200 'Salom! Bu ConfigMap dan keldi!\n';
          add_header Content-Type text/plain;
        }
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
  namespace: amaliyot
spec:
  volumes:
  - name: config-volume
    configMap:
      name: nginx-config      # yuqoridagi ConfigMap
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - mountPath: /etc/nginx/nginx.conf   # pod ichida shu joyga
      name: config-volume
      subPath: nginx.conf                # ConfigMap dagi qaysi kalit
```

### Amaliyot

```bash
kubectl apply -f configmap.yaml
kubectl get pods -n amaliyot

# ConfigMap ishlayaptimi tekshir
kubectl exec -it configmap-pod -n amaliyot -- curl localhost
# Salom! Bu ConfigMap dan keldi!  ✅
```

Oddiy nginx emas — **bizning config** ishladi!

### Real hayotda qachon ishlatiladi?

```
nginx.conf          → ConfigMap
.env fayllar        → ConfigMap (maxfiy bo'lmasa)
database.yml        → ConfigMap
application.conf    → ConfigMap
```

---

## 5. Secret

ConfigMap ga o'xshash, lekin **shifrlangan**. Parol, token, API key uchun.

### ConfigMap vs Secret farqi

| | ConfigMap | Secret |
|--|-----------|--------|
| Shifrlangan | ❌ | ✅ base64 |
| Qachon | Config fayllar | Parol, token, API key |
| Docker ekvivalenti | docker config | docker secret |

### Secret turlari

| Tur | Qachon |
|-----|--------|
| `Opaque` | Oddiy parol, token — **eng ko'p ishlatiladigan** |
| `kubernetes.io/tls` | SSL sertifikat |
| `kubernetes.io/dockerconfigjson` | Docker registry parol |

### stringData vs data

```yaml
# stringData → oddiy matn yozasan, K8s o'zi base64 qiladi ✅ oson
stringData:
  DB_PASSWORD: "mypassword123"

# data → o'zing base64 qilib yozasan
data:
  DB_PASSWORD: bXlwYXNzd29yZDEyMw==    # echo -n "mypassword123" | base64
```

### YAML

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: amaliyot
type: Opaque
stringData:
  DB_PASSWORD: "mypassword123"
  DB_USER: "admin"
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
  namespace: amaliyot
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo DB_USER=$DB_USER && echo DB_PASSWORD=$DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret       # qaysi Secret dan
          key: DB_USER          # Secret ichidagi qaysi kalit
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD
```

### Amaliyot

```bash
kubectl apply -f secret.yaml
kubectl get pods -n amaliyot

# Secret env variable sifatida keldi
kubectl logs secret-pod -n amaliyot
# DB_USER=admin
# DB_PASSWORD=mypassword123  ✅

# K8s ichida base64 shifrlangan saqlanadi
kubectl get secret db-secret -n amaliyot -o yaml
# data:
#   DB_PASSWORD: bXlwYXNzd29yZDEyMw==   ← base64
```

### base64 qilish

```bash
echo -n "mypassword123" | base64
# bXlwYXNzd29yZDEyMw==

# Decode qilish
echo "bXlwYXNzd29yZDEyMw==" | base64 -d
# mypassword123
```

---

## 6. hostPath + PV + PVC

Production da eng ko'p ishlatiladigan storage usuli.

### Rollar

```
Admin     → PV yaratadi    ("serverda 1GB joy bor" deb e'lon qiladi)
Developer → PVC yaratadi   ("menga 500MB kerak" deb so'raydi)
K8s       → Bind qiladi    (mos PV ga ulab beradi)
Pod       → PVC ishlatadi
```

### Ketma-ketlik

```
1. Serverda papka yarat   (hostPath uchun)
2. PV yarat               (admin)
3. PVC yarat              (developer)
4. Pod yarat              (PVC ishlatadi)
```

### YAML

```yaml
# storage.yaml

# 1. PV — admin yaratadi
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/k8s-data          # serverda real papka
    type: DirectoryOrCreate      # yo'q bo'lsa yaratib olsin
---
# 2. PVC — developer yaratadi
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: amaliyot
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi             # 500MB so'raydi
---
# 3. Pod — PVC ishlatadi
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
  namespace: amaliyot
spec:
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc          # yuqoridagi PVC
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'Salom PV dan!' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - mountPath: /data           # pod ichida /data papkasi
      name: my-storage
```

### Amaliyot

```bash
# Serverda papka yaratish
sudo mkdir -p /mnt/k8s-data
sudo chmod 777 /mnt/k8s-data

kubectl apply -f storage.yaml

# Holat tekshirish
kubectl get pv
kubectl get pvc -n amaliyot
kubectl get pods -n amaliyot

# Pod ichida ma'lumot bor
kubectl exec -it storage-pod -n amaliyot -- cat /data/test.txt
# Salom PV dan!  ✅
```

### Asosiy test — pod o'chsa ma'lumot qoladi

```bash
# Pod o'chir
kubectl delete pod storage-pod -n amaliyot

# Serverda ma'lumot qolganmi?
sudo find /var/lib/rancher/k3s/storage -name "test.txt" -exec cat {} \;
# Salom PV dan!  ✅ qoldi!

# Yangi pod yaratib eski ma'lumotni o'qi
kubectl run storage-pod2 \
  --image=busybox \
  --restart=Never \
  -n amaliyot \
  -- sh -c "cat /data/test.txt"

# Salom PV dan!  ✅ yangi pod eski ma'lumotni o'qidi!
```

---

## 7. PV parametrlari tushuntirish

```bash
kubectl get pv

NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM
my-pv   1Gi        RWO            Retain           Available
auto    500Mi      RWO            Delete           Bound       amaliyot/my-pvc
```

### ACCESS MODES

| Qisqartma | To'liq nom | Ma'nosi | Qachon |
|-----------|------------|---------|--------|
| **RWO** | ReadWriteOnce | 1 node yozadi/o'qiydi | Database, postgres |
| **ROX** | ReadOnlyMany | Ko'p node faqat o'qiydi | Static fayllar |
| **RWX** | ReadWriteMany | Ko'p node yozadi/o'qiydi | Shared upload fayllar |
| **RWOP** | ReadWriteOncePod | Faqat 1 pod yozadi | RWO ning qattiqroq versiyasi |

```
Postgres, MongoDB  → RWO  (faqat 1 pod yozsin)
nginx static files → ROX  (hammasi o'qisin)
upload fayllar     → RWX  (NFS bilan, hammasi yozsin)
```

> RWX ishlatmoqchi bo'lsang **NFS** kerak — hostPath RWX ni to'liq qo'llab-quvvatlamaydi.

### RECLAIM POLICY

PVC o'chirilganda PV bilan nima bo'ladi?

| Policy | Ma'nosi |
|--------|---------|
| **Retain** | PV qoladi, ma'lumot saqlanadi — qo'lda tozalash kerak |
| **Delete** | PV ham, ma'lumot ham o'chiriladi |

```
my-pv → Retain  ← sen o'chirsang PV qoladi, ma'lumot xavfsiz
auto  → Delete  ← PVC o'chirsang avtomatik yo'qoladi
```

### STATUS

| Status | Ma'nosi |
|--------|---------|
| **Available** | Hech kimga ulanmagan, bo'sh |
| **Bound** | PVC ga ulangan, ishlatilmoqda |
| **Released** | PVC o'chirilgan, lekin PV hali bor |
| **Failed** | Xato yuz berdi |

### K3s avtomatik PV yaratishi

K3s da `local-path` StorageClass mavjud. PVC yaratganda mos PV topilmasa — **K3s o'zi avtomatik PV yaratadi:**

```
Sen PVC yaratding (500Mi so'radi)
    ↓
K3s my-pv ga ulamoqchi bo'ldi
    ↓
my-pv → StorageClass yo'q → mos kelmadi
    ↓
K3s local-path StorageClass orqali yangi PV avtomatik yaratdi ✅
    ↓
my-pvc → avtomatik PV ga Bound bo'ldi
```

---

## 8. Hammasi taqqoslab

| Tur | Pod o'chsa | Docker ekvivalenti | Production? | Qachon |
|-----|-----------|-------------------|-------------|--------|
| `emptyDir` | Yo'qoladi ❌ | tmpfs | ❌ | 2 container ulashsin |
| `configMap` | Qoladi ✅ | docker config | ✅ | Config fayllar |
| `secret` | Qoladi ✅ | docker secret | ✅ | Parol, token |
| `hostPath` | Qoladi ✅ | `-v /host:/container` | ⚠️ test | O'rganish |
| `PVC` | Qoladi ✅ | named volume | ✅ | Database, fayllar |

### Real loyihada qanday ishlatiladi?

```yaml
# joytop-backend misoli
postgres:
  volume: PVC (ReadWriteOnce)        # database

nginx:
  configMap: nginx.conf              # config

django:
  secret: DATABASE_URL, SECRET_KEY   # parollar
  configMap: .env                    # sozlamalar

media fayllar:
  PVC (ReadWriteMany + NFS)          # upload fayllar
```

---

## 9. Foydali Commandlar

```bash
# PV va PVC
kubectl get pv
kubectl get pvc -n <namespace>
kubectl describe pv <nom>
kubectl describe pvc <nom> -n <namespace>

# ConfigMap
kubectl get configmap -n <namespace>
kubectl describe configmap <nom> -n <namespace>
kubectl get configmap <nom> -n <namespace> -o yaml

# Secret
kubectl get secret -n <namespace>
kubectl describe secret <nom> -n <namespace>
kubectl get secret <nom> -n <namespace> -o yaml

# base64
echo -n "matn" | base64           # shifrlash
echo "c2hifr" | base64 -d         # decode

# Volume tekshirish
kubectl exec -it <pod> -n <namespace> -- df -h    # disk holati
kubectl exec -it <pod> -n <namespace> -- ls /data  # papka ichini ko'r
```

---

## Intervyu savollari

**emptyDir nima uchun kerak?**
> Bir pod ichidagi bir nechta container ma'lumot ulashishi uchun. Pod o'chsa ma'lumot yo'qoladi.

**ConfigMap va Secret farqi?**
> ConfigMap — oddiy config fayllar uchun. Secret — parol, token, API key uchun, base64 shifrlangan saqlanadi.

**PV va PVC farqi?**
> PV — admin tomonidan e'lon qilingan real disk joyi. PVC — developer tomonidan joy so'rash. K8s mos PV ga ulab beradi.

**Pod o'chsa ma'lumot yo'qolmasligi uchun nima qilish kerak?**
> PVC ishlatish kerak. emptyDir ishlatilsa pod o'chganda ma'lumot yo'qoladi.

**RWO va RWX farqi?**
> RWO — faqat 1 node yoza oladi (database uchun). RWX — ko'p node yoza oladi (shared fayllar uchun, NFS kerak).

---

*Amaliyot: K3s on Vultr VPS | April 2026*