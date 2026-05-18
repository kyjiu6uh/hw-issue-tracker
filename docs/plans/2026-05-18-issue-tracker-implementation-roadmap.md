# Issue Tracker с AI-автотриажем — общий roadmap реализации

Дата: 2026-05-18  
Источник: `docs/designs/2026-05-18-issue-tracker-ai-triage.md`  
Назначение: верхнеуровневый порядок реализации первой версии и координация отдельных планов.

## Зафиксированные решения для первой реализации

| Решение | Выбор |
| --- | --- |
| Архитектура | Модульный монолит: ASP.NET Core Web API + PostgreSQL + отдельная React SPA |
| Backend-структура | `Api`, `Domain`, `Application`, `Infrastructure`, `tests` |
| API | REST с префиксом `/api/v1`, OpenAPI как контракт для frontend |
| Auth | Email/password, JWT Bearer access token для MVP |
| JWT storage на frontend | `sessionStorage` для первой версии, без refresh-cookie схемы |
| UI stack | React + TypeScript + Vite, TanStack Query, React Router, React Hook Form + Zod, Tailwind + shadcn/Radix-подход |
| Search | Реализовать PostgreSQL FTS с API, допускающим последующую замену реализации |
| AI provider | Абстракция над OpenAI-compatible provider + fake provider для тестов |
| AI suggestions storage | Создать `ai_triage_suggestions` с первой миграцией |

## Последовательность реализации

- [ ] Создать backend solution и базовую структуру проектов согласно design-документу.
- [ ] Подключить конфигурацию, PostgreSQL, EF Core, миграции, health/config sanity checks и локальные настройки без секретов в коде.
- [ ] Реализовать доменные сущности, enum, DbContext, связи, ограничения и индексы, включая `ai_triage_suggestions`.
- [ ] Добавить базовые seed/debug данные для локальной разработки без зависимости от AI provider.
- [ ] Реализовать auth: register, login, `/me`, password hashing, JWT issuing и защиту endpoints.
- [ ] Реализовать users search для assignee picker.
- [ ] Реализовать projects CRUD и labels CRUD с проверками slug, color и case-insensitive uniqueness labels внутри project.
- [ ] Реализовать issues без AI: quick create, detail, patch, delete, labels assignment, status transitions, `closed_at`, filters, pagination и search.
- [ ] Реализовать comments lifecycle с ограничением edit/delete только для автора.
- [ ] Сгенерировать и зафиксировать OpenAPI-контракт, затем использовать его как основу frontend API-типа или клиента.
- [ ] Создать React SPA и инфраструктуру frontend-приложения: routing, auth provider, API client, layout, базовые UI primitives.
- [ ] Реализовать auth screens, projects list/create, project issue list, filters, quick create и issue detail.
- [ ] Реализовать labels settings, assignee picker, inline-редактирование issue fields и comments UI.
- [ ] Реализовать backend AI triage: provider abstraction, prompt builder, output parser/normalizer, persistence, suggest/apply endpoints.
- [ ] Реализовать frontend AI suggestion panel с apply all, apply selected, edit before apply и discard.
- [ ] Добавить backend unit tests для доменных инвариантов и нормализации AI-ответа.
- [ ] Добавить backend integration tests для auth, projects, labels, issues, comments и AI endpoints с fake provider.
- [ ] Добавить frontend tests для quick create, filters, AI panel, comments ownership controls и скрытия closed issues по умолчанию.
- [ ] Проверить end-to-end happy path вручную: register → project → labels → issue → AI suggest/apply → comments → close/reopen.
- [ ] Обновить README инструкциями локального запуска backend, frontend, PostgreSQL и настройками AI provider.

## Зависимости между подробными планами

- [ ] Выполнить `2026-05-18-backend-foundation-and-core-api.md` до полноценной frontend-интеграции.
- [ ] Начать `2026-05-18-frontend-spa-and-issue-ui.md` после появления стабильных auth/projects/issues контрактов или OpenAPI draft.
- [ ] Начать `2026-05-18-ai-triage-integration.md` после готовности projects, labels, issues и базового issue detail UI.
- [ ] Вести `2026-05-18-contracts-and-test-coverage.md` параллельно, но закрывать финальные пункты после завершения backend, frontend и AI-потоков.

## Definition of Done первой версии

- [ ] Пользователь может зарегистрироваться, войти и получить текущий профиль.
- [ ] Пользователь может создать project и labels.
- [ ] Пользователь может создать issue только с title.
- [ ] Issue получает `status=backlog`, `priority=medium`, `assignee=null`, `reporter=current user`.
- [ ] Пользователь может фильтровать и искать issues с пагинацией.
- [ ] Пользователь может редактировать priority, status, assignee, labels, description и acceptance criteria.
- [ ] Backend автоматически управляет `closed_at` при входе в `done` и выходе из `done`.
- [ ] Пользователь может создавать, редактировать и удалять свои comments.
- [ ] Чужие comments нельзя редактировать или удалять.
- [ ] AI suggest возвращает валидированное предложение priority, labels и acceptance criteria.
- [ ] Пользователь может применить AI suggestion целиком или частично.
- [ ] Frontend ясно отделяет AI-предложение от уже сохраненных данных.
- [ ] Основные backend integration tests и ключевые frontend tests проходят.
- [ ] OpenAPI или другой API-контракт доступен frontend и будущим сессиям реализации.
