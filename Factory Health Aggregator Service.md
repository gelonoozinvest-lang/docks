Техническое задание: Factory Health Aggregator Service

Единый сервис агрегации технических, бизнес‑, маркетинговых и финансовых метрик фабрики

---

1. Цель

Создать сервис Factory Health Aggregator, который:

- собирает данные из всех технических, бизнес‑, маркетинговых, рекламных и финансовых систем фабрики;  
- нормализует и агрегирует их;  
- вычисляет статусы, риски, SLA, аномалии;  
- предоставляет единый API для панели «Factory Health»;  
- обеспечивает мониторинг всех бизнесов фабрики в реальном времени.

Сервис является центральным источником правды (Single Source of Truth) для состояния фабрики.

---

2. Основные функции сервиса

2.1. Сбор данных из всех источников
Сервис должен собирать данные из:

Технические источники
- Prometheus (метрики сервисов)  
- Wazuh (security alerts, см. `Wazuh.md`)
- Cloudflare (WAF, Worker, Traffic, Errors)  
- Tailscale (узлы, ACL, трафик)  
- Vault (health, seal status)  
- n8n (workflow status)  
- Router AI Agents (очереди, SLA, ошибки)  
- RAG (latency, index health)  

Маркетинговые источники
- Google Analytics 4  
- Google Ads  
- Meta Ads  
- TikTok Ads  
- LinkedIn Ads (опционально)

Финансовые источники
- OpenAI billing  
- Anthropic billing  
- Google Cloud billing  
- AWS billing  
- Cloudflare billing  
- Tailscale billing  
- PSP (Stripe, PayPal, Revolut)  
- Рекламные бюджеты  

---

2.2. Нормализация данных
Сервис должен:

- приводить данные к единому формату  
- вычислять агрегированные показатели  
- объединять данные по бизнесам  
- вычислять SLA, SLO, SLI  
- вычислять ROI, ROAS, CAC, LTV  
- определять статусы (OK / Warning / Critical)

---

2.3. Вычисление рисков и аномалий
Сервис должен автоматически определять:

- резкое падение трафика  
- рост ошибок  
- рост задержек  
- превышение бюджета  
- падение ROAS  
- отключение рекламных кампаний  
- блокировки аккаунтов  
- аномалии в расходах  
- аномалии в поведении пользователей  
- аномалии в работе агентов  

---

2.4. Формирование единого API
Сервис предоставляет REST API:

GET /health
Общее состояние фабрики.

GET /business/:id
Состояние конкретного бизнеса.

GET /agents
Состояние всех агентов.

GET /billing
Расходы по сервисам.

GET /ads
Рекламные метрики.

GET /analytics
GA4‑метрики.

GET /security
Состояние безопасности.

GET /alerts
Алерты и риски.

---

2.5. Кэширование и обновление
- обновление данных каждые 30–300 секунд (по типу источника)  
- кэширование для снижения нагрузки  
- асинхронные обновления  
- очереди задач (Redis / PubSub)  

---

3. Архитектура сервиса

`
+-----------------------------------------------------------+
|                 Factory Health Aggregator                 |
+----------------------+------------------+------------------+
                       |                  |
                       v                  v
        +------------------------+   +------------------------+
        | Technical Collectors   |   | Business Collectors    |
        | Prometheus, Wazuh,     |   | GA4, Ads, Billing      |
        | Cloudflare, Router     |   | PSP, Revenue           |
        +------------------------+   +------------------------+
                       |                  |
                       +---------+--------+
                                 |
                                 v
                     +------------------------+
                     | Normalization Engine   |
                     | - Metrics fusion       |
                     | - SLA/SLO              |
                     | - Risk scoring         |
                     | - Anomaly detection    |
                     +------------------------+
                                 |
                                 v
                     +------------------------+
                     | Unified Health Model   |
                     | (Single Source of Truth)|
                     +------------------------+
                                 |
                                 v
                     +------------------------+
                     | Factory Health API     |
                     +------------------------+
`

---

4. Модульная структура

4.1. Collectors
Каждый источник — отдельный модуль:

- GA4 Collector  
- Ads Collector  
- Billing Collector  
- Prometheus Collector  
- Wazuh Collector (см. `Wazuh.md`)
- Cloudflare Collector  
- Router Collector  
- RAG Collector  
- n8n Collector  

4.2. Normalization Engine
Выполняет:

- агрегацию  
- нормализацию  
- вычисление статусов  
- вычисление KPI  
- вычисление рисков  

4.3. Risk Engine
Определяет:

- технические риски  
- маркетинговые риски  
- финансовые риски  
- security‑риски  

4.4. API Layer
Отдаёт данные панели Factory Health.

---

5. Метрики, которые должен собирать сервис

5.1. Технические
- latency  
- error rate  
- uptime  
- queue length  
- agent health  
- RAG index health  
- n8n workflow failures  

5.2. Маркетинговые
- users  
- sessions  
- conversions  
- CTR  
- CPC  
- CPM  
- CPA  
- ROAS  

5.3. Финансовые
- расходы по сервисам  
- прогноз расходов  
- превышения лимитов  
- расходы по бизнесам  
- прибыльность  
- маржинальность  

5.4. Security
- Wazuh alerts (см. `Wazuh.md`)
- AI‑WAF blocks  
- SQL/XSS attempts  
- jailbreak attempts  
- Cloudflare WAF events  

---

6. Алерты

Сервис должен генерировать алерты:

- Telegram  
- Slack  
- Email  

Типы алертов:

- технические  
- маркетинговые  
- финансовые  
- security  
- бизнесовые  

---

7. Требования к производительности

- обновление данных: 30–300 секунд  
- API latency: < 100 ms  
- поддержка 10–15 бизнесов  
- горизонтальное масштабирование  

---

8. Требования к безопасности

- Zero‑Trust  
- API‑ключи  
- OAuth2 для GA/Ads  
- шифрование секретов в Vault  
- логирование в Wazuh (см. `Wazuh.md`)
- rate limiting  

---

9. Результат

После реализации Factory Health Aggregator:

- фабрика получает единый экран состояния  
- все бизнесы контролируются в реальном времени  
- маркетинг, техничка, финансы и безопасность объединены  
- риски обнаруживаются автоматически  
- принятие решений ускоряется  
- фабрика становится управляемой как система  

---

10. Related Documents
- `Service Discovery Layer.md`
- `Wazuh.md`

---

Если хочешь, я могу:

✔ собрать это ТЗ в Pages

✔ добавить DFD, sequence, trust boundaries

✔ включить в Security Whitepaper

✔ подготовить архитектурную диаграмму уровня CTO

Скажи — собрать в документ?
