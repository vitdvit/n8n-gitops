# n8n GitOps Infrastructure

GitOps-розгортання n8n у Kubernetes через FluxCD.

## Стек

| Компонент | Рішення |
|---|---|
| Застосунок | n8n (workflow automation) |
| База даних | PostgreSQL 16 |
| DB Operator | CloudNativePG |
| GitOps | FluxCD v2.8.8 |
| Helm | Власний чарт (`apps/base/n8n`) |
| Кластер | Kubernetes v1.30, CRI-O, Flannel |

## Структура репозиторію

```
├── clusters/aire/          # точки входу для Flux (infrastructure + apps kustomizations)
├── infrastructure/
│   ├── controllers/cnpg/   # CloudNativePG operator (HelmRelease)
│   └── configs/postgres/   # PostgreSQL Cluster CR для staging і production
├── apps/
│   ├── base/n8n/           # Власний Helm чарт n8n
│   ├── staging/            # overlay: 1 репліка, мін. ресурси, app.staging.local
│   └── production/         # overlay: HPA 2-5, ресурси, app.local
```

## Середовища

| Параметр | Staging | Production |
|---|---|---|
| Namespace | `staging` | `production` |
| Репліки | 1 (фіксовано) | 2-5 (HPA) |
| Ingress | `app.staging.local` | `app.local` |
| PostgreSQL | 1 інстанс | 2 інстанси (HA) |
| Ресурси | без лімітів | requests + limits |

## Результати розгортання

### flux get helmreleases -A

```
NAMESPACE    NAME           REVISION  SUSPENDED  READY  MESSAGE
flux-system  cnpg           0.28.2    False      True   Helm install succeeded for release cnpg-system/cnpg-system-cnpg.v1 with chart cloudnative-pg@0.28.2
flux-system  n8n-production 0.1.0     False      True   Helm upgrade succeeded for release production/production-n8n-production.v2 with chart n8n@0.1.0
flux-system  n8n-staging    0.1.0     False      True   Helm install succeeded for release staging/staging-n8n-staging.v1 with chart n8n@0.1.0
```

### flux get kustomizations -A

```
NAMESPACE    NAME            REVISION           SUSPENDED  READY  MESSAGE
flux-system  apps-production main@sha1:0e5776fd False      True   Applied revision: main@sha1:0e5776fd
flux-system  apps-staging    main@sha1:0e5776fd False      True   Applied revision: main@sha1:0e5776fd
flux-system  flux-system     main@sha1:0e5776fd False      True   Applied revision: main@sha1:0e5776fd
flux-system  infrastructure  main@sha1:0e5776fd False      True   Applied revision: main@sha1:0e5776fd
```

### kubectl get pods -n staging

```
NAME                                       READY   STATUS    RESTARTS   AGE
postgres-staging-1                         1/1     Running   0          24m
staging-n8n-staging-n8n-5ffc478c45-5d75x   1/1     Running   0          66s
```

### kubectl get pods -n production

```
NAME                                             READY   STATUS    RESTARTS   AGE
postgres-production-1                            1/1     Running   0          24m
postgres-production-2                            1/1     Running   0          21m
production-n8n-production-n8n-6dd7f9c9bc-nbh2n   1/1     Running   0          20m
production-n8n-production-n8n-6dd7f9c9bc-qjsbx   1/1     Running   0          20m
```

### kubectl get ingress -A

```
NAMESPACE   NAME                           CLASS  HOSTS              ADDRESS  PORTS  AGE
production  production-n8n-production-n8n  nginx  app.local                   80    34m
staging     staging-n8n-staging-n8n        nginx  app.staging.local            80    34m
```

### kubectl get hpa -n production

```
NAME                            REFERENCE                                  TARGETS     MINPODS  MAXPODS  REPLICAS  AGE
production-n8n-production-n8n   Deployment/production-n8n-production-n8n   cpu: 2%/70%  2       5        2         34m
```

## Self-Healing

Flux автоматично відновлює видалені ресурси. Перевірка:

```sh
# Видаляємо HelmRelease і спостерігаємо відновлення
kubectl delete deployment staging-n8n-staging-n8n -n staging

# Форсуємо reconcile
flux reconcile helmrelease n8n-staging -n flux-system

# Deployment відновлено автоматично
kubectl get pods -n staging
```

## Bootstrap

```sh
export GITHUB_TOKEN=<your-pat-token>

flux bootstrap github \
  --owner=vitdvit \
  --repository=n8n-gitops \
  --branch=main \
  --path=./clusters/aire \
  --personal \
  --token-auth
```

## Секрети (не в репо, створити вручну перед bootstrap)

```sh
# Staging
kubectl create namespace staging
kubectl create secret generic postgres-staging-credentials \
  --from-literal=username=n8n \
  --from-literal=password=<STRONG_PASSWORD> -n staging
kubectl create secret generic n8n-encryption-key \
  --from-literal=key=<RANDOM_32_CHARS> -n staging

# Production
kubectl create namespace production
kubectl create secret generic postgres-production-credentials \
  --from-literal=username=n8n \
  --from-literal=password=<STRONG_PASSWORD> -n production
kubectl create secret generic n8n-encryption-key \
  --from-literal=key=<RANDOM_32_CHARS> -n production
```
