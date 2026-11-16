# AGENTS.md — System Instructions for AI Coding Agents

## Project Context
**Project:** `omni-agui` — Multi-client AI Task & Note Assistant powered by the **AG-UI (Agent-User Interaction)** protocol.
**Goal:** One Python FastAPI backend streaming standardized AG-UI events to three independent clients: React Web App, Telegram Bot (`aiogram`), and Python CLI (`rich`).

## Tech Stack & Tooling
- **Language:** Python 3.12+ (Backend, Bot, CLI), TypeScript (Web Client)
- **Package Manager:** `uv` (strictly use `uv add`, `uv run`, `uv sync` — never raw `pip`)
- **Backend Framework:** FastAPI, Uvicorn, Pydantic v2, `ag-ui-protocol`
- **State & Storage:** Redis (`redis.asyncio`), `jsonpatch` (RFC 6902)
- **LLM Client:** Official `openai` async client (configured via `LLM_BASE_URL` and `LLM_API_KEY`)
- **Telegram Client:** `aiogram` 3.x + `httpx`
- **CLI Client:** `httpx` + `rich`
- **Web Client:** React 18+, Vite, Tailwind CSS, `@ag-ui/client`
- **Code Quality:** `ruff` (linter & formatter), `mypy --strict` (static type checking), `pytest` + `pytest-asyncio`

## Architectural Rules
1. **Protocol Isolation:** The `backend/` must have ZERO knowledge of which client (Web, Telegram, CLI) is connected. Communicate exclusively via standard AG-UI events (`RUN_STARTED`, `TEXT_MESSAGE_*`, `TOOL_CALL_*`, `STATE_SNAPSHOT`, `STATE_DELTA`, `RUN_FINISHED`, `RUN_ERROR`).
2. **State Synchronization:** All task list mutations must update the Pydantic state model in Redis and emit a `STATE_DELTA` event containing valid JSON Patch operations. Emit `STATE_SNAPSHOT` at the start of a run or upon client request.
3. **Human-in-the-loop (HITL):** Destructive actions (e.g., clearing all completed tasks) must pause execution via an interrupt/tool confirmation flow and resume only after explicit user approval on the same `thread_id`.
4. **Telegram Rate Limits:** The Telegram client MUST buffer incoming `TEXT_MESSAGE_CONTENT` deltas and throttle `edit_text` API calls (minimum `0.8s` interval) to avoid Telegram flood control limits.
5. **Async First:** All I/O operations (Redis, HTTP SSE streams, LLM calls) must be strictly asynchronous.

## Coding Standards & Safety
- **Strict Typing:** Every Python function, method, and module must pass `mypy --strict`. Avoid `Any` unless strictly required by third-party untyped libraries.
- **Validation:** Use `pydantic.BaseModel` and `pydantic_settings.BaseSettings` for all payloads and configuration.
- **No Hardcoded Secrets:** Never hardcode API keys, tokens, or URLs. Always load them from environment variables / `.env`.
- **Verification Gate:** Before marking any task or stage complete, run:
  ```bash
  uv run ruff check .
  uv run ruff format --check .
  uv run mypy src
  uv run pytest
  ```

## Git Workflow & Branching
- **Never commit directly to `main`:** Before starting any stage from `PLAN.md`, check the current branch (`git branch --show-current`). If on `main`, create and checkout the corresponding feature branch specified in `PLAN.md` (e.g., `git checkout -b feat/stage-1-core-backend`).
- **Atomic Commits:** Commit logical chunks of work using Conventional Commits format (`feat:`, `fix:`, `test:`, `docs:`, `chore:`).
- **Pre-Commit Verification:** Only commit code after all linters (`ruff`), type checks (`mypy`), and tests (`pytest`) pass without errors.
- **Stage Completion:** At the end of a stage, summarize all changes made and remind the user to push the branch (`git push -u origin <branch-name>`) and open a Pull Request to trigger GitHub Actions CI.
