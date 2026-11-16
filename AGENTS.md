# AGENTS.md — Инструкции для AI-агента (Opencode)

## О проекте
**Название:** `agui-omni-assistant` (Мульти-клиентский ИИ-ассистент на протоколе AG-UI).
**Цель:** Продемонстрировать единый Python-бэкенд на базе `FastAPI` и `ag-ui-protocol`, который общается по стандартизированному потоку событий (SSE) с тремя независимыми клиентами: Web (React), Telegram (`aiogram`) и CLI (`rich`).

## Стек и стандарты кода
- **Пакетный менеджер:** `uv` (строго использовать `uv add`, `uv run`, `uv sync`).
- **Python:** 3.12+
- **Бэкенд:** FastAPI, `ag-ui-protocol`, Pydantic v2, Redis (`redis.asyncio`), `openai` (или совместимый AsyncOpenAI клиент), `jsonpatch`.
- **Качество кода:**
  - `ruff` для линтинга и форматирования (длина строки 100 символов).
  - `mypy --strict` — обязательная строгая типизация для всего Python-кода.
  - `pytest` + `pytest-asyncio` для тестирования генерации событий и стейта.
- **Клиенты:**
  - `clients/cli`: Python + `httpx` + `rich`.
  - `clients/telegram_bot`: Python + `aiogram` 3.x + `httpx`.
  - `clients/web`: React + TypeScript + Vite + Tailwind CSS + `@ag-ui/client`.

## Правила работы над репозиторием
1. **Изоляция протокола:** Логика агента в `backend/` не должна знать, какой клиент к ней подключен. Взаимодействие идет строго через модели `RunAgentInput` и события `ag_ui.core`.
2. **Асинхронность:** Весь I/O код (LLM, Redis, HTTP-клиенты, Telegram-бот) должен быть полностью асинхронным (`async`/`await`).
3. **Пошаговая разработка:** Выполняй задачи строго по этапам из `PLAN.md`. После каждого этапа проверяй код через `uv run ruff check .` и `uv run mypy .`.
4. **Безопасность:** Никогда не хардкодь API-ключи. Используй `pydantic-settings` и `.env.example`.
