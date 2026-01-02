Vault Namespace Specification (VNS v1.0)
Стандарт организации секретов для модульной бизнес‑фабрики
1. Цель документа
Этот документ определяет единый, масштабируемый, детерминированный стандарт хранения секретов в HashiCorp Vault для всех автоматизаций:
n8n
Google API
GitHub
HubSpot
Telegram
AWS / GCP
CRM / Ads / Social
DevOps / CI/CD
Стандарт обеспечивает:
горизонтальное масштабирование
изоляцию клиентов
изоляцию окружений
предсказуемость путей
автоматизацию онбординга
безопасность и ротацию секретов
2. Принципы организации пространства
2.1. Иерархия Vault строится по схеме:
Код
/environment/client/integration/resource

2.2. Каждый уровень выполняет свою роль
Уровень
Назначение
environment
dev / staging / prod / sandbox
client
клиент, проект или internal‑пространство
integration
GitHub, Google, HubSpot, Telegram, AWS, GCP…
resource
конкретный секрет: token, api_key, service_account

3. Базовая структура Vault
Код
kv/
  data/
    prod/
      clientA/
        github/
          token
        google/
          service_account
        hubspot/
          api_key
        telegram/
          bot_token
      clientB/
        ...
    staging/
      clientA/
        github/
        google/
    dev/
      internal/
        github/
        google/
        openai/

4. Правила именования
4.1. environment
Используются только нижний регистр:
prod
staging
dev
sandbox
qa
4.2. client
Только латиница, без пробелов:
clientA
clientB
internal
4.3. integration
Название сервиса в нижнем регистре:
github
google
hubspot
telegram
aws
gcp
4.4. resource
Тип секрета:
token
api_key
service_account
credentials
bot_token
5. Примеры путей
5.1. GitHub токен клиента A в проде
Код
kv/data/prod/clientA/github/token

5.2. Google Service Account клиента B в staging
Код
kv/data/staging/clientB/google/service_account

5.3. Telegram bot token для внутренней автоматизации
Код
kv/data/dev/internal/telegram/bot_token

6. Формат хранения секретов
6.1. GitHub
json
{
  "token": "ghp_xxx"
}

6.2. Google Service Account
json
{
  "client_email": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n",
  "project_id": "my-project",
  "token_uri": "https://oauth2.googleapis.com/token"
}

6.3. Telegram
json
{
  "bot_token": "123456:ABCDEF"
}

7. Политика доступа (ACL)
Роль
Доступ
n8n-prod
read-only к prod/*
n8n-staging
read-only к staging/*
n8n-dev
read-only к dev/*
admin
read/write ко всему
client‑scoped
read-only к prod/clientX/*

8. Интеграция с MDS‑модулями
Каждый модуль получает:
json
{
  "auth": {
    "vault_path": "prod/clientA/github/token",
    "keys": ["token"]
  }
}

Модуль Vault.FetchSecrets возвращает:
json
{
  "data": {
    "token": "ghp_xxx"
  }
}

Любой модуль (GitHub, Google, HubSpot) принимает это напрямую.
9. Масштабирование по горизонтали
Структура поддерживает:
✔ добавление новых клиентов
Код
prod/clientC/...
prod/clientD/...

✔ добавление новых интеграций
Код
prod/clientA/slack/
prod/clientA/stripe/
prod/clientA/notion/

✔ добавление новых окружений
Код
preview/
qa/
sandbox/

✔ автоматический онбординг клиента
Скрипт создаёт:
Код
prod/clientX/github/
prod/clientX/google/
prod/clientX/hubspot/
prod/clientX/telegram/

10. Антипаттерны (запрещено)
❌ хранить все секреты в одном месте
Код
kv/data/integrations/github

❌ смешивать окружения
Код
kv/data/github/prod
kv/data/github/dev

❌ хранить разные интеграции в одном JSON
Код
github + google + hubspot в одном документе

❌ использовать client_id в ключах
Код
client_12345_github_token

❌ использовать плоскую структуру
Код
kv/data/github_token

11. Related Documents
- `Module Design Specification.md`
