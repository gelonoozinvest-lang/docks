Техническое задание: Добавление Terraform в фабрику разработки
1. Цель
Внедрить Terraform как единый инструмент управления инфраструктурой (Cloudflare, AWS, GCP) в рамках фабрики разработки. Terraform должен работать без UI, через GitOps, с секретами из Vault, с запуском через n8n или CI/CD, с модульной структурой и поддержкой мультиоблачных провайдеров.
2. Требования к архитектуре
Terraform должен:
- Работать как CLI.
- Запускаться через n8n или CI/CD.
- Хранить состояние в R2/S3/GCS.
- Получать секреты из Vault.
- Поддерживать провайдеры Cloudflare, AWS, GCP.
3. Структура репозитория Terraform
Рекомендуемая структура каталогов:
- infrastructure/modules/cloudflare/...
- infrastructure/modules/aws/...
- infrastructure/modules/gcp/...
- infrastructure/envs/prod, stage, dev
- global/providers.tf, backend.tf, variables.tf, versions.tf
4. Интеграция с Vault
- Хранение Cloudflare API Token, AWS Keys, GCP SA JSON.
- n8n передает секреты Terraform через переменные окружения.
5. Интеграция с n8n
Workflow terraform_apply: получение параметров, получение секретов, формирование tfvars, запуск init/plan/apply, сохранение логов, обновление Jira.
6. Требования к модулям Terraform
- Атомарность.
- Параметризация.
- README.md.
- Примеры.
7. GitOps
- Любое изменение в Git вызывает деплой.
- Все изменения через PR.
- Git — источник правды.
8. Результат
Фабрика должна уметь автоматически создавать, обновлять, восстанавливать инфраструктуру, документировать изменения, анализировать ошибки и подключать Cloudflare/AWS/GCP модули.
