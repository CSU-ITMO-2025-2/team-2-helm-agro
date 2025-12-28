# Автоматический деплой через ArgoCD

## Как это работает

1. **Push изменений в `team-2`** → CI/CD автоматически:
   - Собирает Docker образ
   - Публикует в `ghcr.io/CSU-ITMO-2025-2/<service-name>:latest`
   - Обновляет `image.tag` в Helm values в репозитории `team-2-helm-agro`

2. **ArgoCD автоматически**:
   - Обнаруживает изменения в Git репозитории `team-2-helm-agro`
   - Синхронизирует изменения в Kubernetes кластер
   - Обновляет поды с новыми образами

## Настройка

### 1. Настройка прав доступа для GitHub Actions

Для того, чтобы CI/CD мог обновлять репозиторий `team-2-helm-agro`, нужно настроить права доступа:

#### Вариант A: Использовать GITHUB_TOKEN (рекомендуется для одной организации)

Если оба репозитория в одной организации, `GITHUB_TOKEN` должен работать. Убедитесь, что:
- Оба репозитория (`team-2` и `team-2-helm-agro`) находятся в организации `CSU-ITMO-2025-2`
- В настройках организации разрешена запись через GITHUB_TOKEN

#### Вариант B: Создать Personal Access Token (PAT)

1. Создайте Personal Access Token (PAT) в GitHub:
   - Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Создайте токен с правами: `repo` (полный доступ к репозиториям)
   - Скопируйте токен

2. Добавьте токен как Secret в репозиторий `team-2`:
   - Settings → Secrets and variables → Actions
   - Создайте новый secret: `HELM_REPO_TOKEN`
   - Вставьте ваш PAT

### 2. Проверка настроек ArgoCD

Убедитесь, что ArgoCD Application настроены на автоматическую синхронизацию:

```yaml
syncPolicy:
  automated:
    prune: true      # Автоматически удалять ресурсы, которых нет в Git
    selfHeal: true   # Автоматически исправлять расхождения
```

Это уже настроено в ваших Application манифестах.

### 3. Проверка подключения ArgoCD к репозиторию

Убедитесь, что ArgoCD подключен к репозиторию `team-2-helm-agro`:

```bash
# Через ArgoCD CLI
argocd repo list

# Или через UI
# Settings → Repositories → Добавить репозиторий
# URL: https://github.com/CSU-ITMO-2025-2/team-2-helm-agro
```

## Тестирование

1. Внесите изменения в код любого сервиса в `team-2`
2. Закоммитьте и запушьте в `main`:
   ```bash
   git add services/api-gateway/app/main.py
   git commit -m "test: update api-gateway"
   git push origin main
   ```

3. Проверьте CI/CD:
   - Перейдите в Actions в репозитории `team-2`
   - Убедитесь, что сборка прошла успешно
   - Проверьте, что образ опубликован

4. Проверьте обновление Helm values:
   - Перейдите в репозиторий `team-2-helm-agro`
   - Проверьте последний коммит - должен быть автоматический коммит от `github-actions[bot]`
   - Проверьте, что `deploy/helm/<service>/values.yaml` обновлен с новым тегом

5. Проверьте ArgoCD:
   ```bash
   # Проверить статус приложения
   argocd app get api-gateway
   
   # Или через UI
   # Applications → api-gateway → Sync Status
   ```

## Troubleshooting

### CI/CD не может обновить Helm repo

**Проблема**: Ошибка доступа при попытке записать в `team-2-helm-agro`

**Решение**:
1. Проверьте, что `HELM_REPO_TOKEN` настроен (если используете PAT)
2. Убедитесь, что токен имеет права `repo`
3. Проверьте, что оба репозитория в одной организации

### ArgoCD не синхронизирует изменения

**Проблема**: ArgoCD не видит изменения в Git

**Решение**:
1. Проверьте подключение репозитория: `argocd repo list`
2. Проверьте, что ArgoCD имеет доступ к репозиторию
3. Принудительно синхронизируйте: `argocd app sync api-gateway`
4. Проверьте логи ArgoCD: `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller`

### Образы не обновляются в кластере

**Проблема**: Поды используют старый образ

**Решение**:
1. Проверьте, что `imagePullPolicy: IfNotPresent` или `Always` в Deployment
2. Принудительно пересоздайте поды: `kubectl rollout restart deployment/api-gateway`
3. Проверьте, что новый образ доступен: `docker pull ghcr.io/CSU-ITMO-2025-2/api-gateway:latest`

## Дополнительные настройки

### Использование конкретных тегов вместо latest

Если хотите использовать SHA коммита как тег образа, измените в `build-push.yml`:

```yaml
IMAGE_TAG="${{ github.sha }}"
```

Или используйте короткий SHA:
```yaml
IMAGE_TAG=$(echo "${{ github.sha }}" | cut -c1-7)
```

### Отключение автоматического деплоя для определенных сервисов

Добавьте условие в шаг обновления Helm values:

```yaml
if: github.event_name != 'pull_request' && github.ref == 'refs/heads/main' && matrix.service != 'analytics-service'
```

## Полезные команды

```bash
# Проверить статус всех ArgoCD приложений
argocd app list

# Синхронизировать все приложения
argocd app sync --all

# Проверить историю синхронизации
argocd app history api-gateway

# Посмотреть diff между Git и кластером
argocd app diff api-gateway
```

