Kubernetes learning
Workload types
Pod - k8sdagi eng kichik birlik. Ichida bir yoki bir nechta container run qilishimiz mumkin bo’ladi.

Replicaset - bir nechta podlarni ishga tushirishimiz mumkin. Masalan har doim 3ta pod ishlab tursin desak buni replicaset bitta .yaml fileni 3ta qilib ishga tushiradi va doim uchalasini trafiklarni uchalasi uchunam ushlab turadi. Nimaga replicasetni alohida ishlata olmaymiz. Sababi unda rollback qilish mumkin emas, va update ham qila olmaymiz vazifasi faqat 2 3 ta replica yaratish

Deployment - Replicasetni boshqaradi. Bizga update va rollback qilish imkonini beradi. Deploymentni ichida replicaset joylashgan shu uchun deploymentni uzi orqali replicasetni boshqara olamiz. Deployment Replicasetga aytadi Replicaset esa podlarni yaratadi. Stateless applarni deployment orqali run qilamiz.

DaemonSet - Har bir nodeda bitta 1 ta pod ishlatiladi. Node qo’shilsa avtomatik pod yaratiladi. Node o’chsa pod ham o’chadi. Asosan log yigish uchun (fluentd | filebeat), monitoring uchun (Prometheus, Node Exporter), Network Plugin (CNI)

Statefulset - Har bir podni o’z nomi va storagesi buladi. Pod o’chsa aynan o’sha nom bilan qayta yaratiladi. Deploymentda podlar mypod-234zxcfglsa-112334 nomlari bilan yaratiladi yani random nom oladi. StatefulSet da esa mysql-0, mysql-1 — doim bir xil. Aynan nima maqsadda ishlatiladi:
1. database (mysql, postgresql, mongodb)
2. Message queue (Kafka, MongoDB)
3. Elasticsearch cluster

Savol: K8s ichidagi podlar qanday aloqa qiladi? Dockerda containerlar nomlari orqali bir biri bilan aloqa bula oladi k8sda qanday
Javob: k8sda pod nomlari bilan aloqa qilolmaydi. K8sda service ishlatiladi. Umuman olganda k8sda har bir podni o’zini ip addresi buladi dynamic ular har doim pod restart bulganda qaytadan beriladi. Service ishlatilganda quyidagicha buladi

[Pod A - NestJS]  →  http://postgres-service:5432  →  [Service]  →  [Pod B - PostgreSQL]

Serviceni nomi dns sifatida ishlaydi va shuning uchun ipsi uzgarmaydi. Pod qaytadan yaratilsa service uni topib beradi.

*P.S: .envda nestjsda quyidagicha yozilgan buladi:
.env
DB_HOST=postgres-service #service_name
DB_PORT=5432

Serviceni o’zimiz qo’lda yozamiz. Quyidagicha syntax buladi

———
apiVersion: v1
kind: Service
metadata:
name: postgres-service   # ← shu nom DNS bo'ladi
spec:
selector:
app: postgres           # ← shu label li podlarni topadi
ports:
- port: 5432
  targetPort: 5432
  ———

selector orqali bilib oladi nestjs postgresni qayerda ekanini

——

Service (postgres-service)
selector: app=postgres
│
↓ label mos kelgan podlarni topadi
[Pod]  labels: app=postgres  ✅
[Pod]  labels: app=nginx     ❌
[Pod]  labels: app=postgres  ✅

——

Shunaqa ishlaydi. Pod IP si o'zgarsayam Service label orqali yangi IP ni topadi. Biza faqat postgres-service deb murojaat qilamiz— qolganini Kubernetes hal qiladi.

——
NestJS Pod  →  "postgres-service qayerda?"
↓
Kubernetes DNS: "10.244.0.8 da"
↓
PostgreSQL Pod (10.244.0.8)
——

Statefull va Stateless farqi
Statefull bu agar container yoki pod o’chsa malumotlar yoqolsa malumotlarni uzida saqlab tursa application statefull deyiladi.
Stateless - malumotlar pod yoki container o’chib qolganda o’chib ketmasa malumotlarni uzida saqlab turmasa bunga stateless deyiladi/

Deployment strategy
Deploy qilishda rollbackda har xil strategiyalar bilan qilishimiz mumkin. Quyidagi turlar mavjud

Recreate
Hamma versiyalar o’chadi

—
v1, v1, v1 (o’chiriladi) -> v2, v2, v2 (ko’tariladi)
—

Bu usulni kamchiligi shundaki v1 lar uchirilganda va v2 ko’tarish davrida downtime buladi

Rolling Update (default)
Podlar bitta bitta yangilandi

—
v1 v1 v1
v2 v1 v1
v2 v2 v1
v2 v2 v2
—

— yaml
strategy:
type: RollingUpdate
rollingUpdate:
maxUnavailable: 1   # bir vaqtda max 1ta o'chishi mumkin
maxSurge: 1         # extra 1ta pod qo'shimcha yaratishi mumkin
—

Blue/Green Deployment
Parallel ishlab turadi yani quyidagicha buladi

—
Blue (v1) - Barcha trafic hozirda shu blue deploymentda bo’ladi
Green (v2) - Ishga tushib testdan o’tkanidan so’ng barcha trafic bluedan greenga o’tkaziladi
—

Rollback qilish farqlanadi yani Blue/Green da bu jarayon juda tez bo’ladi ammo Rolling update sekinroq yani bitta bitta buladi.

Canary (foiz)
Trafikni foizlarda bulib yangi versiyani sinab kuradi. Asosan katta kompaniyalarda buladi bu.

—
100ta foydalanuvchi bulsa
90ta -> v1 (versionda)
10ta -> v1

Muammo yo'q  bo’lsa → asta-sekin 50/50, keyin 0/100
Muammo bor bo’lsa → v2 ni o'chiramiz, hech kim bilmaydi
—

rollback commandRe —
# Oxirgi versiyaga qaytish
kubectl rollout undo deployment nginx-deployment

# Aniq versiyaga qaytish
kubectl rollout undo deployment nginx-deployment --to-revision=2

# Versiyalarni ko'rish
kubectl rollout history deployment nginx-deployment
—

Workload deployment
Deploymentda nimalar qilsa bo’ladi

Create
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

Rollback
# Tarixni ko’rish
kubectl rollout history deployment/nginx-deployment

# Oxirgi versiyaga qaytish
kubectl rollout undo deployment/nginx-deployment

# Aniq biror bir versiyaga qaytish
kubectl rollout undo deployment/nginx-deployment --to-revision=2

Scale
# Pod sonini oshirish yoki kamaytirish
kubectl scale deployment/nginx-deployment --replicas=5

Pause yoki Resume
Bir nechta o'zgartirish qilmoqchi bo’lsak, avval to'xtatib, hammasini kiritib, keyin ishga tushirish kk:

kubectl rollout pause deployment/nginx-deployment
# o'zgartirishlar qilamiz…
kubectl rollout resume deployment/nginx-deployment

Status checking
kubectl rollout status deployment/nginx-deployment

Cleanup (eski replicasetlarni tozalash)
Deployment automatic o’zi tozalab turadi eski replicalarni nechtasini saqlashni esa belgilab quyishimiz mumkin.

# yaml ichida
spec:
revisionHistoryLimit: 3  # faqat 3ta eski RS saqlanadi




























