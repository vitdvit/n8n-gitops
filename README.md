# n8n GitOps Infrastructure

GitOps-розгортання n8n у Kubernetes через FluxCD.

## Стек

| Компонент | Рішення |
|---|---|
| Застосунок | n8n (workflow automation) |
| База даних | PostgreSQL 16 |
| DB Operator | CloudNativePG |
| GitOps | FluxCD |
| Helm | Власний чарт (`apps/base/n8n`) |

## Структура репозиторію

```
├── clusters/aire/          # точки входу для Flux (infrastructure + apps kustomizations)
├── infrastructure/
│   ├── controllers/cnpg/   # CloudNativePG operator (HelmRelease)
│   └── configs/postgres/   # PostgreSQL Cluster CR для staging і production
├── apps/
│   ├── base/n8n/           # Helm чарт n8n
│   ├── staging/            # overlay для staging (1 репліка, мін. ресурси)
│   └── production/         # overlay для production (HPA 2-5, ресурси)
```

## Середовища

| Параметр | Staging | Production |
|---|---|---|
| Namespace | `staging` | `production` |
| Репліки | 1 (фіксовано) | 2-5 (HPA) |
| Ingress | `app.staging.local` | `app.local` |
| PostgreSQL | 1 інстанс | 2 інстанси |
| Ресурси | без лімітів | requests + limits |

## Bootstrap

```sh
flux bootstrap github \
  --owner=vitdvit \
  --repository=n8n-gitops \
  --branch=main \
  --path=./clusters/aire \
  --personal
```

## Перевірка роботи

```sh
# Статус HelmReleases
flux get helmreleases -A

# Статус Kustomizations
flux get kustomizations -A

# Поди в обох середовищах
kubectl get pods -A

# Ingress
kubectl get ingress -A
```

## Секрети (не в репо)

Перед bootstrap створити вручну:

```sh
# Staging
kubectl create secret generic postgres-staging-credentials \
  --from-literal=password=<STRONG_PASSWORD> -n staging

kubectl create secret generic n8n-encryption-key \
  --from-literal=key=<RANDOM_32_CHARS> -n staging

# Production
kubectl create secret generic postgres-production-credentials \
  --from-literal=password=<STRONG_PASSWORD> -n production

kubectl create secret generic n8n-encryption-key \
  --from-literal=key=<RANDOM_32_CHARS> -n production
```
