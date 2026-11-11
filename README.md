# Carsharing GitOps — Лабораторна робота №5 (Argo CD)

Репозиторій конфігурації GitOps для розгортання мікросервісів `carsharing` через Argo CD (патерн **App of Apps**).

## Структура репозиторію

```text
carsharing-gitops/
├── bootstrap/
│   └── root.yaml                  # Кореневий Application (App of Apps)
├── argocd-apps/
│   └── carsharing.yaml            # Дочірній Application для Helm-чарту carsharing
└── charts/
    └── carsharing/                # Helm-чарт мікросервісів
        ├── Chart.yaml
        ├── values.yaml
        ├── values-minikube.yaml   # Конфігурація для локального кластера Minikube
        ├── values-prod.yaml
        └── templates/
```

## Образи в Docker Hub
- `cnnblesss/carsharing-user-service:1.0.0` та `1.1.0`
- `cnnblesss/carsharing-fleet-service:1.0.0` та `1.1.0`
- `cnnblesss/carsharing-order-service:1.0.0` та `1.1.0`

## Секрети (поза Git)
Секрет `carsharing-secret` створений у namespace `carsharing` вручну (не зберігається у відкритому вигляді в Git):
```powershell
kubectl create namespace carsharing
kubectl -n carsharing create secret generic carsharing-secret `
  --from-literal=DB_PASSWORD=K8sSecureDevPassword2026! `
  --from-literal=JWT_SECRET_KEY=c2hhcmluZy1zZWNyZXQta2V5LWZvci10b2tlbnM=
```

## Шпаргалка для живого захисту (Пункт 7)

### 1. Доступ до веб-інтерфейсу Argo CD та сервісу
```powershell
# Термінал 1 — Веб-інтерфейс Argo CD (https://localhost:8080)
kubectl -n argocd port-forward svc/argocd-server 8080:443

# Термінал 2 — Проброс порту до user-service (http://localhost:8081/version)
kubectl -n carsharing port-forward svc/carsharing-carsharing-user 8081:8082
```

### 2. Перевірка поточного стану та версії
```powershell
kubectl -n argocd get applications
kubectl -n carsharing get pods
curl.exe http://localhost:8081/version
```

### 3. Демонстрація `selfHeal` (ручний дрейф)
```powershell
kubectl -n carsharing scale deployment carsharing-carsharing-user --replicas=3
kubectl -n carsharing get pods -w
# Argo CD автоматично поверне кількість реплік до 1
```

### 4. Демонстрація релізу / зміни тегу через Git
Змініть тег у `charts/carsharing/values-minikube.yaml` (наприклад, між `"1.0.0"` і `"1.1.0"`):
```powershell
git commit -am "Update image tag" && git push
argocd app get carsharing --refresh
curl.exe http://localhost:8081/version
```

### 5. Демонстрація `Synced` + `Degraded` та відкату (`git revert`)
Змініть тег у `charts/carsharing/values-minikube.yaml` на `"9.9.9"`:
```powershell
git commit -am "Set non-existent tag 9.9.9" && git push
argocd app get carsharing --refresh
kubectl -n carsharing get pods

# Відкат через git revert:
git revert HEAD --no-edit && git push
argocd app get carsharing --refresh
git log --oneline -n 6
argocd app history carsharing
```
