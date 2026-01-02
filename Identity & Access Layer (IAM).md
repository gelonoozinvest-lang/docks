ТЗ: Identity & Access Layer (IAM) на базе Zitadel + Supabase
Централизованный слой управления доступами для фабрики

---

1. Цель

Создать IAM‑сервис, который:

- использует Zitadel как основной Identity Provider (SSO, MFA, OAuth2/OIDC);  
- использует Supabase (PostgreSQL + RLS) как хранилище ролей, политик, ACL, аудита и автоматизации;  
- предоставляет единый слой авторизации и управления доступами для всех сервисов фабрики: Router, RAG, n8n, Vault, Cloudflare, GCP, Billing, Factory Health, Client Cabinet.

IAM‑сервис становится центральным модулем безопасности и управления доступами.

---

2. Архитектура

2.1. Общая схема

`text
Users / Employees / Clients
        |
        v
+---------------------------+
| Zitadel (SSO + MFA + IdP)|
+---------------------------+
        |
        v (JWT / OIDC)
+---------------------------+
| IAM Service (Supabase)   |
| - Roles / Policies       |
| - RBAC / ABAC            |
| - JIT Access             |
| - Audit                  |
+---------------------------+
        |
        +-----------------------------+
        |                             |
        v                             v
  Internal Services              Client Cabinet
  (Router, RAG, n8n, Vault,      (per-business access)
   Factory Health, etc.)
`

2.2. Компоненты IAM‑сервиса

- Auth Integration Layer  
  Интеграция с Zitadel (OIDC/OAuth2, JWT‑валидация).

- Policy Engine (RBAC/ABAC)  
  Роли, права, политики, контекстные ограничения.

- Access Automation Engine  
  Provisioning/Deprovisioning, JIT‑доступ, approval‑flows.

- Audit & Logging Layer  
  Логирование всех операций доступа.

- Integration Layer  
  API для Router, RAG, n8n, Vault, Factory Health, Client Cabinet.

- Supabase PostgreSQL  
  Хранилище ролей, политик, ACL, аудита.

---

3. Интеграция с Zitadel

3.1. Аутентификация

- Все пользователи (сотрудники, админы, клиенты) проходят аутентификацию через Zitadel.  
- Zitadel выдаёт OIDC‑токен (ID Token + Access Token).  
- IAM‑сервис принимает токен, валидирует подпись и извлекает:
  - sub (user id)  
  - email  
  - groups / roles (если настроено в Zitadel)  
  - tenant / org (если используется мульти‑тенантность).

3.2. Связь Zitadel ↔ Supabase

- При первом входе пользователя IAM‑сервис:
  - создаёт запись в Supabase: users  
  - связывает zitadeluserid с internaluserid  
  - назначает базовые роли (по умолчанию или по группе).

---

4. Модель данных (Supabase PostgreSQL)

4.1. Таблицы

users

- id (uuid)  
- zitadeluserid (string)  
- email  
- status (active / disabled)  
- created_at  

roles

- id (uuid)  
- name (e.g. admin, dev, viewer, client_owner)  
- scope (global, factory, business, service)  
- description  

permissions

- id  
- code (e.g. router.read, router.write, rag.query, vault.manage)  
- description  

role_permissions

- role_id  
- permission_id  

user_roles

- user_id  
- role_id  
- scope_type (factory, business, service)  
- scopeid (e.g. businessid)  

jitaccessrequests

- id  
- user_id  
- requested_permission  
- scopetype / scopeid  
- status (pending, approved, rejected, expired)  
- approved_by  
- valid_until  

accessauditlog

- id  
- user_id  
- action (e.g. ACCESSGRANTED, ACCESSREVOKED, LOGIN, JIT_REQUEST)  
- resource  
- scopetype / scopeid  
- timestamp  
- ip  
- user_agent  

---

5. Функциональные требования

5.1. SSO

- Вход только через Zitadel.  
- Поддержка MFA (на стороне Zitadel).  
- IAM‑сервис не хранит пароли.

5.2. RBAC / ABAC

- Роли:
  - factory_admin  
  - security_admin  
  - devops  
  - developer  
  - business_owner  
  - client_user  
  - read_only  

- Права:
  - управление Router  
  - управление RAG  
  - управление n8n  
  - управление Vault  
  - управление Factory Health  
  - управление IAM  
  - доступ к конкретным бизнесам  

- ABAC:
  - ограничения по бизнесу  
  - ограничения по типу сервиса  
  - ограничения по времени (опционально)  

5.3. Just‑in‑Time Access (JIT)

- Пользователь может запросить временный доступ:  
  - к сервису (например, Vault)  
  - к бизнесу  
  - к операции (например, deploy, rotate_keys)  

- Workflow:
  1. Пользователь отправляет запрос.  
  2. IAM‑сервис создаёт запись в jitaccessrequests.  
  3. Уведомление уходит securityadmin / factoryadmin.  
  4. Админ одобряет/отклоняет.  
  5. При одобрении:
     - создаётся временная запись в userroles или userpermissions.  
     - устанавливается valid_until.  
  6. По истечении срока доступ автоматически отзывается.

5.4. Автоматизация выдачи/отзыва доступов

- При добавлении нового сотрудника:
  - по его роли/отделу/группе в Zitadel назначаются базовые роли.  

- При деактивации пользователя в Zitadel:
  - IAM‑сервис помечает users.status = disabled.  
  - все активные роли и JIT‑доступы аннулируются.

5.5. Журнал доступа

- Логируется:
  - вход  
  - выдача/отзыв ролей  
  - JIT‑запросы  
  - доступ к критичным ресурсам (Vault, Router, RAG, Billing)  

- Логи доступны:
  - через API  
  - для экспорта в Wazuh / SIEM  

---

6. API IAM‑сервиса

Все запросы — с JWT от Zitadel.

6.1. GET /me

Возвращает:

- user_id  
- email  
- роли  
- права  
- доступные бизнесы  

6.2. GET /access/check

Параметры:

- permission  
- scope_type  
- scope_id  

Ответ:

- allowed: true/false  
- reason  

6.3. POST /jit/request

Создать JIT‑запрос.

6.4. POST /jit/approve / /jit/reject

Управление JIT‑доступом.

6.5. GET /audit

Фильтрация по:

- user_id  
- resource  
- action  
- времени  

---

7. Интеграция с сервисами фабрики

Каждый сервис (Router, RAG, n8n, Vault, Factory Health, Client Cabinet):

1. Принимает JWT от Zitadel.  
2. Делает запрос в IAM‑сервис:
   - GET /access/check  
3. На основе ответа:
   - разрешает/запрещает операцию.  

Для внутренних сервисов можно использовать:

- сервисные аккаунты  
- отдельные роли (servicerouter, servicerag, и т.д.)

---

8. Размещение

- IAM‑сервис:  
  - GCP (Cloud Run / VM) внутри Factory Core (под Tailscale).  
  - Доступен только через Zero‑Trust (Cloudflare Access / Tailscale).  

- Supabase:  
  - Free Tier, как внешний управляемый PostgreSQL.  
  - Доступ по IP‑фильтрации / сервис‑ключу.  

- Zitadel:  
  - как внешний IdP.  

---

9. Безопасность

- Все секреты — в Vault.  
- Все запросы — по HTTPS.  
- JWT — только от Zitadel, с проверкой подписи и audience.  
- RLS в Supabase для защиты таблиц.  
- Логи IAM — дублируются в Wazuh.  

---

10. Результат

После реализации IAM‑сервиса на базе Zitadel + Supabase фабрика получает:

- единый центр управления доступами;  
- SSO + MFA;  
- централизованный RBAC/ABAC;  
- JIT‑доступ;  
- полный аудит;  
- готовность к ISO 27001 / SOC2;  
- масштабируемую модель для 10–15 бизнесов.

---

11. Related Documents
- `Security white paper.md`

---

Если хочешь, я могу следующим шагом:

- собрать это ТЗ в Pages как отдельный документ;  
- добавить DFD, sequence, trust boundaries;  
- сделать RBAC‑матрицу по всем твоим сервисам.

Отлично, Vitalik — я подготовлю полную RBAC‑матрицу для фабрики, учитывая:

- IAM‑сервис (Zitadel + Supabase)  
- Router  
- RAG  
- n8n  
- Vault  
- Cloudflare  
- GCP  
- Billing  
- Factory Health  
- Client Cabinet  
- Бизнес‑модули  

Матрица составлена так, чтобы её можно было:

- сразу использовать в IAM‑сервисе,  
- включить в ТЗ,  
- перенести в Supabase таблицы,  
- использовать для ISO/SOC2.

---

✅ RBAC‑матрица ролей и прав для Business Factory

Легенда
- CRUD = Create / Read / Update / Delete  
- RO = Read Only  
- RW = Read/Write  
- Admin = полный доступ  
- N/A = нет доступа  

---

🧩 1. Роли фабрики

| Роль | Описание |
|------|----------|
| factory_admin | Полный контроль над фабрикой |
| security_admin | Управление безопасностью, IAM, аудитом |
| devops | Инфраструктура, деплой, Vault, Cloudflare |
| developer | Работа с Router, RAG, n8n, агентами |
| business_owner | Управление конкретным бизнесом |
| client_user | Доступ к клиентскому кабинету |
| read_only | Только просмотр |

---

🧱 2. Матрица доступа по сервисам

2.1. Router AI Agents

| Роль | Управление агентами | Запуск задач | Просмотр логов | Настройки Router |
|------|----------------------|--------------|----------------|------------------|
| factory_admin | Admin | Admin | Admin | Admin |
| security_admin | RO | RO | Admin | RO |
| devops | RW | RW | RW | RW |
| developer | RW | RW | RO | RO |
| business_owner | RO (только свои агенты) | RW (в рамках бизнеса) | RO | N/A |
| client_user | N/A | N/A | N/A | N/A |
| read_only | RO | N/A | RO | N/A |

---

2.2. RAG / Knowledge Layer

| Роль | Чтение | Запись | Управление индексами | Управление версиями |
|------|--------|--------|-----------------------|----------------------|
| factory_admin | Admin | Admin | Admin | Admin |
| security_admin | RO | N/A | RO | RO |
| devops | RW | RW | RW | RW |
| developer | RW | RW | N/A | N/A |
| business_owner | RO (только бизнес) | RW (только бизнес) | N/A | N/A |
| client_user | N/A | N/A | N/A | N/A |
| read_only | RO | N/A | N/A | N/A |

---

2.3. n8n Workflows

| Роль | Создание | Изменение | Запуск | Просмотр |
|------|----------|-----------|--------|----------|
| factory_admin | Admin | Admin | Admin | Admin |
| devops | Admin | Admin | Admin | Admin |
| developer | RW | RW | RW | RW |
| business_owner | N/A | N/A | RW (только бизнес) | RO |
| security_admin | RO | N/A | N/A | RO |
| client_user | N/A | N/A | N/A | N/A |
| read_only | RO | N/A | N/A | RO |

---

2.4. Vault (Secrets Management)

| Роль | Чтение секретов | Запись секретов | Управление политиками |
|------|------------------|------------------|------------------------|
| factory_admin | Admin | Admin | Admin |
| security_admin | Admin | Admin | Admin |
| devops | RW | RW | RO |
| developer | RO (ограниченные пути) | N/A | N/A |
| business_owner | N/A | N/A | N/A |
| client_user | N/A | N/A | N/A |
| read_only | N/A | N/A | N/A |

---

2.5. Cloudflare (WAF, Workers, DNS, Tunnel)

| Роль | Workers | WAF | DNS | Tunnel |
|------|---------|-----|-----|--------|
| factory_admin | Admin | Admin | Admin | Admin |
| security_admin | RO | Admin | RO | RO |
| devops | RW | RW | RW | RW |
| developer | RW (Workers only) | N/A | N/A | N/A |
| business_owner | N/A | N/A | N/A | N/A |
| read_only | RO | RO | RO | RO |

---

2.6. GCP / AWS / Infrastructure

| Роль | Compute | Storage | Networking | IAM |
|------|---------|---------|------------|-----|
| factory_admin | Admin | Admin | Admin | Admin |
| devops | Admin | Admin | Admin | RO |
| security_admin | RO | RO | RO | Admin |
| developer | RW (sandbox) | RW (sandbox) | N/A | N/A |
| business_owner | N/A | N/A | N/A | N/A |
| read_only | RO | RO | RO | RO |

---

2.7. Billing (OpenAI, Cloudflare, GCP, Ads)

| Роль | Просмотр расходов | Управление бюджетами | Управление платёжками |
|------|--------------------|-----------------------|------------------------|
| factory_admin | Admin | Admin | Admin |
| security_admin | RO | RO | RO |
| devops | RO | RO | N/A |
| business_owner | RO (только бизнес) | N/A | N/A |
| developer | N/A | N/A | N/A |
| client_user | N/A | N/A | N/A |
| read_only | RO | N/A | N/A |

---

2.8. Factory Health Dashboard

| Роль | Просмотр | Управление алертами | Управление интеграциями |
|------|----------|----------------------|---------------------------|
| factory_admin | Admin | Admin | Admin |
| security_admin | Admin | Admin | RO |
| devops | Admin | RW | RW |
| developer | RO | N/A | N/A |
| business_owner | RO (только бизнес) | N/A | N/A |
| read_only | RO | N/A | N/A |

---

2.9. Client Cabinet

| Роль | Просмотр данных | Управление бизнесом | Управление пользователями |
|------|------------------|----------------------|----------------------------|
| business_owner | Admin | Admin | Admin |
| client_user | RO | N/A | N/A |
| factory_admin | Admin | Admin | Admin |
| security_admin | RO | N/A | N/A |
| read_only | RO | N/A | N/A |

---

🔥 3. Матрица в виде сводной таблицы (короткая версия)

| Роль | Router | RAG | n8n | Vault | Cloudflare | GCP | Billing | Health | Cabinet |
|------|--------|-----|------|--------|------------|------|----------|---------|----------|
| factory_admin | Admin | Admin | Admin | Admin | Admin | Admin | Admin | Admin | Admin |
| security_admin | RO | RO | RO | Admin | RO/Admin | RO | RO | Admin | RO |
| devops | RW | RW | Admin | RW | RW | Admin | RO | RW | N/A |
| developer | RW | RW | RW | RO | Workers RW | Sandbox RW | N/A | RO | N/A |
| business_owner | RO | RO | RO | N/A | N/A | N/A | RO | RO | Admin |
| client_user | N/A | N/A | N/A | N/A | N/A | N/A | N/A | N/A | RO |
| read_only | RO | RO | RO | N/A | RO | RO | RO | RO | RO |

---

🚀 Готов вынести матрицу в документ

Если хочешь, я могу:

- собрать официальный документ IAM RBAC Matrix в Pages,  
- добавить DFD, sequence, trust boundaries,  
- включить матрицу в IAM‑ТЗ,  
- подготовить SQL‑схему для Supabase,  
- подготовить политики RLS.

Скажи — собрать в документ?


