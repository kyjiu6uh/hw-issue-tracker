# Issue Tracker с AI-автотриажем — дизайн-документ SDD

Дата: 2026-05-18  
Статус: дизайн для последующего планирования реализации  
Целевой стек: .NET 9, PostgreSQL, React + TypeScript + Vite, REST API

## 1. Контекст и цель

Нужно построить full-stack issue-трекер с базовыми сценариями управления проектами, задачами, комментариями, метками и AI-фичей автотриажа. Основной продуктовый фокус — быстрый прием входящих задач и снижение ручной работы при разборе backlog. AI не принимает финальных решений: он предлагает приоритет, метки и черновик acceptance criteria, а пользователь принимает, редактирует или игнорирует результат.

Система проектируется как небольшой, прагматичный продукт без избыточной enterprise-сложности: нет организаций, ролей, OAuth, членства в проектах, аудита всех изменений, уведомлений, вложений и real-time синхронизации. Эти вещи можно добавить позже, но в первой версии они будут мешать скорости реализации и чистоте доменной модели.

Главные критерии успеха:

- пользователь может зарегистрироваться, войти и работать с проектами;
- issue можно создать мгновенно, указав только title;
- backlog можно фильтровать, искать и разбирать вручную или через AI suggest;
- AI-триаж использует только реальные labels проекта и возвращает структурированный результат;
- пользователь сохраняет контроль над каждым AI-предложением;
- закрытые issue скрыты по умолчанию и доступны через фильтр;
- backend и frontend имеют устойчивые контракты, пригодные для тестирования и дальнейшего развития.

## 2. Границы первой версии

### Включено

- Email/password аутентификация без OAuth.
- CRUD для проектов.
- CRUD для labels внутри проекта.
- Создание, просмотр, редактирование и фильтрация issues внутри проекта.
- Назначение assignee из любого пользователя системы.
- Управление status, priority, labels, acceptance criteria.
- Автоматическое заполнение reporter при создании issue.
- Автоматическое управление `closed_at` при входе в `done` и выходе из `done`.
- Комментарии к issue с ограничением: редактировать и удалять может только автор.
- Полнотекстовый или приближенный текстовый поиск по `title` и `description`.
- AI suggest для triage: priority, labels, acceptance criteria.
- UI для принятия AI-предложений целиком или частично.

### Не включено

- OAuth, SSO, 2FA.
- Роли, permissions и членство в проектах.
- Организации/workspaces.
- Уведомления, email-рассылки, webhooks.
- Вложения файлов.
- История изменений issue и audit log.
- Real-time updates через SignalR/WebSocket.
- Kanban drag-and-drop как обязательная функция. Можно сделать простые списки и фильтры; доска может быть отдельным улучшением.
- Обучение AI на истории решений проекта. В первой версии AI использует текущий контекст проекта и issue.

## 3. Рекомендуемый архитектурный подход

### Выбор

Рекомендуется модульный монолит: один ASP.NET Core Web API backend, одна PostgreSQL база, отдельное React SPA. Это проще и надежнее для текущего масштаба, чем микросервисы или server-side frontend. Границы модулей задаются кодом и контрактами, а не сетевыми вызовами.

### Рассмотренные альтернативы

1. **Модульный монолит + React SPA** — рекомендуемый вариант. Простая эксплуатация, быстрый локальный запуск, понятные REST-контракты, хорошая база для тестов. Минус: нужно дисциплинированно держать границы модулей, чтобы код не превратился в неструктурированный API-проект.

2. **Микросервисы** — преждевременно. AI-триаж, issues, users и projects можно разделить логически, но выделять их в отдельные сервисы сейчас дорого: потребуется межсервисная аутентификация, деплой, наблюдаемость, распределенные ошибки и усложненная локальная разработка.

3. **Blazor вместо React** — технически возможен, но React + TypeScript лучше подходит для отдельного SPA, широкого набора UI-библиотек и привычной frontend-разработки. Для команды, не ограниченной .NET-only стеком, React будет более универсален.

## 4. Структура решения

Рекомендуемая структура репозитория:

```text
hw-issue-tracker/
  backend/
    src/
      HwIssueTracker.Api/
      HwIssueTracker.Domain/
      HwIssueTracker.Application/
      HwIssueTracker.Infrastructure/
    tests/
      HwIssueTracker.Tests/
  frontend/
    src/
      app/
      features/
      shared/
  docs/
    designs/
```

Backend:

- `Api`: ASP.NET Core endpoints, authentication middleware, request/response DTO, OpenAPI.
- `Domain`: сущности, enum, доменные инварианты, простые value rules.
- `Application`: use cases/services: create issue, update status, suggest triage, apply triage, comment operations.
- `Infrastructure`: EF Core, PostgreSQL mappings, AI provider client, password hashing, clock abstraction.

Frontend:

- `app`: роутинг, bootstrap, auth provider, API client setup.
- `features/projects`: project list/detail/settings.
- `features/issues`: issue list, filters, issue detail, create/edit forms.
- `features/triage`: AI suggest panel and partial apply UI.
- `features/comments`: comment list/editor.
- `shared`: reusable UI components, markdown preview, form primitives, typed API helpers.

## 5. Доменная модель

### User

Поля:

- `id: uuid`
- `email: string`, уникален, используется как логин
- `password_hash: string`
- `name: string`
- `avatar_url: string?`
- `created_at: timestamptz`
- `updated_at: timestamptz`

Любой пользователь может создавать projects, issues, назначаться assignee и оставлять комментарии. В первой версии нет проектных ролей, поэтому проверки доступа минимальны: аутентифицированный пользователь видит и изменяет все проекты и issues.

### Project

Поля:

- `id: uuid`
- `slug: string`, уникален глобально
- `name: string`
- `description: string?`
- `owner_id: uuid`
- `created_at: timestamptz`
- `updated_at: timestamptz`

`slug` используется в URL и API. Рекомендуется разрешить только lowercase latin, digits and hyphen: `^[a-z0-9]+(?:-[a-z0-9]+)*$`.

### Label

Поля:

- `id: uuid`
- `project_id: uuid`
- `name: string`
- `name_normalized: string`
- `color: string`
- `created_at: timestamptz`
- `updated_at: timestamptz`

Инвариант: `name` уникален внутри project без учета регистра. На уровне БД это фиксируется уникальным индексом `(project_id, name_normalized)`, где `name_normalized = lower(trim(name))`. `color` хранится как hex `#RRGGBB`.

### Issue

Поля:

- `id: uuid`
- `project_id: uuid`
- `title: string`, обязательно, максимум 200 символов
- `description: string?`, markdown
- `status: IssueStatus`
- `priority: IssuePriority`
- `assignee_id: uuid?`
- `reporter_id: uuid`
- `acceptance_criteria: string?`, markdown
- `acceptance_criteria_ai_suggested: bool`
- `created_at: timestamptz`
- `updated_at: timestamptz`
- `closed_at: timestamptz?`

Enums:

- `IssueStatus`: `backlog`, `todo`, `in-progress`, `in-review`, `done`
- `IssuePriority`: `low`, `medium`, `high`, `urgent`

Инварианты:

- issue создается со `status = backlog`, `priority = medium`, `assignee_id = null`;
- `reporter_id` устанавливается из текущего пользователя и не меняется;
- переходы между статусами свободные, включая откаты и перепрыгивания;
- при установке `status = done` backend ставит `closed_at`, если оно еще не задано;
- при выходе из `done` backend очищает `closed_at`;
- labels issue должны принадлежать тому же project.

### Comment

Поля:

- `id: uuid`
- `issue_id: uuid`
- `author_id: uuid`
- `body: string`, markdown
- `created_at: timestamptz`
- `updated_at: timestamptz`

Порядок отображения — `created_at asc`. Редактировать и удалять комментарий может только `author_id`. Удаление в первой версии может быть hard delete; audit не требуется.

## 6. Модель данных PostgreSQL

Основные таблицы:

- `users`
- `projects`
- `labels`
- `issues`
- `issue_labels`
- `comments`
- опционально `ai_triage_suggestions` для трассировки AI-ответов

Рекомендуемые индексы:

- `users(email)` unique
- `projects(slug)` unique
- `projects(owner_id)`
- `labels(project_id, name_normalized)` unique
- `issues(project_id, status, updated_at desc)`
- `issues(project_id, assignee_id, updated_at desc)`
- `issues(project_id, priority, updated_at desc)`
- `issue_labels(issue_id, label_id)` unique
- `issue_labels(label_id, issue_id)`
- `comments(issue_id, created_at asc)`

Для поиска по title/description в первой версии есть два приемлемых варианта:

1. PostgreSQL full text search через generated `tsvector` или индекс GIN по выражению `to_tsvector('simple', coalesce(title,'') || ' ' || coalesce(description,''))`.
2. Упрощенный `ILIKE` поиск для MVP.

Рекомендация: заложить FTS сразу, но оставить API таким, чтобы реализация могла временно использовать `ILIKE`. Язык `simple` безопаснее для смешанных русско-английских issues, чем жесткий английский словарь.

Для AI-триажа полезна таблица `ai_triage_suggestions`:

- `id: uuid`
- `issue_id: uuid`
- `requested_by_id: uuid`
- `model: string?`
- `prompt_version: string`
- `suggested_priority: text`
- `suggested_label_ids: uuid[]` или отдельная join-таблица
- `suggested_acceptance_criteria: text?`
- `raw_response: jsonb?`
- `created_at: timestamptz`

Она не обязательна для работы UI, но полезна для отладки, регрессий промпта и анализа качества. Хранить полный prompt не рекомендуется по умолчанию, чтобы не раздувать БД и не сохранять лишний пользовательский контент сверх необходимого.

## 7. REST API

API версионируется префиксом `/api/v1`. Все write/read endpoints, кроме login/register, требуют JWT Bearer access token.

### Auth

- `POST /api/v1/auth/register`
  - request: `{ email, password, name }`
  - response: `{ user, accessToken }`
- `POST /api/v1/auth/login`
  - request: `{ email, password }`
  - response: `{ user, accessToken }`
- `GET /api/v1/auth/me`
  - response: `{ user }`

Для первой версии достаточно JWT access token, хранимого фронтендом в памяти или session storage. Более безопасная refresh-cookie схема лучше, но добавляет больше инфраструктуры. Если продукт сразу предполагается как публичный интернет-сервис, стоит перейти на `httpOnly Secure SameSite` cookie и CSRF-защиту; для учебного/внутреннего проекта JWT Bearer проще.

### Users

- `GET /api/v1/users?query=` — поиск пользователей для assignee picker.

### Projects

- `GET /api/v1/projects`
- `POST /api/v1/projects`
- `GET /api/v1/projects/{projectSlug}`
- `PATCH /api/v1/projects/{projectSlug}`
- `DELETE /api/v1/projects/{projectSlug}`

### Labels

- `GET /api/v1/projects/{projectSlug}/labels`
- `POST /api/v1/projects/{projectSlug}/labels`
- `PATCH /api/v1/projects/{projectSlug}/labels/{labelId}`
- `DELETE /api/v1/projects/{projectSlug}/labels/{labelId}`

Удаление label снимает ее со всех issues через cascade на `issue_labels`, но не удаляет сами issues.

### Issues

- `GET /api/v1/projects/{projectSlug}/issues`
  - query: `status`, `assigneeId`, `labelIds`, `q`, `includeClosed`, `page`, `pageSize`, `sort`
  - default: `includeClosed=false`, то есть `done` скрыты, если явно не запрошены
- `POST /api/v1/projects/{projectSlug}/issues`
  - quick create request: `{ title }`
  - expanded create request: `{ title, description? }`
  - backend игнорирует входящие `status`, `priority`, `assignee`, `reporter`
- `GET /api/v1/projects/{projectSlug}/issues/{issueId}`
- `PATCH /api/v1/projects/{projectSlug}/issues/{issueId}`
  - request может менять: `title`, `description`, `status`, `priority`, `assigneeId`, `labelIds`, `acceptanceCriteria`, `acceptanceCriteriaAiSuggested`
  - `reporterId`, `createdAt`, `closedAt` напрямую не принимаются
- `DELETE /api/v1/projects/{projectSlug}/issues/{issueId}`

### Comments

- `GET /api/v1/projects/{projectSlug}/issues/{issueId}/comments`
- `POST /api/v1/projects/{projectSlug}/issues/{issueId}/comments`
- `PATCH /api/v1/projects/{projectSlug}/issues/{issueId}/comments/{commentId}`
- `DELETE /api/v1/projects/{projectSlug}/issues/{issueId}/comments/{commentId}`

### AI triage

- `POST /api/v1/projects/{projectSlug}/issues/{issueId}/ai-triage/suggest`
  - request: `{}` или `{ promptMode?: "default" }`
  - response: `{ suggestionId, priority, labels, acceptanceCriteria, promptVersion }`
- `POST /api/v1/projects/{projectSlug}/issues/{issueId}/ai-triage/apply`
  - request: `{ suggestionId?, priority?, labelIds?, acceptanceCriteria?, acceptanceCriteriaAiSuggested? }`
  - response: updated issue

`apply` не обязан применить предложение целиком. Пользователь может отправить только выбранные поля или уже отредактированные значения. Backend валидирует, что labelIds принадлежат проекту, priority входит в enum, issue принадлежит project.

## 8. AI-триаж: поведение и контракт

AI-триаж вызывается вручную пользователем из issue detail. Он читает:

- project name;
- project description;
- список labels проекта: id, name, optionally description if later added;
- issue title;
- issue description;
- текущий priority/status/labels можно передавать как контекст, но модель должна понимать, что это не обязательно финальные значения.

Модель должна вернуть строго структурированный JSON:

```json
{
  "priority": "medium",
  "labelNames": ["bug", "backend"],
  "acceptanceCriteria": "- [ ] ..."
}
```

Backend не должен доверять ответу модели напрямую:

- priority валидируется по enum;
- labels сопоставляются только с существующими labels проекта;
- неизвестные labelNames отбрасываются, а не создаются автоматически;
- если модель вернула невалидный JSON, backend возвращает управляемую ошибку или делает одну повторную попытку с repair prompt;
- acceptance criteria ограничивается по максимальной длине;
- raw response можно сохранить для диагностики, но UI получает только нормализованный результат.

Prompt должен явно запрещать придумывать labels и требовать выбирать только из списка. Пример системного намерения:

```text
You are an assistant helping triage software issues.
Return only valid JSON matching the schema.
Choose priority from: low, medium, high, urgent.
Choose labels only from the provided project labels. Do not invent new labels.
Acceptance criteria must be practical and verifiable.
```

AI provider должен быть спрятан за интерфейсом, например `IAiTriageService` или `ILlmClient`, чтобы в тестах использовать fake provider, а в production подключать OpenAI-compatible API через конфигурацию. Конкретный провайдер не должен протекать в домен или frontend.

Состояние `acceptance_criteria_ai_suggested` означает, что текущее поле acceptance criteria было создано или применено из AI-предложения. Если пользователь вручную редактирует это поле после применения, frontend должен отправлять `acceptanceCriteriaAiSuggested=false`, если пользователь явно не оставляет пометку как AI-suggested. Это сохраняет честность происхождения текста без сложной истории изменений.

## 9. Пользовательские сценарии

### 9.1 Быстрое создание issue

1. Пользователь открывает project.
2. Нажимает `New issue`.
3. Вводит только `title`.
4. Backend создает issue:
   - `status = backlog`
   - `priority = medium`
   - `assignee = null`
   - `reporter = current user`
5. UI возвращает пользователя к списку или открывает issue detail.

Форма quick create должна быть минимальной. Расширенные поля не должны мешать главному сценарию приема входящих.

### 9.2 AI suggest для backlog

1. Пользователь открывает issue без метаданных.
2. Нажимает `AI suggest`.
3. UI показывает loading state и не блокирует просмотр issue.
4. Backend собирает контекст, вызывает AI, валидирует ответ.
5. UI показывает suggestion panel:
   - suggested priority;
   - suggested labels;
   - suggested acceptance criteria;
   - действия `Apply all`, `Apply selected`, `Edit before apply`, `Discard`.
6. Пользователь применяет целиком или частично.

### 9.3 Триаж и назначение

Пользователь фильтрует backlog, выбирает issue, выставляет priority, labels, acceptance criteria, assignee и переводит в `todo`. Все изменения можно сохранить через один `PATCH issue`.

### 9.4 Работа над issue

Исполнитель меняет status `todo -> in-progress`, оставляет комментарии, затем переводит в `in-review`. URL pull request в первой версии разумнее хранить как markdown-ссылку в комментарии. Отдельное поле `pull_request_url` не указано в домене и добавлять его сейчас не стоит: это расширит модель ради одного сценария, который уже покрывается комментариями.

### 9.5 Поиск и фильтры

Список issues поддерживает фильтры по status, assignee, labels, текстовый поиск и пагинацию. Закрытые issues скрыты по умолчанию. Если пользователь выбирает status `done` или включает `includeClosed=true`, закрытые задачи показываются.

### 9.6 Закрытие issue

При переводе issue в `done` backend ставит `closed_at`. При переводе из `done` в любой другой статус backend очищает `closed_at`. UI показывает закрытые задачи отдельно или только по явному фильтру.

## 10. Frontend-дизайн

Основные маршруты:

- `/login`
- `/register`
- `/projects`
- `/projects/new`
- `/projects/:projectSlug`
- `/projects/:projectSlug/issues/:issueId`
- `/projects/:projectSlug/settings/labels`

Рекомендуемые библиотеки:

- React + TypeScript + Vite.
- TanStack Query для server state: списки, детали, мутации, invalidation.
- React Router для маршрутизации.
- React Hook Form + Zod для форм и валидации.
- Небольшая component library на выбор: shadcn/ui, Radix UI primitives или MUI. Для чистого современного интерфейса без тяжелого визуального стиля лучше shadcn/ui/Radix + Tailwind.
- Markdown rendering через безопасный renderer с sanitizer.

Ключевые экраны:

1. **Auth screens** — login/register.
2. **Projects list** — список проектов, создание проекта.
3. **Project issues** — фильтры, поиск, пагинация, quick create issue.
4. **Issue detail** — поля issue, labels, assignee, status, priority, acceptance criteria, comments.
5. **AI triage panel** — отдельная панель в issue detail, показывающая предложение и позволяющая применить выбранные части.
6. **Labels settings** — управление labels проекта.

UX-принципы:

- quick create всегда должен быть самым быстрым путем;
- изменение status/priority/assignee не должно требовать полной формы редактирования;
- AI suggestion визуально отделяется от сохраненных данных;
- пользователь должен явно понимать, что AI-предложение еще не применено;
- ошибки API и AI должны быть видимыми и восстановимыми без потери введенного текста.

## 11. Backend-дизайн

Backend строится на ASP.NET Core 9 Web API и EF Core. Рекомендуемый стиль endpoint-ов — Minimal APIs с feature grouping или controllers. Для этого проекта оба варианта подходят; важнее держать use cases отдельно от HTTP-слоя. Практичный выбор: Minimal APIs + endpoint groups по feature, если команда комфортно работает с ними; иначе controllers не будут ошибкой.

Ключевые application services:

- `AuthService`: register, login, password hashing, JWT issuing.
- `ProjectService`: project CRUD.
- `LabelService`: label CRUD and normalized uniqueness.
- `IssueService`: create, patch, search/filter, status transitions, label validation.
- `CommentService`: create/edit/delete with author ownership check.
- `AiTriageService`: build prompt, call LLM client, validate response, persist suggestion.

Технические решения:

- Password hashing через ASP.NET Core `PasswordHasher<TUser>` или Argon2/bcrypt library. Для простоты и надежной интеграции достаточно `PasswordHasher`.
- JWT signing key и AI API key только через configuration/secrets, не в коде.
- Все даты в UTC через `timestamptz`.
- UUID primary keys.
- Enum в БД можно хранить как text для читаемости и простых миграций.
- Ошибки возвращаются в формате `ProblemDetails`.
- Валидация request DTO на границе API; доменные инварианты дублируются в application/domain слое там, где это критично.

## 12. Безопасность и права доступа

В первой версии security model простая:

- незалогиненный пользователь может только register/login;
- залогиненный пользователь может читать и изменять все проекты и issues;
- comment edit/delete разрешены только автору комментария;
- reporter issue не меняется после создания;
- backend игнорирует или запрещает клиентские попытки изменить системные поля.

Это намеренно простая модель, потому что в домене явно сказано, что понятия участника проекта нет. Добавлять роли сейчас было бы ложной точностью: UI выглядел бы сложнее, API требовал бы лишних checks, а продуктовые правила пока этого не требуют.

Markdown нужно рендерить безопасно. Backend может хранить markdown как текст, но frontend обязан использовать sanitizer при отображении HTML. Любые avatar URL нужно валидировать как URL или хранить как строку без автоматического встраивания небезопасного HTML.

AI prompt injection не является полностью устранимой проблемой, но риск снижается тем, что модель не выполняет действий напрямую. Backend валидирует output, не создает labels автоматически и не меняет issue без явного пользовательского apply.

## 13. Ошибки и edge cases

Общие ошибки:

- `400 Bad Request`: невалидная форма, enum, slug, color, title length.
- `401 Unauthorized`: нет или невалиден token.
- `403 Forbidden`: попытка изменить/удалить чужой comment.
- `404 Not Found`: project/issue/label/comment не найден или не принадлежит expected parent.
- `409 Conflict`: duplicate email, duplicate project slug, duplicate label name in project.
- `502 Bad Gateway` или `503 Service Unavailable`: AI provider недоступен.

Edge cases:

- Issue без description должен нормально отправляться в AI; prompt явно говорит, что description может отсутствовать.
- Если в project нет labels, AI возвращает пустой labels list; UI не должен считать это ошибкой.
- Если AI предложил labels вне project, backend отбрасывает их и может вернуть warning в response.
- Если пользователь удалил label между suggest и apply, apply должен проигнорировать отсутствующий label или вернуть validation error. Рекомендация: вернуть validation error, чтобы пользователь видел, что suggestion устарел.
- Если issue уже `done`, AI suggest все еще можно вызвать, но UI может показывать предупреждение, что задача закрыта.
- Если пользователь редактирует comment после создания, `updated_at` меняется; порядок остается по `created_at`.

## 14. Тестовая стратегия

### Backend unit tests

- Issue creation defaults: backlog, medium, no assignee, reporter=current user.
- Reporter cannot be changed through patch.
- `closed_at` set on transition to done.
- `closed_at` cleared on transition from done.
- Label uniqueness case-insensitive per project.
- Issue labels must belong to same project.
- Comment edit/delete allowed only for author.
- AI response normalization rejects invalid priority and unknown labels.

### Backend integration tests

- Auth register/login/me.
- Projects CRUD with unique slug conflict.
- Labels CRUD with duplicate name conflict.
- Issue list filters: status, assignee, labels, query, includeClosed, pagination.
- Comments lifecycle.
- AI suggest endpoint with fake LLM client.
- AI apply endpoint with partial fields.

### Frontend tests

- Quick create issue form submits only title and renders created issue.
- Issue filters update query and results.
- AI suggestion panel supports apply all, partial apply and discard.
- Comment edit/delete controls visible only for author.
- Closed issues hidden by default and visible with filter.

### Contract tests

OpenAPI should be generated from backend and used as source for frontend types, or at least checked in CI. Strong recommendation: generate TypeScript API types from OpenAPI to avoid drift between backend DTO and frontend client.

## 15. Спецификации по фичам

### Spec: Auth

**Given** новый пользователь  
**When** он регистрируется с уникальным email и валидным password  
**Then** система создает user и возвращает access token.

**Given** существующий пользователь  
**When** он входит с верным email/password  
**Then** система возвращает access token и user profile.

**Given** пользователь вводит занятый email  
**When** он пытается зарегистрироваться  
**Then** система возвращает conflict.

### Spec: Quick issue creation

**Given** аутентифицированный пользователь находится в project  
**When** он создает issue только с title  
**Then** issue создается в backlog, priority medium, без assignee, reporter=current user.

### Spec: Status workflow

**Given** issue в любом статусе кроме done  
**When** пользователь переводит его в done  
**Then** backend устанавливает closed_at.

**Given** issue в done  
**When** пользователь переводит его в любой другой статус  
**Then** backend очищает closed_at.

### Spec: AI triage

**Given** issue и project labels  
**When** пользователь вызывает AI suggest  
**Then** backend возвращает валидированное предложение priority, labels из project labels и acceptance criteria.

**Given** пользователь получил AI suggestion  
**When** он применяет только priority и acceptance criteria  
**Then** labels issue не меняются.

**Given** AI вернул неизвестную label  
**When** backend нормализует ответ  
**Then** неизвестная label не применяется.

### Spec: Comments

**Given** comment создан пользователем A  
**When** пользователь B пытается его изменить или удалить  
**Then** backend возвращает forbidden.

## 16. Минимальная последовательность реализации для будущих планов

Этот документ не является планом работ, но фиксирует логически безопасный порядок, от которого удобно строить будущие планы:

1. Создать backend solution, базовые проекты, конфигурацию, PostgreSQL и EF Core migrations.
2. Реализовать доменные сущности, DbContext, индексы и seed/debug data.
3. Реализовать auth.
4. Реализовать projects и labels.
5. Реализовать issues без AI: create, patch, filters, status rules.
6. Реализовать comments.
7. Создать React SPA: auth flow, project list, issue list, issue detail.
8. Добавить AI provider abstraction и fake provider для тестов.
9. Реализовать AI suggest/apply на backend.
10. Реализовать AI suggestion panel на frontend.
11. Добавить интеграционные и frontend tests вокруг ключевых сценариев.

## 17. Открытые решения для будущей реализации

Эти решения не блокируют дизайн, но должны быть выбраны перед кодированием соответствующих частей:

1. JWT storage strategy: простой Bearer token в session storage или более безопасная cookie-based схема.
2. UI library: shadcn/ui + Tailwind, MUI или собственные минимальные компоненты.
3. Конкретный LLM provider: OpenAI-compatible endpoint, Azure OpenAI или локальная модель. Код должен зависеть от абстракции, а не от провайдера.
4. Точная реализация text search: сразу PostgreSQL FTS или временный `ILIKE`.
5. Нужно ли хранить `ai_triage_suggestions` в первой миграции или добавить после базового AI suggest. Рекомендация: хранить сразу, потому что это сильно упрощает диагностику.

## 18. Definition of Done для первой версии

Первая версия считается завершенной, когда:

- пользователь может зарегистрироваться, войти и получить текущий профиль;
- пользователь может создать project и labels;
- пользователь может создать issue только с title;
- issue получает корректные default values и reporter;
- пользователь может фильтровать и искать issues с пагинацией;
- пользователь может редактировать priority, status, assignee, labels, description и acceptance criteria;
- `closed_at` управляется backend автоматически;
- пользователь может создавать, редактировать и удалять свои comments;
- чужие comments нельзя редактировать или удалять;
- AI suggest возвращает валидированное предложение;
- пользователь может применить AI suggestion целиком или частично;
- frontend ясно отличает AI-предложение от уже сохраненных данных;
- основные backend integration tests и ключевые frontend tests проходят;
- OpenAPI или другой контракт API доступен для frontend и будущих сессий реализации.

