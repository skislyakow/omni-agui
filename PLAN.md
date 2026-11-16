# PLAN.md — Пошаговый план разработки для Opencode

Выполняй этапы последовательно. Не переходи к следующему этапу, пока текущий не протестирован и не проходит проверки `ruff` и `mypy --strict`.

## Этап 1: Инициализация монорепозитория и ядра Backend (MVP)
- [ ] Создать структуру директорий согласно `SPEC.md`.
- [ ] Инициализировать `backend/` через `uv init` (Python 3.12+).
- [ ] Добавить зависимости: `fastapi`, `uvicorn`, `ag-ui-protocol`, `pydantic-settings`, `openai`, `jsonpatch`, `redis`.
- [ ] Добавить dev-зависимости: `ruff`, `mypy`, `pytest`, `pytest-asyncio`, `httpx`.
- [ ] Настроить `pyproject.toml` для `ruff` и `mypy` (`strict = true`).
- [ ] Реализовать `config.py` и `main.py` с базовым эндпоинтом `POST /agent`, который принимает `RunAgentInput` и возвращает тестовый стрим событий (`RUN_STARTED` -> `TEXT_MESSAGE_*` -> `RUN_FINISHED`).
- [ ] Написать базовый unit-тест в `backend/tests/test_stream.py` с использованием `httpx.AsyncClient` и `ASGITransport`.

## Этап 2: Модели состояния, Redis и инструменты LLM
- [ ] Создать Pydantic-модели в `backend/src/state/models.py`: `Task` (`id`, `title`, `status`, `priority`, `created_at`) и `TaskListState` (`tasks: list[Task]`, `pending_confirmation: str | None`).
- [ ] Реализовать `backend/src/state/patch.py` для вычисления `STATE_DELTA` через `jsonpatch.make_patch`.
- [ ] Реализовать `backend/src/storage/redis_repo.py` для сохранения и загрузки `TaskListState` и истории сообщений по `thread_id` (с фоллбэком на in-memory словарь для локальных тестов без Redis).
- [ ] Реализовать инструменты агента в `backend/src/agent/tools.py`: `add_task`, `update_task_status`, `request_bulk_delete` (HITL), `confirm_bulk_delete`.
- [ ] Связать `AsyncOpenAI` (с поддержкой function calling / tool calls и streaming) в `backend/src/agent/core.py` с генерацией событий AG-UI.

## Этап 3: CLI-клиент на Python (`rich` + `httpx`)
- [ ] Инициализировать `clients/cli/` через `uv init`.
- [ ] Написать асинхронный парсер SSE-событий от `POST /agent`.
- [ ] Реализовать консольный интерфейс на `rich`:
  - Потоковый вывод ответов ассистента.
  - Отображение статусов вызова инструментов.
  - Отрисовка актуальной таблицы задач при получении `STATE_SNAPSHOT` и применении `STATE_DELTA`.
  - Поддержка подтверждения HITL-действий через ввод `y/n`.

## Этап 4: Telegram-бот (`aiogram` 3.x)
- [ ] Инициализировать `clients/telegram_bot/` через `uv init` (`aiogram`, `httpx`, `jsonpatch`, `pydantic-settings`).
- [ ] Использовать `chat_id` пользователя в качестве `thread_id` для синхронизации сессии с другими клиентами.
- [ ] Реализовать потоковую отправку ответа с буферизацией `TEXT_MESSAGE_CONTENT` и редактированием сообщения с интервалом 0.9 сек.
- [ ] При получении запроса на подтверждение (HITL) отображать `InlineKeyboardMarkup` с кнопками «Подтвердить» / «Отмена».
- [ ] Добавить команду `/tasks` для вывода текущего состояния задач.

## Этап 5: Web-интерфейс (React + TypeScript + `@ag-ui/client`)
- [ ] Инициализировать проект в `clients/web/` через Vite (`react-ts`) и настроить Tailwind CSS.
- [ ] Установить `@ag-ui/client` и `fast-json-patch`.
- [ ] Подключить `HttpAgent` к эндпоинту бэкенда, передавая общий `thread_id` (с возможностью указать свой ID треда в UI для демонстрации синхронизации с Telegram/CLI).
- [ ] Реализовать двухколоночный интерфейс:
  - Слева: чат со стримингом сообщений, плашками вызова инструментов и кнопками подтверждения HITL.
  - Справа: интерактивная доска задач, обновляемая в реальном времени по `STATE_SNAPSHOT` и `STATE_DELTA`.

## Этап 6: Инфраструктура, CI и документация
- [ ] Написать `Dockerfile` для `backend`, `telegram_bot` и `web`.
- [ ] Собрать `docker-compose.yml` (включая сервис `redis:7-alpine`).
- [ ] Настроить `.github/workflows/ci.yml` для автоматического прогона `ruff`, `mypy` и `pytest`.
- [ ] Оформить подробный `README.md` со схемой архитектуры и инструкцией по запуску.
