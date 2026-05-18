# Frontend SPA и issue UI — план реализации

Дата: 2026-05-18  
Источник: `docs/designs/2026-05-18-issue-tracker-ai-triage.md`  
Назначение: реализовать React SPA для базовых сценариев issue tracker и подготовить место для AI suggestion panel.

## Область плана

План покрывает React + TypeScript + Vite приложение, routing, auth flow, projects, labels, issue list, issue detail, quick create, filters, comments и базовые frontend tests. AI suggestion panel вынесена в отдельный план, но issue detail должен быть подготовлен к интеграции.

## Порядок реализации

### 1. Bootstrap frontend-приложения

- [ ] Создать директорию `frontend/` и Vite React TypeScript приложение.
- [ ] Настроить TypeScript, lint/test scripts и базовую структуру `src/app`, `src/features`, `src/shared`.
- [ ] Подключить React Router для маршрутов `/login`, `/register`, `/projects`, `/projects/new`, `/projects/:projectSlug`, `/projects/:projectSlug/issues/:issueId`, `/projects/:projectSlug/settings/labels`.
- [ ] Подключить TanStack Query для server state, mutations и cache invalidation.
- [ ] Подключить React Hook Form + Zod для форм и client-side validation.
- [ ] Настроить Tailwind и базовый набор UI primitives в стиле shadcn/Radix без тяжелой визуальной кастомизации.
- [ ] Настроить безопасный markdown renderer с sanitizer для description, acceptance criteria и comments.
- [ ] Создать typed API layer на основе OpenAPI или вручную синхронизированных DTO до появления генерации.
- [ ] Реализовать единый API error handling для `400`, `401`, `403`, `404`, `409`, `502/503`.

### 2. Auth flow

- [ ] Реализовать `AuthProvider` с хранением JWT access token в `sessionStorage`.
- [ ] Реализовать API interceptor/helper для добавления Bearer token к защищенным requests.
- [ ] Реализовать logout с очисткой token и query cache.
- [ ] Реализовать protected route wrapper для всех страниц, кроме login/register.
- [ ] Реализовать `/register` с полями email, password, name и отображением duplicate email conflict.
- [ ] Реализовать `/login` с email/password и обработкой неверных credentials.
- [ ] Реализовать загрузку текущего пользователя через `/api/v1/auth/me` при старте authenticated session.
- [ ] Реализовать redirect на `/projects` после успешного login/register.

### 3. Projects UI

- [ ] Реализовать `/projects` со списком проектов, empty state и кнопкой создания.
- [ ] Реализовать `/projects/new` с формой name, slug, description.
- [ ] Валидировать slug на клиенте тем же правилом, что backend: lowercase latin, digits, hyphen.
- [ ] Отображать `409 Conflict` при duplicate slug без потери введенных данных.
- [ ] Реализовать переход в `/projects/:projectSlug` после создания project.
- [ ] Реализовать базовый project header с name, description и ссылкой на labels settings.

### 4. Labels settings

- [ ] Реализовать `/projects/:projectSlug/settings/labels` со списком labels проекта.
- [ ] Реализовать создание label с name и color.
- [ ] Валидировать color как hex `#RRGGBB` на клиенте.
- [ ] Реализовать редактирование label name/color.
- [ ] Реализовать удаление label с понятным подтверждением, что label будет снята с issues, но issues не удалятся.
- [ ] Отображать duplicate label conflict без потери введенных данных.
- [ ] Инвалидировать queries issue list/detail после изменения labels, если это влияет на отображение.

### 5. Project issue list и quick create

- [ ] Реализовать `/projects/:projectSlug` как issue list screen.
- [ ] Реализовать quick create form, где обязательным является только `title`.
- [ ] После quick create обновлять список и либо оставаться на списке, либо открывать issue detail по выбранному UX-решению реализации.
- [ ] Отображать созданный issue с defaults `backlog`, `medium`, no assignee и current reporter.
- [ ] Реализовать фильтр status, включая поведение для `done`.
- [ ] Реализовать toggle `includeClosed`, выключенный по умолчанию.
- [ ] Реализовать фильтр assignee с user search.
- [ ] Реализовать фильтр labels с labels проекта.
- [ ] Реализовать текстовый поиск `q` по title/description.
- [ ] Реализовать pagination с сохранением query params в URL.
- [ ] Реализовать loading, empty и error states без сброса текущих фильтров.

### 6. Issue detail

- [ ] Реализовать `/projects/:projectSlug/issues/:issueId` с загрузкой issue, project labels и assignee options.
- [ ] Реализовать редактирование `title` и `description`.
- [ ] Реализовать inline-изменение `status`.
- [ ] Реализовать inline-изменение `priority`.
- [ ] Реализовать assignee picker с возможностью очистить assignee.
- [ ] Реализовать labels picker только из labels текущего project.
- [ ] Реализовать редактирование `acceptanceCriteria`.
- [ ] При ручном изменении acceptance criteria отправлять `acceptanceCriteriaAiSuggested=false`.
- [ ] Отображать `closed_at`, когда issue находится в `done`.
- [ ] Показывать предупреждение или метку для closed issue, не блокируя просмотр.
- [ ] Объединить частые изменения в понятные PATCH mutations с optimistic или conservative cache update, выбранным на этапе реализации.
- [ ] Не показывать AI suggestion как сохраненные данные до явного apply.
- [ ] Зарезервировать место в issue detail для AI suggestion panel из отдельного плана.

### 7. Comments UI

- [ ] Реализовать список comments в issue detail с сортировкой по `created_at asc`.
- [ ] Реализовать форму добавления comment с markdown input.
- [ ] Реализовать безопасный markdown preview/render для comment body.
- [ ] Показывать edit/delete controls только для comments текущего пользователя.
- [ ] Реализовать редактирование своего comment с сохранением введенного текста при ошибке API.
- [ ] Реализовать удаление своего comment с подтверждением.
- [ ] Не показывать edit/delete controls для чужих comments.
- [ ] Корректно обработать `403 Forbidden`, если backend отклонил операцию из-за устаревшего UI state.

### 8. Frontend verification

- [ ] Добавить frontend tests для quick create issue form: отправка только title и отображение созданного issue.
- [ ] Добавить frontend tests для issue filters: изменение query и обновление списка.
- [ ] Добавить frontend tests для скрытия closed issues по умолчанию и отображения через filter/toggle.
- [ ] Добавить frontend tests для comment edit/delete controls только у автора.
- [ ] Проверить ручной flow: login/register → projects → labels → quick issue → filters → issue detail → comments.
- [ ] Проверить, что ошибки API видимы и не уничтожают введенный пользователем текст.

## Критерии готовности frontend core

- [ ] SPA запускается локально и подключается к backend API через конфигурацию base URL.
- [ ] Auth flow сохраняет session только в `sessionStorage` и корректно очищает ее при logout.
- [ ] Пользователь может создать project и labels через UI.
- [ ] Пользователь может быстро создать issue только по title.
- [ ] Issue list поддерживает status, assignee, labels, search, includeClosed и pagination.
- [ ] Issue detail позволяет менять основные поля issue без полной формы редактирования.
- [ ] Comments UI соблюдает авторские controls и корректно обрабатывает forbidden.
- [ ] Markdown отображается через sanitizer.
