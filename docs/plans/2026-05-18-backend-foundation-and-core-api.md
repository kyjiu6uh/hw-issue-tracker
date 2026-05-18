# Backend foundation и core API — план реализации

Дата: 2026-05-18  
Источник: `docs/designs/2026-05-18-issue-tracker-ai-triage.md`  
Назначение: построить backend-основание issue tracker без AI-логики, но с готовой моделью данных для последующей AI-интеграции.

## Область плана

План покрывает ASP.NET Core Web API, PostgreSQL, EF Core, доменную модель, auth, projects, labels, issues, comments, OpenAPI и базовые backend-тесты. AI provider, prompt engineering и AI UI вынесены в отдельный план.

## Порядок реализации

### 1. Solution и инфраструктурный каркас

- [ ] Создать директорию `backend/` и solution для .NET 9.
- [ ] Создать проекты `HwIssueTracker.Api`, `HwIssueTracker.Domain`, `HwIssueTracker.Application`, `HwIssueTracker.Infrastructure` и `HwIssueTracker.Tests`.
- [ ] Настроить project references так, чтобы `Api` зависел от `Application` и `Infrastructure`, `Application` от `Domain`, `Infrastructure` от `Application`/`Domain`, а `Domain` не зависел от остальных слоев.
- [ ] Добавить базовую конфигурацию ASP.NET Core, dependency injection, ProblemDetails и OpenAPI.
- [ ] Подключить PostgreSQL provider для EF Core и конфигурацию connection string через appsettings/user secrets/environment variables.
- [ ] Добавить clock abstraction для UTC timestamps.
- [ ] Добавить единый формат ошибок через `ProblemDetails` для validation, unauthorized, forbidden, not found, conflict и upstream failures.

### 2. Доменная модель и persistence

- [ ] Создать доменные сущности `User`, `Project`, `Label`, `Issue`, `Comment`, `AiTriageSuggestion`.
- [ ] Создать enum `IssueStatus` со значениями `backlog`, `todo`, `in-progress`, `in-review`, `done`.
- [ ] Создать enum `IssuePriority` со значениями `low`, `medium`, `high`, `urgent`.
- [ ] Реализовать доменные правила issue creation: `status=backlog`, `priority=medium`, `assignee=null`, `reporter=current user`.
- [ ] Реализовать доменное правило неизменяемости `reporter_id` после создания issue.
- [ ] Реализовать доменное правило `closed_at`: установить при переходе в `done`, очистить при выходе из `done`.
- [ ] Реализовать нормализацию label name в `name_normalized = lower(trim(name))`.
- [ ] Настроить EF Core mappings для таблиц `users`, `projects`, `labels`, `issues`, `issue_labels`, `comments`, `ai_triage_suggestions`.
- [ ] Настроить UUID primary keys и `timestamptz` для всех timestamp fields.
- [ ] Хранить enum в базе как text для читаемости миграций.
- [ ] Настроить уникальный индекс `users(email)`.
- [ ] Настроить уникальный индекс `projects(slug)`.
- [ ] Настроить индекс `projects(owner_id)`.
- [ ] Настроить уникальный индекс `labels(project_id, name_normalized)`.
- [ ] Настроить индексы `issues(project_id, status, updated_at desc)`, `issues(project_id, assignee_id, updated_at desc)` и `issues(project_id, priority, updated_at desc)`.
- [ ] Настроить уникальный индекс `issue_labels(issue_id, label_id)` и индекс `issue_labels(label_id, issue_id)`.
- [ ] Настроить индекс `comments(issue_id, created_at asc)`.
- [ ] Добавить PostgreSQL FTS индекс по `title` и `description` с конфигурацией `simple`.
- [ ] Создать первую EF Core migration и проверить применимость к пустой PostgreSQL базе.
- [ ] Добавить seed/debug data для локальной разработки: несколько пользователей, проект, labels, issues и comments.

### 3. Auth и security baseline

- [ ] Реализовать `AuthService` для register, login и current user lookup.
- [ ] Добавить password hashing через ASP.NET Core `PasswordHasher<TUser>`.
- [ ] Добавить JWT issuing с signing key из configuration/secrets.
- [ ] Реализовать `POST /api/v1/auth/register` с conflict при duplicate email.
- [ ] Реализовать `POST /api/v1/auth/login` с единообразной ошибкой при неверных credentials.
- [ ] Реализовать `GET /api/v1/auth/me` для текущего профиля.
- [ ] Защитить все endpoints, кроме register/login, JWT Bearer authentication.
- [ ] Убедиться, что backend не принимает клиентские изменения системных полей: `reporterId`, `createdAt`, `closedAt`.
- [ ] Реализовать `403 Forbidden` для операций edit/delete чужого comment.

### 4. Users, projects и labels API

- [ ] Реализовать `GET /api/v1/users?query=` для assignee picker с ограничением размера результата.
- [ ] Реализовать `GET /api/v1/projects`.
- [ ] Реализовать `POST /api/v1/projects` с валидацией `slug` по `^[a-z0-9]+(?:-[a-z0-9]+)*$`.
- [ ] Реализовать `GET /api/v1/projects/{projectSlug}`.
- [ ] Реализовать `PATCH /api/v1/projects/{projectSlug}`.
- [ ] Реализовать `DELETE /api/v1/projects/{projectSlug}`.
- [ ] Реализовать `GET /api/v1/projects/{projectSlug}/labels`.
- [ ] Реализовать `POST /api/v1/projects/{projectSlug}/labels` с валидацией hex color `#RRGGBB`.
- [ ] Реализовать `PATCH /api/v1/projects/{projectSlug}/labels/{labelId}`.
- [ ] Реализовать `DELETE /api/v1/projects/{projectSlug}/labels/{labelId}` со снятием label со всех issues через cascade на `issue_labels`.
- [ ] Возвращать `409 Conflict` для duplicate project slug и duplicate label name внутри project.

### 5. Issues API без AI

- [ ] Реализовать `GET /api/v1/projects/{projectSlug}/issues` с query parameters `status`, `assigneeId`, `labelIds`, `q`, `includeClosed`, `page`, `pageSize`, `sort`.
- [ ] Сделать `includeClosed=false` значением по умолчанию, чтобы `done` issues скрывались без явного запроса.
- [ ] Показать `done` issues при `includeClosed=true` или при явном фильтре `status=done`.
- [ ] Реализовать текстовый поиск по `title` и `description` через PostgreSQL FTS.
- [ ] Реализовать пагинацию и стабильную сортировку по `updated_at desc` по умолчанию.
- [ ] Реализовать `POST /api/v1/projects/{projectSlug}/issues` для quick create `{ title }` и expanded create `{ title, description? }`.
- [ ] Игнорировать или запрещать входящие `status`, `priority`, `assignee`, `reporter` при создании issue.
- [ ] Реализовать `GET /api/v1/projects/{projectSlug}/issues/{issueId}`.
- [ ] Реализовать `PATCH /api/v1/projects/{projectSlug}/issues/{issueId}` для `title`, `description`, `status`, `priority`, `assigneeId`, `labelIds`, `acceptanceCriteria`, `acceptanceCriteriaAiSuggested`.
- [ ] Валидировать, что `labelIds` принадлежат тому же project.
- [ ] Валидировать, что `assigneeId`, если задан, относится к существующему пользователю.
- [ ] Реализовать `DELETE /api/v1/projects/{projectSlug}/issues/{issueId}`.
- [ ] Возвращать `404 Not Found`, если issue не принадлежит указанному project.

### 6. Comments API

- [ ] Реализовать `GET /api/v1/projects/{projectSlug}/issues/{issueId}/comments` с сортировкой `created_at asc`.
- [ ] Реализовать `POST /api/v1/projects/{projectSlug}/issues/{issueId}/comments` с `author=current user`.
- [ ] Реализовать `PATCH /api/v1/projects/{projectSlug}/issues/{issueId}/comments/{commentId}` только для автора comment.
- [ ] Реализовать `DELETE /api/v1/projects/{projectSlug}/issues/{issueId}/comments/{commentId}` только для автора comment.
- [ ] Обновлять `updated_at` при изменении comment.
- [ ] Использовать hard delete для comments в первой версии.

### 7. OpenAPI и backend verification

- [ ] Убедиться, что все public DTO отражены в OpenAPI без протекания EF entities напрямую.
- [ ] Зафиксировать OpenAPI generation как часть локальной разработки или CI-команды.
- [ ] Добавить backend unit tests для issue defaults, reporter immutability, `closed_at`, label uniqueness, label project ownership и comment ownership.
- [ ] Добавить backend integration tests для auth register/login/me.
- [ ] Добавить backend integration tests для projects CRUD и duplicate slug conflict.
- [ ] Добавить backend integration tests для labels CRUD и duplicate name conflict.
- [ ] Добавить backend integration tests для issue list filters: status, assignee, labels, query, includeClosed и pagination.
- [ ] Добавить backend integration tests для comments lifecycle и forbidden edit/delete чужого comment.
- [ ] Проверить локально миграции, запуск API и smoke requests через OpenAPI UI или HTTP collection.

## Критерии готовности backend core

- [ ] Backend запускается локально с PostgreSQL и примененной первой миграцией.
- [ ] Register/login/me работают через JWT Bearer token.
- [ ] Projects и labels поддерживают полный CRUD и корректные conflict responses.
- [ ] Issue quick create работает с правильными defaults и reporter.
- [ ] Issue patch соблюдает status/closed rules, label ownership и protected system fields.
- [ ] Issue list поддерживает фильтры, search, pagination и скрывает closed issues по умолчанию.
- [ ] Comments доступны, сортируются по `created_at asc`, а edit/delete чужого comment возвращает forbidden.
- [ ] OpenAPI доступен и пригоден для frontend-типа или генерации API client.
