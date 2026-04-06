# GitHub Actions — CI/CD для AgenticIT

## Что делают воркфлоу

| Файл | Триггер | Что делает |
|------|---------|------------|
| `ci.yml` | Push в любую ветку (кроме main) + PR → main | Build + тесты backend (dotnet), type-check + build frontend |
| `deploy.yml` | Push в main + ручной запуск | Строит Docker образы, пушит в ACR, деплоит helm chart в AKS |

Типичный флоу:
```
feature branch  →  CI (build/test)  →  PR → main  →  CI (build/test)  →  Deploy
```

---

## 1. Подготовка Azure

### 1.1 Service Principal

```bash
# Создать SP с правами на подписку (или ограничить до RG)
az ad sp create-for-rbac \
  --name "sp-agenticit-github" \
  --role Contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP> \
  --sdk-auth
```

Команда выведет JSON — это значение секрета `AZURE_CREDENTIALS`:
```json
{
  "clientId": "...",
  "clientSecret": "...",
  "subscriptionId": "...",
  "tenantId": "..."
}
```

### 1.2 Права SP на ACR и AKS

```bash
# Получить object ID SP
SP_ID=$(az ad sp show --id <clientId> --query id -o tsv)

# Права на push в ACR
az role assignment create \
  --assignee-object-id $SP_ID \
  --role AcrPush \
  --scope $(az acr show --name agenticitacr --query id -o tsv)

# Права на деплой в AKS
az role assignment create \
  --assignee-object-id $SP_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $(az aks show --resource-group <RG> --name <AKS_NAME> --query id -o tsv)
```

### 1.3 Привязать ACR к AKS (чтобы кластер мог тянуть образы)

```bash
az aks update \
  --resource-group <RESOURCE_GROUP> \
  --name <AKS_CLUSTER_NAME> \
  --attach-acr agenticitacr
```

После этого `global.containerRegistry.dockerConfig` в values.yaml оставить пустым — AKS сам аутентифицируется в ACR.

---

## 2. GitHub: Environment и переменные

### 2.1 Создать Environment

`Settings → Environments → New environment` → назвать **`production`**.

Опционально: добавить Required reviewers (защита от случайного деплоя) и задать Deployment branches: `main`.

### 2.2 Variables (не секретные)

`Settings → Environments → production → Environment variables`:

| Имя | Пример | Описание |
|-----|--------|----------|
| `ACR_NAME` | `agenticitacr` | Имя ACR (без `.azurecr.io`) |
| `AKS_RESOURCE_GROUP` | `rg-agenticit` | Resource group кластера |
| `AKS_CLUSTER_NAME` | `aks-agenticit` | Имя AKS кластера |
| `K8S_NAMESPACE` | `agenticit-ns` | Kubernetes namespace |
| `HELM_RELEASE_NAME` | `prod` | Префикс ресурсов в helm (`prod-agenticit-backend` и т.д.) |
| `APP_DOMAIN` | `app.agenticit.com` | Домен приложения |
| `POSTGRES_INTERNAL` | `True` | `True` = internal StatefulSet, `False` = Azure PostgreSQL |
| `POSTGRES_HOST` | `` | Хост внешней БД (если `POSTGRES_INTERNAL=False`) |
| `POSTGRES_USER` | `agenticit` | Пользователь БД |

### 2.3 Secrets (секретные)

`Settings → Environments → production → Environment secrets`:

| Имя | Обязательный | Описание |
|-----|-------------|----------|
| `AZURE_CREDENTIALS` | ✅ | JSON от `az ad sp create-for-rbac --sdk-auth` |
| `POSTGRES_PASSWORD` | ✅ | Пароль БД (пустой = helm сгенерирует случайный) |
| `OPENROUTER_API_KEY` | ✅ | OpenRouter API key (`sk-or-...`) |
| `JWT_SECRET` | ✅ | Секрет JWT (min 32 символа) |
| `TOKEN_ENCRYPTION_KEY` | ✅ | Ключ шифрования токенов (min 32 символа) |
| `FRONTEND_URL` | ✅ | `https://app.agenticit.com` |
| `BACKEND_URL` | ✅ | `https://app.agenticit.com` |
| `CORS_ORIGINS` | ✅ | `https://app.agenticit.com` |
| `OAUTH_MICROSOFT_CLIENT_ID` | ☑️ | Azure AD App ID (для Microsoft OAuth) |
| `OAUTH_MICROSOFT_CLIENT_SECRET` | ☑️ | Azure AD App Secret |
| `OAUTH_GOOGLE_CLIENT_ID` | ☑️ | Google OAuth client ID |
| `OAUTH_GOOGLE_CLIENT_SECRET` | ☑️ | Google OAuth client secret |
| `OAUTH_GITHUB_CLIENT_ID` | ☑️ | GitHub OAuth App client ID |
| `OAUTH_GITHUB_CLIENT_SECRET` | ☑️ | GitHub OAuth App client secret |
| `VBOX_OAUTH_ISSUER` | ☑️ | vBox OIDC issuer URL |
| `VBOX_OAUTH_CLIENT_ID` | ☑️ | vBox OAuth client ID |
| `VBOX_OAUTH_CLIENT_SECRET` | ☑️ | vBox OAuth client secret |
| `VBOX_OAUTH_REDIRECT_URI` | ☑️ | vBox OAuth redirect URI |
| `MCP_SERVER_URL` | ☑️ | vBox MCP server URL |
| `MCP_AUTH_TOKEN` | ☑️ | vBox MCP auth token |
| `TAVILY_API_KEY` | ☑️ | Tavily web search API key |
| `PREVIEW_TOKEN_SECRET` | ☑️ | Секрет для preview tokens |
| `ANTHROPIC_DIRECT_API_KEY` | ☑️ | Direct Anthropic API key |
| `LANGFUSE_PUBLIC_KEY` | ☑️ | Langfuse observability public key |
| `LANGFUSE_SECRET_KEY` | ☑️ | Langfuse observability secret key |
| `LANGFUSE_BASE_URL` | ☑️ | Langfuse instance URL |

> ✅ обязательный, ☑️ опциональный (оставьте пустым, если не используется)

Сгенерировать случайные секреты:
```bash
# JWT_SECRET
openssl rand -base64 48

# TOKEN_ENCRYPTION_KEY
openssl rand -base64 32
```

---

## 3. Первый деплой (вручную)

До первого деплоя через GitHub Actions нужно один раз создать namespace и установить cert-manager (если используете Let's Encrypt):

```bash
# Подключиться к кластеру
az aks get-credentials --resource-group rg-agenticit --name aks-agenticit

# Создать namespace
kubectl create namespace agenticit-ns

# Установить cert-manager (если TLS через Let's Encrypt)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=90s
```

Затем — запустить deploy workflow вручную:
`Actions → Deploy → Run workflow → main`.

---

## 4. DNS и TLS

После первого деплоя получите внешний IP Traefik:

```bash
kubectl get svc -n kube-system -l app.kubernetes.io/name=traefik
# или если Traefik в другом namespace:
kubectl get svc -A | grep traefik
```

Создайте A-запись `app.agenticit.com → <EXTERNAL_IP>`.

cert-manager автоматически получит TLS-сертификат от Let's Encrypt (занимает 1-2 минуты после появления DNS-записи).

---

## 5. Обновление приложения

Любой push в ветку `main` запускает полный CI → Deploy.

Флоу:
1. Push в main
2. CI (`ci.yml`) — для PR уже прошёл, но для прямых пушей запускается
3. `deploy.yml` — собирает образы с тегом `<short SHA>`, пушит в ACR, делает `helm upgrade --atomic`
4. Если после деплоя health check упал — helm автоматически откатывается (`--atomic`)

---

## 6. Откат вручную

```bash
# Список деплоев
helm history agenticit -n agenticit-ns

# Откат на предыдущую ревизию
helm rollback agenticit -n agenticit-ns

# Откат на конкретную ревизию
helm rollback agenticit 3 -n agenticit-ns
```

Или откатить на конкретный образ:
```bash
kubectl set image deployment/prod-agenticit-backend \
  prod-agenticit-backend=agenticitacr.azurecr.io/agenticit-backend:<TAG> \
  -n agenticit-ns
```

---

## 7. Staging окружение (опционально)

Для staging создайте второй environment в GitHub:
- `Settings → Environments → New environment → staging`
- Добавьте те же переменные, но с другими значениями (`HELM_RELEASE_NAME=staging`, `APP_DOMAIN=staging.agenticit.com` и т.д.)

В `deploy.yml` добавьте триггер на ветку `develop`:
```yaml
on:
  push:
    branches: [main, develop]
```

И используйте `environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}`.

---

## 8. Диагностика

```bash
# Статус деплоя
kubectl get pods -n agenticit-ns

# Логи backend
kubectl logs -n agenticit-ns -l app=prod-agenticit-backend --tail=100 -f

# Проверить env vars (без секретов)
kubectl exec -n agenticit-ns deployment/prod-agenticit-backend \
  -- env | grep -E "DATABASE|REDIS|AUTH|LLM|MCP"

# Health check
kubectl exec -n agenticit-ns deployment/prod-agenticit-backend \
  -- curl -sf http://localhost:8000/api/health && echo OK

# Посмотреть helm values (без секретов)
helm get values agenticit -n agenticit-ns
```
