# Лабораторная работа №6: Kustomize и Helm

Развертывание Sales Aggregator в Kubernetes с разделением инфраструктуры (PostgreSQL) и приложения.

## Структура

```
.
├── infrastructure/k8s/          # PostgreSQL
│   ├── helm/postgres-infra/
│   └── kustomization/
├── app/k8s/                     # Sales Aggregator
│   ├── helm/sales-aggregator/
│   └── kustomization/
└── README.md
```

## Требования

- Kubernetes кластер (Docker Desktop)
- kubectl, helm, kustomize

## Развертывание

### 1. Инфраструктура (PostgreSQL)

```bash
cd infrastructure

# Helm
helm upgrade --install sales-db ./k8s/helm/postgres-infra \
  --namespace sales-demo --create-namespace \
  -f ./k8s/helm/postgres-infra/values-dev.yaml

# Kustomize
kubectl apply -k k8s/kustomization/overlays/dev
```

### 2. Приложение

```bash
cd app

# Построить образ
docker build -t sales-aggregator:latest .

# Развернуть (Helm)
helm upgrade --install sales-app ./k8s/helm/sales-aggregator \
  --namespace sales-demo \
  -f ./k8s/helm/sales-aggregator/values-dev.yaml

# Развернуть (Kustomize)
kubectl apply -k k8s/kustomization/overlays/dev
```

### 3. Проверка

```bash
kubectl get pods -n sales-demo
kubectl logs -n sales-demo deployment/sales-aggregator -f

# Health check
kubectl port-forward -n sales-demo deployment/sales-aggregator 8000:8000
curl http://localhost:8000/health
```

## Контракт: Application ↔ Database

| | Dev | Prod |
|---|-----|------|
| Хост | `postgres-0.postgres` | `postgres-0.postgres.sales-prod.svc.cluster.local` |
| Порт | 5432 | 5432 |
| БД | `sales_db` | `sales_db` |
| Пользователь | `postgres` | `postgres` |
| Пароль | `postgres` | Из CI/CD |

## Окружения

**Dev** (sales-demo):
- Приложение: 1 реплика
- База: 1ГБ, 128МБ памяти

**Prod** (sales-prod):
- Приложение: 3 реплики
- База: 10ГБ, 512МБ памяти

## Удаление

```bash
helm uninstall sales-app -n sales-demo
helm uninstall sales-db -n sales-demo
kubectl delete pvc -n sales-demo -l app=postgres
```