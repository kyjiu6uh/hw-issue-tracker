# AI triage integration — план реализации

Дата: 2026-05-18  
Источник: `docs/designs/2026-05-18-issue-tracker-ai-triage.md`  
Назначение: реализовать backend и frontend поток AI suggest/apply так, чтобы AI предлагал triage, но не принимал финальных решений за пользователя.

## Область плана

План покрывает AI provider abstraction, prompt builder, output validation, persistence в `ai_triage_suggestions`, REST endpoints `suggest`/`apply`, frontend suggestion panel и тестирование AI-сценариев. Базовые projects, labels, issues и issue detail должны быть готовы до начала большей части этого плана.

## Порядок реализации

### 1. Backend AI abstraction

- [ ] Создать application-level контракт для AI-триажа, например `IAiTriageService`, который не зависит от конкретного LLM provider.
- [ ] Создать infrastructure-level контракт `ILlmClient` или аналогичный adapter для вызова OpenAI-compatible API.
- [ ] Создать fake LLM client для integration tests и локальной разработки без внешнего AI provider.
- [ ] Добавить конфигурацию AI provider через environment variables/user secrets: endpoint, api key, model, timeout, prompt version.
- [ ] Убедиться, что AI API key не попадает в appsettings, logs, OpenAPI responses или frontend bundle.
- [ ] Настроить graceful error mapping: provider unavailable → `502 Bad Gateway` или `503 Service Unavailable`.

### 2. Prompt builder и контекст

- [ ] Создать prompt builder, который принимает project name, project description, labels проекта, issue title, issue description, current priority/status/labels.
- [ ] Явно указать в prompt, что description может отсутствовать.
- [ ] Явно запретить модели придумывать labels вне предоставленного списка.
- [ ] Явно потребовать JSON-only response со схемой `priority`, `labelNames`, `acceptanceCriteria`.
- [ ] Ограничить prompt только необходимым контекстом, не включая лишние пользовательские данные.
- [ ] Версионировать prompt через `prompt_version` и сохранять эту версию вместе с suggestion.

### 3. Output parsing и normalization

- [ ] Реализовать парсинг JSON response от модели в промежуточный DTO.
- [ ] Валидировать `priority` строго по enum `low`, `medium`, `high`, `urgent`.
- [ ] Сопоставлять `labelNames` только с существующими labels текущего project.
- [ ] Отбрасывать неизвестные labelNames и возвращать warning в нормализованном response, если контракт response на этапе реализации это поддерживает.
- [ ] Не создавать новые labels на основании AI output.
- [ ] Ограничить длину `acceptanceCriteria` и возвращать validation/provider error при неприемлемом результате.
- [ ] Реализовать одну управляемую retry/repair попытку при невалидном JSON, если это не усложнит контракт; иначе возвращать понятную ошибку без частичного применения.
- [ ] Убедиться, что raw response не отдается frontend напрямую.

### 4. Persistence suggestion history

- [ ] Записывать AI suggestion в `ai_triage_suggestions` после успешной normalization.
- [ ] Сохранять `issue_id`, `requested_by_id`, `model`, `prompt_version`, `suggested_priority`, `suggested_label_ids`, `suggested_acceptance_criteria`, `raw_response`, `created_at`.
- [ ] Не сохранять полный prompt по умолчанию.
- [ ] Связать `suggested_label_ids` через выбранную модель хранения из миграции: `uuid[]` или join-таблица.
- [ ] Возвращать frontend только нормализованные поля: `suggestionId`, `priority`, `labels`, `acceptanceCriteria`, `promptVersion`.

### 5. Backend endpoints

- [ ] Реализовать `POST /api/v1/projects/{projectSlug}/issues/{issueId}/ai-triage/suggest`.
- [ ] Валидировать, что project существует и issue принадлежит project.
- [ ] Загружать labels проекта и передавать их в AI context.
- [ ] Корректно обрабатывать project без labels: возвращать пустой labels list без ошибки.
- [ ] Разрешить suggest для issue в `done`, но возвращать информацию, достаточную UI для предупреждения, если выбран такой UX.
- [ ] Реализовать `POST /api/v1/projects/{projectSlug}/issues/{issueId}/ai-triage/apply`.
- [ ] Разрешить partial apply: `priority`, `labelIds`, `acceptanceCriteria`, `acceptanceCriteriaAiSuggested` независимо друг от друга.
- [ ] Валидировать, что `priority`, если передан, входит в enum.
- [ ] Валидировать, что `labelIds`, если переданы, принадлежат project.
- [ ] Если label удалили между suggest и apply, возвращать validation error, чтобы пользователь видел устаревшее предложение.
- [ ] Если `acceptanceCriteria` применены из suggestion, устанавливать `acceptanceCriteriaAiSuggested=true`, если request явно не передал другое допустимое значение.
- [ ] Возвращать updated issue после apply.

### 6. Frontend AI suggestion panel

- [ ] Добавить кнопку `AI suggest` в issue detail.
- [ ] Показывать loading state для AI suggest, не блокируя просмотр issue.
- [ ] Показывать recoverable error state для provider/API ошибок без потери текущих edits в issue detail.
- [ ] Визуально отделить AI suggestion от сохраненных полей issue.
- [ ] Отображать suggested priority, suggested labels и suggested acceptance criteria.
- [ ] Показывать предупреждение, если AI suggest вызван для issue в `done`.
- [ ] Реализовать `Apply all`, который отправляет priority, labelIds и acceptanceCriteria.
- [ ] Реализовать `Apply selected`, который позволяет выбрать отдельные поля для применения.
- [ ] Реализовать `Edit before apply` для acceptance criteria перед отправкой.
- [ ] Реализовать `Discard`, который очищает panel локально без backend mutation.
- [ ] После успешного apply инвалидировать issue detail и issue list queries.
- [ ] После ручного редактирования acceptance criteria вне AI flow отправлять `acceptanceCriteriaAiSuggested=false`.
- [ ] Не показывать unknown/discarded labels как применимые варианты.

### 7. AI tests и verification

- [ ] Добавить unit tests для prompt builder: labels перечисляются явно, description может отсутствовать, JSON-only instruction присутствует.
- [ ] Добавить unit tests для normalization: invalid priority отклоняется.
- [ ] Добавить unit tests для normalization: unknown labels отбрасываются и не применяются.
- [ ] Добавить unit tests для normalization: project без labels возвращает пустой labels list.
- [ ] Добавить unit tests для acceptance criteria max length.
- [ ] Добавить integration tests для suggest endpoint с fake LLM client.
- [ ] Добавить integration tests для apply endpoint с partial fields.
- [ ] Добавить integration tests для stale label между suggest и apply.
- [ ] Добавить frontend tests для AI suggestion panel: apply all, partial apply, edit before apply и discard.
- [ ] Проверить вручную flow: issue без description → AI suggest → partial apply priority + acceptance criteria → labels не меняются.
- [ ] Проверить вручную flow: AI возвращает unknown label → backend не применяет ее.

## Критерии готовности AI triage

- [ ] AI provider скрыт за backend abstraction и не протекает в domain/frontend.
- [ ] Suggest endpoint возвращает только валидированный, нормализованный result.
- [ ] Apply endpoint не требует применять suggestion целиком и поддерживает partial apply.
- [ ] Backend никогда не создает labels автоматически из AI output.
- [ ] Пользователь явно контролирует применение каждого AI-предложения.
- [ ] Suggestion history сохраняется достаточно для диагностики prompt/model regressions.
- [ ] UI ясно показывает разницу между предложением AI и сохраненными данными issue.
