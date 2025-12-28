# Kubernetes Deployment

Эта директория содержит Helm charts и ArgoCD конфигурацию для развертывания всех компонентов системы в Kubernetes.

## Структура

```
deploy/
├── helm/                    # Helm charts для всех сервисов
│   ├── api-gateway/
│   ├── order-service/
│   ├── notification-service/
│   ├── analytics-service/
│   ├── kafka/
│   └── postgres/
└── argocd/                  # ArgoCD Application манифесты
    ├── applications/
    │   ├── api-gateway-app.yaml
    │   ├── order-service-app.yaml
    │   ├── notification-service-app.yaml
    │   ├── analytics-service-app.yaml
    │   ├── postgres-app.yaml
    │   └── kafka-app.yaml
    └── app-of-apps.yaml
```

## Helm Charts

Каждый Helm chart содержит:
- `Chart.yaml` - метаданные чарта
- `values.yaml` - конфигурация по умолчанию
- `templates/` - Kubernetes манифесты:
  - `deployment.yaml` / `statefulset.yaml` - основной ресурс
  - `service.yaml` - Service для доступа
  - `configmap.yaml` - ConfigMap для конфигурации
  - `secret.yaml` - Secret для секретных данных
  - `hpa.yaml` - HorizontalPodAutoscaler
  - `serviceaccount.yaml` - ServiceAccount для RBAC
  - `ingress.yaml` - Ingress (опционально)

## Развертывание через Helm

### Установка всех компонентов

```bash
# Создать namespace
kubectl create namespace default

# Установить инфраструктуру
helm install postgres ./deploy/helm/postgres
helm install kafka ./deploy/helm/kafka

# Установить сервисы
helm install api-gateway ./deploy/helm/api-gateway
helm install order-service ./deploy/helm/order-service
helm install notification-service ./deploy/helm/notification-service
helm install analytics-service ./deploy/helm/analytics-service
```

### Обновление

```bash
helm upgrade api-gateway ./deploy/helm/api-gateway
```

### Удаление

```bash
helm uninstall api-gateway
```

## Развертывание через ArgoCD (GitOps)

### Предварительные требования

1. Установлен ArgoCD в кластере
2. Репозиторий доступен для ArgoCD

### Настройка ArgoCD

1. Добавить репозиторий в ArgoCD:
```bash
argocd repo add https://github.com/CSU-ITMO-2025-2/team-2-helm-agro.git \
  --name team-2-helm-agro \
  --type git
```

2. Применить app-of-apps:
```bash
kubectl apply -f deploy/argocd/app-of-apps.yaml
```

Или применить все Application манифесты:
```bash
kubectl apply -f deploy/argocd/applications/
```

### Проверка статуса

```bash
# Через ArgoCD CLI
argocd app list
argocd app get api-gateway

# Через kubectl
kubectl get applications -n argocd
```

## Конфигурация

### Изменение образов

В `values.yaml` каждого чарта:
```yaml
image:
  repository: ghcr.io/CSU-ITMO-2025-2/api-gateway
  tag: "latest"
```

### Изменение реплик

```yaml
replicaCount: 2
```

### Настройка HPA

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

### Изменение ресурсов

```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

## Важные замечания

1. **Образы**: Убедитесь, что образы доступны в registry (ghcr.io или другой)
2. **Secrets**: В production используйте Sealed Secrets или внешние системы управления секретами
3. **Storage**: Для Kafka и Postgres требуется PersistentVolume
4. **Namespace**: По умолчанию используется namespace `default`, можно изменить в values.yaml
5. **ArgoCD**: Все Application манифесты уже настроены на репозиторий `team-2-helm-agro`
6. **Образы**: Убедитесь, что образы собираются в репозитории `team-2` и доступны в registry

## Troubleshooting

### Проверка статуса подов

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Проверка Helm релизов

```bash
helm list
helm status api-gateway
helm get values api-gateway
```

### Проверка ArgoCD синхронизации

```bash
argocd app sync api-gateway
argocd app get api-gateway
```

