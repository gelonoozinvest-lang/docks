Техническое задание: RAG-сервис "Knowledge Core" (Qdrant Edition)
1. Цель
Создать централизованный, высокопроизводительный RAG-сервис, использующий Open Source решение Qdrant для векторного поиска, что обеспечивает:

Нулевую стоимость лицензий на векторный движок (Self-hosted / Free Tier).
Высокую скорость поиска (Rust-based engine).
Расширенные возможности фильтрации (Payload filtering).
Хранение всех «твердых» данных и истории в PostgreSQL.
2. Архитектура сервиса
2.1. Основные компоненты
API Layer (FastAPI/Go): Интерфейс взаимодействия.
RDB Layer (PostgreSQL): Хранит пользователей, структуру коллекций, оригиналы документов, историю версий, логи.
Vector Engine (Qdrant): Хранит векторы (embeddings) и текстовые чанки (payload) для быстрого поиска.
Embeddings Service: Микросервис генерации векторов (OpenAI/HuggingFace).
Drive Syncer: Синхронизация с Google Drive.
Async Worker (Celery/Arq): Фоновая обработка файлов и загрузка в Qdrant.
2.2. Архитектурная диаграмма
graph TD
    User[Router / Agents] -->|REST API| API[RAG API Layer]
    
    API -->|Read/Write Meta| DB[(PostgreSQL)]
    API -->|Vector Search| Qdrant[(Qdrant Vector DB)]
    
    API -->|Async Task| Queue[Task Queue / Redis]
    Queue -->|Process| Worker[Background Worker]
    
    Worker -->|Fetch| GDrive[Google Drive]
    Worker -->|Generate| Embed[Embeddings Model]
    Worker -->|Upsert Vectors| Qdrant
    Worker -->|Save Versions| DB
3. Модели данных
3.1. Реляционная модель (PostgreSQL)
Здесь храним "скелет" данных. Таблица embeddings удалена, так как переезжает в Qdrant.

-- 1. Структура и доступы
CREATE TABLE sites (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(255)
);

CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    site_id INTEGER REFERENCES sites(id),
    name VARCHAR(255) NOT NULL
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE,
    role VARCHAR(50) CHECK (role IN ('admin', 'agent', 'system', 'client')),
    department_id INTEGER REFERENCES departments(id)
);

CREATE TABLE clients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id VARCHAR(255),
    name VARCHAR(255),
    site_id INTEGER REFERENCES sites(id)
);

-- 2. Документы (Источник правды)
CREATE TABLE collections (
    id SERIAL PRIMARY KEY,
    owner_type VARCHAR(50), -- 'site', 'department'
    owner_id INTEGER,
    name VARCHAR(255)
);

CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id INTEGER REFERENCES collections(id),
    title VARCHAR(512),
    source_type VARCHAR(50), -- 'gdrive', 'manual'
    external_id VARCHAR(255), -- Google Drive ID
    last_checksum VARCHAR(64),
    current_version INT DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE document_versions (
    id SERIAL PRIMARY KEY,
    document_id UUID REFERENCES documents(id),
    version_number INT,
    content TEXT, -- Полный текст версии
    change_reason VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 3. Логи и Задачи
CREATE TABLE tasks (
    id UUID PRIMARY KEY,
    type VARCHAR(50), -- 'sync_qdrant', 'reindex'
    status VARCHAR(50),
    payload JSONB
);

CREATE TABLE agent_logs (
    id UUID PRIMARY KEY,
    agent_id UUID,
    query_text TEXT,
    qdrant_search_result JSONB, -- Снапшот ответа от Qdrant
    response TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
);

CREATE TABLE client_messages (
    id UUID PRIMARY KEY,
    client_id UUID,
    content TEXT,
    direction VARCHAR(10),
    timestamp TIMESTAMP DEFAULT NOW()
);

CREATE TABLE document_change_log (
    id SERIAL PRIMARY KEY,
    document_id UUID REFERENCES documents(id),
    version_id INT REFERENCES document_versions(id),
    change_type VARCHAR(50), -- 'created', 'updated', 'deleted'
    changed_by_user_id UUID REFERENCES users(id),
    timestamp TIMESTAMP DEFAULT NOW(),
    details JSONB
);
3.2. Векторная модель (Qdrant Collections)
В Qdrant создается одна (или несколько) коллекций. Рекомендуется одна коллекция factory_knowledge с использованием Payload Filtering для разделения по сайтам/отделам.

Конфигурация коллекции Qdrant:

Name: factory_knowledge
Vector Size: 1536 (для OpenAI text-embedding-3-small) или 1024 (для mE5-large).
Distance: Cosine.
Структура Point (Точки):

ID: UUID (генерируется детерминированно на основе document_id + chunk_index).
Vector: [0.012, -0.231, ...]
Payload (Metadata):
{
  "document_id": "uuid-from-postgres",
  "version_id": 123,
  "collection_id": 45,
  "site_id": 2,
  "department_id": 5,
  "text_chunk": "Текстовое содержимое этого фрагмента...",
  "source_url": "https://docs.google.com/...",
  "created_at": "2024-01-01T12:00:00"
}
Важно: Мы храним text_chunk прямо в Qdrant, чтобы при поиске не делать лишний запрос в PostgreSQL ("Retrieve Payload" = true).

4. Версионирование и Синхронизация (Logic)
4.1. Drive Syncer & Qdrant Upsert
Detect Change: Syncer видит изменение файла в GDrive (хеш не совпал).
RDB Update:
Создает новую запись в document_versions (PostgreSQL).
Обновляет documents (PostgreSQL).
Vector Re-indexing (Critical Path):
Шаг 1: Удаление старых векторов из Qdrant.
Filter: Must: [{ key: "document_id", match: { value: <doc_uuid> } }]
Action: Delete points.
Шаг 2: Нарезка нового текста (Chunking).
Шаг 3: Генерация эмбеддингов.
Шаг 4: Загрузка новых точек в Qdrant с обновленным version_id в Payload.
4.2. Versioning Engine
Позволяет "путешествовать во времени".

Если агент хочет искать по состоянию знаний на "прошлый месяц", поиск идет в PostgreSQL (медленный текстовый поиск) или по архиву версий, так как Qdrant хранит актуальное состояние (для экономии ресурсов).
Опционально: Можно хранить старые версии в Qdrant, добавляя поле is_active: false в Payload, и фильтровать при поиске.
5. Embeddings и Vector Search
Провайдер: OpenAI (text-embedding-3-small) или Local (Ollama/SentenceTransformers).
Qdrant Search Query:
Запросы всегда содержат фильтр безопасности.
{
    "vector": [0.1, ...],
    "filter": {
        "must": [
            { "key": "site_id", "match": { "value": 1 } },
            { "key": "is_active", "match": { "value": true } }
        ]
    },
    "limit": 5,
    "with_payload": true
}
6. Авторизация и Безопасность
JWT: Стандартный токен.
Qdrant Security: Qdrant находится внутри закрытого контура (Private Network). Доступ к нему имеет только API Layer. Прямой доступ из интернета закрыт.
API Key Management: API сервис управляет ключами доступа к коллекциям. При запросе пользователя API подставляет соответствующие filter в запрос к Qdrant (например, принудительный фильтр по department_id).

См. `Identity & Access Layer (IAM).md` для получения дополнительной информации об аутентификации и авторизации.
7. API Спецификация (REST)
7.1. Search API (Main)
POST /search
Body: {"query": "текст", "filters": {"site_id": 1}, "limit": 5}
Logic:
API переводит текст в вектор.
API делает запрос в Qdrant с фильтрами.
API логирует запрос и найденные ID документов в agent_logs.
Возвращает список чанков текста.
7.2. Management API
POST /documents/sync — Запуск воркера синхронизации.
GET /collections/{id}/stats — Статистика из Qdrant (количество векторов, использование памяти).
7.3. CRUD API (Postgres)
Стандартные эндпоинты для Users, Clients, Sites, Collections, Documents (как в обычной CRM).

8. Масштабирование (Qdrant specific)
Sharding: Qdrant поддерживает распределенный режим (Distributed deployment). При росте данных можно добавить ноды и включить шардирование коллекции.
Quantization: Для экономии RAM (до 4х раз) в Qdrant включается скалярная квантованизация (Scalar Quantization) — почти без потери точности поиска.
On-disk storage: Если векторы не влезают в RAM, Qdrant настраивается на хранение векторов на NVMe диске (Memmap storage).
9. Результат
Фабрика получает бесплатный в эксплуатации (за исключением токенов LLM), но промышленный по мощности поисковый движок. Qdrant берет на себя всю сложность математики векторов, а PostgreSQL обеспечивает надежность хранения бизнес-данных.

API Specification: RAG Service
1. Overview
The RAG Service API provides access to the centralized knowledge base of the development factory. It supports operations for managing users, clients, documents, embeddings, and search functionality. The API is RESTful and uses JSON for data exchange.
2. Authentication
Authentication is based on JWT tokens. The system supports the following roles: admin, agent, client, and system. Each role has specific permissions to access and modify resources. See `Identity & Access Layer (IAM).md` for more details.
3. Entities
The following entities are supported by the RAG Service API:
- users
- clients
- sites
- departments
- collections
- documents
- document_versions
- embeddings
- agent_logs
- client_messages
- tasks
- document_change_log
4. REST Endpoints
Users API
Method: GET
URL: /api/v1/users
Description: Get a list of users.
---
Method: GET
URL: /api/v1/users/{id}
Description: Get a specific user.
---
Method: POST
URL: /api/v1/users
Description: Create a new user.
---
Method: PUT
URL: /api/v1/users/{id}
Description: Update a user.
---
Method: DELETE
URL: /api/v1/users/{id}
Description: Delete a user.

Clients API
Method: GET
URL: /api/v1/clients
Description: Get a list of clients.
---
Method: GET
URL: /api/v1/clients/{id}
Description: Get a specific client.

Sites API
Method: GET
URL: /api/v1/sites
Description: Get a list of sites.

Departments API
Method: GET
URL: /api/v1/departments
Description: Get a list of departments.

Collections API
Method: GET
URL: /api/v1/collections
Description: Get a list of collections.
---
Method: POST
URL: /api/v1/collections
Description: Create a new collection.

Documents API
Method: GET
URL: /api/v1/documents
Description: Get a list of documents.
---
Method: POST
URL: /api/v1/documents
Description: Create a new document.

Document Versions API
Method: GET
URL: /api/v1/documents/{document_id}/versions
Description: Get all versions of a document.
---
Method: GET
URL: /api/v1/documents/{document_id}/versions/{version_id}
Description: Get a specific version of a document.

Embeddings API
Method: POST
URL: /api/v1/embeddings/search
Description: Search for embeddings in Qdrant.
Request Body: { "vector": [0.1, ...], "limit": 5, "filter": { "must": [{ "key": "site_id", "match": { "value": 1 } }] } }

Search API
Method: POST
URL: /api/v1/search
Description: Perform a RAG search.
Request Body: { "query": "текст", "filters": {"site_id": 1}, "limit": 5 }

Agent Logs API
Method: GET
URL: /api/v1/agent_logs
Description: Get a list of agent logs.

Client Messages API
Method: GET
URL: /api/v1/client_messages
Description: Get a list of client messages.

Tasks API
Method: GET
URL: /api/v1/tasks
Description: Get a list of tasks.
---
Method: POST
URL: /api/v1/tasks
Description: Create a new task.

Document Change Log API
Method: GET
URL: /api/v1/document_change_log
Description: Get a list of document change logs.

5. Data Models
Each entity has a corresponding JSON schema. Below is an example for the "user" entity:
{
  "id": "string",
  "name": "string",
  "email": "string",
  "role": "admin | agent | client | system",
  "created_at": "datetime"
}
6. Error Handling
All errors follow a standard format:
{
  "error": {
    "code": "string",
    "message": "string",
    "details": "optional"
  }
}
Common error codes include: 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 500 Internal Server Error.
7. Rate Limits
The API enforces rate limits per user and per IP. Default: 1000 requests/hour.
8. Versioning
The API uses URI-based versioning. Example: /api/v1/resource
9. Security Notes
Access control is enforced via roles. Data isolation is applied per client and site. All operations are logged for audit purposes.
10. Related Documents
- `Router AI Agents.md`
- `Identity & Access Layer (IAM).md`
- `Agent Context Search Specification.md`
