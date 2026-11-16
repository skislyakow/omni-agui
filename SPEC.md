# SPEC.md — Архитектурная спецификация `agui-omni-assistant`

## 1. Предметная область
Умный ИИ-трекер задач и заметок с общим состоянием (Shared State).
- Пользователь может создавать, обновлять, фильтровать и удалять задачи на естественном языке из любого клиента.
- При подключении клиента к существующему `thread_id` бэкенд отправляет текущее состояние списка задач через `STATE_SNAPSHOT`.
- При изменении задач агентом бэкенд вычисляет JSON Patch и отправляет `STATE_DELTA`, чтобы все клиенты обновили интерфейс в реальном времени.
- **Human-in-the-loop (HITL):** Массовое удаление задач (например, «удали все выполненные задачи») требует подтверждения от пользователя перед выполнением.

## 2. Структура монорепозитория
```text
agui-omni-assistant/
├── AGENTS.md                 # Инструкции для Opencode
├── SPEC.md                   # Техническая спецификация
├── PLAN.md                   # Пошаговый план разработки
├── docker-compose.yml        # Redis, Backend, Telegram Bot, Web
├── .env.example              # Шаблон переменных окружения
├── backend/
│   ├── pyproject.toml        # Зависимости, ruff, mypy --strict
│   ├── src/
│   │   ├── main.py           # Точка входа FastAPI + CORS
│   │   ├── config.py         # Настройки (pydantic-settings)
│   │   ├── api/
│   │   │   └── router.py     # POST /agent (SSE-стриминг AG-UI)
│   │   ├── agent/
│   │   │   ├── core.py       # Оркестратор вызова LLM и генерации событий
│   │   │   ├── prompts.py    # Системные промпты ассистента
│   │   │   └── tools.py      # Инструменты (add_task, complete_task, delete_tasks)
│   │   ├── state/
│   │   │   ├── models.py     # Pydantic-схемы Task, TaskListState
│   │   │   └── patch.py      # Генерация STATE_DELTA (jsonpatch)
│   │   └── storage/
│   │       └── redis_repo.py # Сохранение истории и стейта по thread_id
│   └── tests/
├── clients/
│   ├── cli/                  # Консольный клиент (httpx + rich)
│   │   ├── pyproject.toml
│   │   └── main.py
│   ├── telegram_bot/         # Бот на aiogram 3.x
│   │   ├── pyproject.toml
│   │   └── bot.py
│   └── web/                  # React + Vite + Tailwind + @ag-ui/client
│       ├── package.json
│       └── src/
└── .github/workflows/ci.yml  # CI: ruff, mypy, pytest
```

## 3. Контракт событий AG-UI (Backend -> Clients)
Эндпоинт `POST /agent` принимает `ag_ui.core.RunAgentInput` и возвращает `StreamingResponse` с `EventEncoder`.

Обязательный жизненный цикл одного запуска (`run`):
1. `RunStartedEvent(thread_id=..., run_id=...)`
2. `StateSnapshotEvent(snapshot=current_state.model_dump())` (при старте или восстановлении сессии)
3. При вызове инструментов LLM:
   - `ToolCallStartEvent(tool_call_id=..., tool_call_name=...)`
   - `ToolCallArgsEvent(tool_call_id=..., delta=...)`
   - `ToolCallEndEvent(tool_call_id=...)`
   - `ToolCallResultEvent(tool_call_id=..., content=...)`
4. При изменении списка задач:
   - `StateDeltaEvent(delta=json_patch_ops)`
5. При потоковом ответе ассистента:
   - `TextMessageStartEvent(message_id=..., role="assistant")`
   - `TextMessageContentEvent(message_id=..., delta=chunk)`
   - `TextMessageEndEvent(message_id=...)`
6. `RunFinishedEvent(thread_id=..., run_id=...)` (или `RunErrorEvent` при исключении).

## 4. Специфика реализации клиентов
- **Web (`@ag-ui/client`):** Использует `HttpAgent`. Слева — чат со стримингом и индикаторами `TOOL_CALL_*`. Справа — канбан/список задач, реактивно обновляемый по `STATE_SNAPSHOT` и `STATE_DELTA`.
- **Telegram (`aiogram`):** Асинхронно читает SSE-поток через `httpx`. Реализует троттлинг (буферизацию текста и вызов `edit_text` не чаще 1 раза в 0.9 сек) во избежание Telegram FloodWait. При получении `STATE_DELTA` прикладывает актуальный список задач в подвал сообщения или отдельным блоком.
- **CLI (`rich`):** Использует `rich.live.Live` для одновременного рендеринга стриминга ответа и таблицы задач в терминале.
