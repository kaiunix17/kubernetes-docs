# Kubernetes (K3s) Amaliyot — To'liq Qo'llanma

> **Maqsad:** kubeadm, kubelet, kubectl, namespace, pod, deployment, service — hammasini amalda tushunish  
> **Server:** Vultr (1 vCPU / 2GB RAM, Ubuntu 22.04)  
> **Tool:** K3s (lightweight Kubernetes)

---

## Mundarija

1. [K3s nima va nima uchun?](#1-k3s-nima-va-nima-uchun)
2. [K3s o'rnatish](#2-k3s-urnatish)
3. [Namespace](#3-namespace)
4. [Pod](#4-pod)
5. [Deployment](#5-deployment)
6. [Service](#6-service)
7. [Umumiy arxitektura](#7-umumiy-arxitektura)
8. [Intervyu savollari](#8-intervyu-savollari)

---

## 1. K3s nima va nima uchun?

**K3s** — Rancher tomonidan ishlab chiqilgan yengil Kubernetes distributivi.

### Kubernetes o'rnatish usullari taqqoslash

| Usul | Qachon ishlatiladi | RAM talab |
|------|--------------------|-----------|
| **kubeadm** | Production, qo'lda cluster | 2GB+ |
| **K3s** | Lightweight, edge, o'rganish | 512MB+ |
| **k0s** | kubeadm'ga o'xshash, engil | 1GB+ |
| **minikube** | Faqat local laptop | 2GB+ |
| **kind** | Docker ichida, CI/CD test | 2GB+ |
| **EKS/GKE/AKS** | Cloud managed | — |

> K3s da ham xuddi kubeadm dagi kabi `kubectl`, namespace, deployment, service, ingress — hammasi bir xil ishlaydi.

### Kubernetes asosiy komponentlari

| Komponent | Vazifasi |
|-----------|----------|
| `kubelet` | Har bir nodeda ishlaydigan agent, podlarni boshqaradi |
| `kubeadm` | Clusterni bootstrap qiluvchi tool |
| `kubectl` | Cluster bilan gaplashish uchun CLI tool |
| `etcd` | Cluster ma'lumotlari saqlanadigan baza |
| `kube-apiserver` | Barcha so'rovlar shu orqali o'tadi |
| `kube-controller-manager` | Deployment, ReplicaSet boshqaradi |
| `kube-scheduler` | Podlarni qaysi nodega joylashtirishni hal qiladi |
| `containerd` | Container runtime (Docker Swarm'dagi Docker o'rniga) |

---

## 2. K3s O'rnatish

```bash
curl -sfL https://get.k3s.io | sh -
```

### Tekshirish

```bash
systemctl status k3s
kubectl get nodes
kubectl get namespaces
kubectl get pods -A
```

### Natija

```
NAME    STATUS   ROLES           AGE     VERSION
vultr   Ready    control-plane   3d21h   v1.34.6+k3s1
```

### Default namespacelар (K3s o'zi yaratadiganlari)

```
NAME              STATUS   AGE
default           Active   3d21h    ← sen hech narsa ko'rsatmasang shu yerga tushadi
kube-node-lease   Active   3d21h    ← node sog'ligini kuzatish
kube-public       Active   3d21h    ← hamma o'qiy oladigan ma'lumotlar
kube-system       Active   3d21h    ← k8s o'zining ichki servicelari
cert-manager      Active   2d2h     ← SSL sertifikat boshqarish
```

---

## 3. Namespace

### Namespace nima uchun kerak?

Bir serverda bir nechta loyiha yoki muhitni **izolyatsiya** qilish uchun.

**Misol:** Sende 4 loyiha bor:
```
joytop/        → joytop-backend podlari
carinpocket/   → carinpocket podlari
booking/       → booking podlari
mahsulottop/   → mahsulottop podlari
```

**Asosiy foydasi:**
- **Izolyatsiya** — bir loyiha boshqasiga tegmaydi
- **Resource limit** — har bir loyihaga CPU/RAM cheklash
- **Huquq ajratish** — developer faqat `dev` namespace ga kira olsin
- **Tozalik** — bir namespace o'chirsang ichidagi hamma narsa ketadi

### Namespace yaratish

```bash
# Usul 1
kubectl create namespace amaliyot

# Usul 2 (qisqa)
kubectl create ns tashkent

# Ko'rish
kubectl get namespaces
kubectl get ns
```

### Natija

```
NAME              STATUS   AGE
amaliyot          Active   52s
cert-manager      Active   2d2h
default           Active   3d21h
kube-node-lease   Active   3d21h
kube-public       Active   3d21h
kube-system       Active   3d21h
tashkent          Active   6s
```

### Namespace izolyatsiyasi amalda

```bash
kubectl get pods                    # faqat default namespace
kubectl get pods -n amaliyot        # faqat amaliyot namespace
kubectl get pods -A                 # barcha namespacelар
```

> `-n` flag ko'rsatmasang har doim `default` namespace da qidiradi!

---

## 4. Pod

**Pod** — Kubernetes'dagi eng kichik birlik. Bir yoki bir nechta container o'z ichida.

### Pod vs Deployment farqi

| | Pod | Deployment |
|--|-----|------------|
| O'lsa | Qaytmaydi ❌ | Avtomatik qayta yaratiladi ✅ |
| Scale | Yo'q | Bor |
| Production | Ishlatilmaydi | Ishlatiladi |

### Pod yaratish (YAML)

```yaml
# amaliyot.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: amaliyot
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f amaliyot.yaml
kubectl get pods -n amaliyot
```

### Pod ichiga kirish

```bash
kubectl exec -it nginx-pod -n amaliyot -- sh

# Ichida:
curl localhost      # nginx ishlayaptimi
hostname            # pod nomi chiqadi: nginx-pod
exit
```

> **Docker Swarm ekvivalenti:** `docker exec -it <container> sh`

### Pod loglarini ko'rish

```bash
kubectl logs nginx-pod -n amaliyot
```

### Pod haqida to'liq ma'lumot

```bash
kubectl describe pod nginx-pod -n amaliyot
```

`describe` chiqaradigan muhim narsalar:
```
IP:           10.42.0.15        ← cluster ichidagi IP
Node:         vultr/192.x.x.x   ← qaysi nodeda ishlayapti
Restart Count: 0                ← hech marta o'chmagan

Events:                         ← eng muhimi! Xato shu yerda ko'rinadi
  Scheduled → Pulling → Pulled → Created → Started
```

---

## 5. Deployment

**Deployment** — podlarni boshqaradi. O'lsa qayta yaratadi, scale qilish mumkin.

### Deployment yaratish

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: amaliyot
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployment -n amaliyot
kubectl get pods -n amaliyot
```

### Natija

```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   3/3     3            3           4m
```

Pod nomlar:
```
nginx-deploy-6f9664446b-dpkwk   ← random hash qo'shiladi
nginx-deploy-6f9664446b-jhb52
nginx-deploy-6f9664446b-vt8qs
```

### Self-healing (o'z-o'zini tiklash)

```bash
# Bitta podni o'chir
kubectl delete pod nginx-deploy-6f9664446b-dpkwk -n amaliyot

# Darhol tekshir — yangi pod avtomatik yaratildi!
kubectl get pods -n amaliyot
```

**Natija:** 4 soniyada yangi pod yaratildi ✅

### Scale qilish

```bash
# 3 dan 5 ga oshir
kubectl scale deployment nginx-deploy --replicas=5 -n amaliyot
kubectl get pods -n amaliyot

# 5 dan 1 ga kamayt
kubectl scale deployment nginx-deploy --replicas=1 -n amaliyot
kubectl get pods -n amaliyot
```

> **Docker Swarm ekvivalenti:** `docker service scale nginx=5`

---

## 6. Service

**Service** — podlarga tarmoq orqali kirish uchun. Pod IP lari o'zgarib turadi, Service esa doimiy.

### Service turlari

| Tur | Kirish | Qachon |
|-----|--------|--------|
| `ClusterIP` | Faqat cluster ichidan | Microservicelar orasida |
| `NodePort` | Tashqaridan (server IP + port) | Test, o'rganish |
| `LoadBalancer` | Cloud load balancer | Production cloud |

### NodePort Service yaratish

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: amaliyot
spec:
  selector:
    app: nginx          # bu label dagi podlarga yo'naltiradi
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080     # tashqaridan kirish porti (30000-32767)
  type: NodePort
```

```bash
kubectl apply -f service.yaml
kubectl get service -n amaliyot
```

### Natija

```
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.43.144.40   <none>        80:30080/TCP   5s
```

### Brauzerdan kirish

```
http://192.248.178.155:30080  ✅
```

### Trafik yo'li

```
Browser → 192.248.178.155:30080
    → Service (10.43.144.40:80)
        → Pod (10.42.0.x:80)
```

---

## 7. Umumiy Arxitektura

```
kubectl (CLI)
    ↓
kube-apiserver
    ↓
etcd (ma'lumotlar)
    ↓
kube-controller-manager
    ↓
kube-scheduler
    ↓
kubelet (nodeda)
    ↓
containerd (container runtime)
    ↓
Pod (nginx, nodejs, django...)
```

---

## 8. Intervyu Savollari

**Namespace nima uchun kerak?**
> Loyihalar va muhitlarni bir-biridan izolyatsiya qilish uchun. Masalan, `dev`, `staging`, `production` namespacelari.

**Pod va Deployment farqi?**
> Pod — bitta container instance, o'lsa qaytmaydi. Deployment — podlarni boshqaradi, self-healing va scaling ta'minlaydi.

**Service nima uchun kerak?**
> Pod IP lari o'zgarib turadi. Service — doimiy endpoint ta'minlaydi va trafik ni podlarga yo'naltiradi.

**kubectl get pods vs kubectl get pods -A?**
> `-A` — barcha namespacelardagi podlarni ko'rsatadi. Ko'rsatilmasa faqat `default` namespace.

**kubeadm, kubelet, kubectl farqi?**
> `kubeadm` — clusterni o'rnatadi. `kubelet` — har bir nodeda ishlab turadi, podlarni boshqaradi. `kubectl` — foydalanuvchi tomonidan cluster ni boshqarish CLI si.

---

## Foydali Commandlar

```bash
# Namespace
kubectl get ns
kubectl create ns <nom>
kubectl delete ns <nom>

# Pod
kubectl get pods -n <namespace>
kubectl describe pod <nom> -n <namespace>
kubectl logs <pod-nom> -n <namespace>
kubectl exec -it <pod-nom> -n <namespace> -- sh
kubectl delete pod <pod-nom> -n <namespace>

# Deployment
kubectl get deploy -n <namespace>
kubectl scale deployment <nom> --replicas=<son> -n <namespace>
kubectl describe deploy <nom> -n <namespace>

# Service
kubectl get svc -n <namespace>
kubectl describe svc <nom> -n <namespace>

# Hammasi
kubectl get all -n <namespace>
kubectl get pods -A
```

---

*Amaliyot: K3s on Vultr VPS | April 2026*