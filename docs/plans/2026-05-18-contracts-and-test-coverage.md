# API contracts и test coverage — план реализации

Дата: 2026-05-18  
Источник: `docs/designs/2026-05-18-issue-tracker-ai-triage.md`  
Назначение: удержать backend/frontend контракты стабильными и покрыть ключевые сценарии первой версии тестами.

## Область плана

План покрывает OpenAPI, DTO discipline, backend unit/integration tests, frontend tests и финальную проверку Definition of Done. Он выполняется параллельно с backend, frontend и AI-планами, но финальная сверка возможна только после интеграции всех основных фич.

## Порядок реализации

### 1. Контракты API

- [ ] Настроить генерацию OpenAPI из backend как источник truth для REST-контрактов.
- [ ] Убедиться, что API endpoints используют explicit request/response DTO, а не EF entities.
- [ ] Зафиксировать response shape для auth: `{ user, accessToken }` и `{ user }`.
- [ ] Зафиксировать response shape для issue list: items, pagination metadata и примененные filters.
- [ ] Зафиксировать response shape для validation errors через `ProblemDetails`.
- [ ] Зафиксировать response shape для conflicts через `ProblemDetails` с понятными machine-readable details, если это поддерживается выбранной реализацией.
- [ ] Зафиксировать response shape для AI suggest: `suggestionId`, `priority`, `labels`, `acceptanceCriteria`, `promptVersion`.
- [ ] Зафиксировать response shape для AI apply как updated issue.
- [ ] Добавить команду или script для экспорта OpenAPI schema в репозиторий или для проверки schema в CI.
- [ ] Настроить frontend API types/client на основе OpenAPI или временно поддерживать typed DTO вручную до подключения генерации.
- [ ] Добавить проверку, что breaking change OpenAPI не проходит незамеченным в CI или локальном verification script.

### 2. Backend unit tests

- [ ] Покрыть issue creation defaults: `backlog`, `medium`, no assignee, reporter=current user.
- [ ] Покрыть невозможность изменить reporter через patch.
- [ ] Покрыть установку `closed_at` при переходе в `done`.
- [ ] Покрыть очистку `closed_at` при переходе из `done` в любой другой status.
- [ ] Покрыть case-insensitive uniqueness labels внутри project.
- [ ] Покрыть правило, что issue labels должны принадлежать тому же project.
- [ ] Покрыть author-only правило для comment edit/delete.
- [ ] Покрыть AI response normalization для invalid priority.
- [ ] Покрыть AI response normalization для unknown labels.
- [ ] Покрыть acceptance criteria max length при AI normalization.

### 3. Backend integration tests

- [ ] Покрыть `POST /api/v1/auth/register` успешным сценарием.
- [ ] Покрыть `POST /api/v1/auth/register` duplicate email conflict.
- [ ] Покрыть `POST /api/v1/auth/login` успешным сценарием.
- [ ] Покрыть `GET /api/v1/auth/me` с валидным token.
- [ ] Покрыть protected endpoint без token как `401 Unauthorized`.
- [ ] Покрыть projects CRUD, включая duplicate slug conflict.
- [ ] Покрыть labels CRUD, включая duplicate name conflict.
- [ ] Покрыть labels delete и снятие label с issues без удаления issues.
- [ ] Покрыть quick issue create только с title.
- [ ] Покрыть expanded issue create с title и description.
- [ ] Покрыть issue patch для status, priority, assignee, labels, description, acceptance criteria.
- [ ] Покрыть issue list filters: status, assignee, labels, query, includeClosed, pagination.
- [ ] Покрыть default behavior: closed/done issues скрыты без `includeClosed=true`.
- [ ] Покрыть `status=done` или `includeClosed=true`, когда closed issues должны отображаться.
- [ ] Покрыть comments lifecycle: list, create, edit own, delete own.
- [ ] Покрыть forbidden edit/delete чужого comment.
- [ ] Покрыть AI suggest endpoint с fake LLM client.
- [ ] Покрыть AI apply endpoint с partial fields.
- [ ] Покрыть AI apply validation error при label, удаленной между suggest и apply.

### 4. Frontend tests

- [ ] Покрыть register/login happy path с mocked API.
- [ ] Покрыть redirect unauthenticated user на login.
- [ ] Покрыть project create form и duplicate slug error.
- [ ] Покрыть labels create/edit/delete и duplicate name error.
- [ ] Покрыть quick create issue form: submit only title и render created issue.
- [ ] Покрыть issue filters: status, assignee, labels, q, includeClosed и pagination query params.
- [ ] Покрыть closed issues hidden by default и visible with filter/toggle.
- [ ] Покрыть issue detail field updates: status, priority, assignee, labels, acceptance criteria.
- [ ] Покрыть ручное редактирование acceptance criteria с `acceptanceCriteriaAiSuggested=false`.
- [ ] Покрыть comment edit/delete controls visible only for author.
- [ ] Покрыть AI suggestion panel: apply all.
- [ ] Покрыть AI suggestion panel: partial apply без изменения labels.
- [ ] Покрыть AI suggestion panel: edit before apply.
- [ ] Покрыть AI suggestion panel: discard без backend mutation.
- [ ] Покрыть recoverable API/AI error states без потери введенного текста.

### 5. Manual acceptance checklist

- [ ] Зарегистрировать нового пользователя и получить access token.
- [ ] Войти существующим пользователем и открыть `/projects`.
- [ ] Создать project с валидным slug.
- [ ] Создать labels внутри project.
- [ ] Создать issue только с title и проверить defaults.
- [ ] Найти issue через search по title/description.
- [ ] Отфильтровать issues по status, assignee и labels.
- [ ] Назначить assignee из любого пользователя системы.
- [ ] Перевести issue в `done` и проверить появление `closed_at`.
- [ ] Убедиться, что closed issue скрыт по умолчанию.
- [ ] Включить отображение closed issues и увидеть закрытый issue.
- [ ] Вернуть issue из `done` и проверить очистку `closed_at`.
- [ ] Создать comment, отредактировать свой comment и удалить свой comment.
- [ ] Проверить, что чужой comment нельзя изменить или удалить.
- [ ] Вызвать AI suggest для issue без description.
- [ ] Применить только priority и acceptance criteria из AI suggestion.
- [ ] Убедиться, что labels не изменились при partial apply без labelIds.
- [ ] Вызвать AI suggest в project без labels и получить пустой labels list без ошибки.
- [ ] Проверить, что неизвестные labels из fake/controlled AI response не применяются.
- [ ] Проверить, что OpenAPI schema доступна и frontend client/types соответствуют текущему backend.

## Критерии готовности quality gate

- [ ] Backend unit tests покрывают критические доменные инварианты.
- [ ] Backend integration tests покрывают auth, projects, labels, issues, comments и AI endpoints.
- [ ] Frontend tests покрывают ключевые пользовательские сценарии и AI panel behavior.
- [ ] API contracts доступны и не требуют ручного угадывания DTO на frontend.
- [ ] Все пункты manual acceptance checklist пройдены перед объявлением первой версии готовой.
