# Kubernetes ConfigMap, Secret ve Kaynak Yönetimi

## Genel Bakış

Bu projede Kubernetes yapılandırma yönetimi ele alınmıştır. nginx web sunucusu üzerinde ConfigMap, Secret ve ResourceQuota kullanılarak ortam değişkenleri, gizli kimlik bilgileri ve kaynak limitleri yönetilmiştir.

---

## Kullanılan Teknolojiler

- Kubernetes (Docker Desktop)
- nginx:1.21
- kubectl

---

## Proje Yapısı
hw_week_09/
├── configmap-dev.yaml       # Development ortamı ConfigMap
├── configmap-prod.yaml      # Production ortamı ConfigMap
├── secret-db.yaml           # Veritabanı kimlik bilgileri Secret
├── deployment.yaml          # nginx Deployment (ConfigMap + Secret + Resource Limits)
├── resourcequota.yaml       # Namespace kaynak kotası
├── service.yaml             # nginx'i expose eden ClusterIP Service
└── commands.txt             # Kullanılan kubectl komutları

---

## Bölüm 1: ConfigMap'ler

### Development ConfigMap (`configmap-dev.yaml`)

`web-config-dev` adında bir ConfigMap oluşturulmuştur.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config-dev
  namespace: default
data:
  ENVIRONMENT: "development"
  LOG_LEVEL: "DEBUG"
  MAX_CONNECTIONS: "100"
```
```bash
kubectl apply -f configmap-dev.yaml
kubectl describe configmap web-config-dev
```

### Production ConfigMap (`configmap-prod.yaml`)

`web-config-prod` adında bir ConfigMap oluşturulmuştur.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config-prod
  namespace: default
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "1000"
```
```bash
kubectl apply -f configmap-prod.yaml
kubectl describe configmap web-config-prod
```

### Deployment (`deployment.yaml`)

`web-server` adında 2 replica'lı bir Deployment oluşturulmuştur. `web-config-dev` ConfigMap'i `envFrom` ile tüm key-value çiftleri ortam değişkeni olarak Pod'a aktarılmıştır.

---

## Bölüm 2: Secret'lar

### Database Secret (`secret-db.yaml`)

`db-credentials` adında bir Secret oluşturulmuştur. Değerler base64 ile kodlanmıştır.
```bash
echo -n "mysql.example.com" | base64   # bXlzcWwuZXhhbXBsZS5jb20=
echo -n "webuser" | base64             # d2VidXNlcg==
echo -n "SecretPass123" | base64       # U2VjcmV0UGFzczEyMw==
```

Secret, Deployment'a `secretRef` ile eklenmiştir. `kubectl describe secret` komutunda değerler gizlenir, yalnızca byte sayısı görünür:
DB_HOST:      17 bytes
DB_PASSWORD:  13 bytes
DB_USER:      7 bytes

---

## Bölüm 3: Kaynak Yönetimi

### Resource Limits

Deployment'a aşağıdaki kaynak limitleri eklenmiştir:

| Kaynak | Request | Limit |
|--------|---------|-------|
| CPU    | 100m    | 200m  |
| Memory | 64Mi    | 128Mi |

### ResourceQuota (`resourcequota.yaml`)

`namespace-limits` adında bir ResourceQuota oluşturulmuştur:

| Kaynak          | Limit |
|-----------------|-------|
| requests.cpu    | 1000m |
| requests.memory | 2Gi   |
| pods            | 10    |

---

## Bölüm 4: Test ve Doğrulama

### ConfigMap değerlerinin doğrulanması
```bash
kubectl exec -it deployment/web-server -- env | grep -E "ENVIRONMENT|LOG_LEVEL|MAX_CONNECTIONS"
```

Çıktı:
MAX_CONNECTIONS=100
ENVIRONMENT=development
LOG_LEVEL=DEBUG

### Secret değerlerinin doğrulanması
```bash
kubectl exec -it deployment/web-server -- env | grep -E "DB_HOST|DB_USER|DB_PASSWORD"
```

Çıktı:
DB_HOST=mysql.example.com
DB_PASSWORD=SecretPass123
DB_USER=webuser

### Resource limitlerinin doğrulanması
```bash
kubectl describe pods | grep -A 4 "Limits:"
```

Çıktı:
Limits:
cpu:     200m
memory:  128Mi
Requests:
cpu:     100m

### Servisin test edilmesi
```bash
kubectl port-forward svc/web-service 8080:80
curl http://localhost:8080
```

nginx "Welcome to nginx!" sayfası başarıyla döndürülmüştür.

---

## Kaynakların Durumu
```bash
kubectl get configmaps
kubectl get secrets
kubectl get pods
kubectl get service web-service
kubectl describe resourcequota namespace-limits
```