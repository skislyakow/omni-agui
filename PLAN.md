# PLAN.md — Step-by-Step Implementation Roadmap

Execute stages sequentially. Do not move to the next stage until all acceptance criteria and quality checks (`ruff`, `mypy`, `pytest`) for the current stage pass.

---

## Stage 1: Core Backend & AG-UI Protocol Setup
- **Git Branch:** `feat/stage-1-core-backend`
- **Goal:** Stand up a working FastAPI server emitting standard AG-UI SSE events.
- **Tasks:**
  1. Initialize `backend/` environment using `uv sync` based on `backend/pyproject.toml`.
  2. Create `backend/src/config.py` using `pydantic-settings` to load `.env` variables.
  3. Implement `POST /agent` endpoint in `backend/src/api/routes.py` accepting `RunAgentInput` and returning `StreamingResponse` encoded with `ag_ui.encoder.EventEncoder`.
  4. Connect async LLM streaming (`openai` SDK) to emit `RUN_STARTED`, `TEXT_MESSAGE_START`, `TEXT_MESSAGE_CONTENT`, `TEXT_MESSAGE_END`, and `RUN_FINISHED`.
  5. Write unit tests in `backend/tests/test_sse_stream.py` verifying event sequence and SSE formatting.
- **Acceptance Criteria:** `curl` or a test script sending `RunAgentInput` receives a valid stream of AG-UI events. `ruff`, `mypy`, and `pytest` pass.

---

## Stage 2: Shared State, Redis Persistence & Agent Tools
- **Git Branch:** `feat/stage-2-shared-state`
- **Goal:** Add task management tools, Redis session persistence, and `STATE_SNAPSHOT` / `STATE_DELTA` synchronization.
- **Tasks:**
  1. Define `TaskItem` and `AppState` Pydantic models in `backend/src/state/models.py`.
  2. Implement `RedisStateStore` in `backend/src/storage/redis_store.py` to load/save conversation history and `AppState` by `thread_id`.
  3. Create JSON Patch diff helper (`jsonpatch`) to generate `STATE_DELTA` events whenever `AppState` changes.
  4. Implement LLM function calling (tools): `add_task`, `complete_task`, `delete_task`, `list_tasks`, emitting `TOOL_CALL_START`, `TOOL_CALL_ARGS`, `TOOL_CALL_END`, and `STATE_DELTA`.
  5. Add Human-in-the-loop confirmation flow for `clear_completed_tasks`.
  6. Write pytest unit tests for state transitions and JSON Patch delta generation.
- **Acceptance Criteria:** Multi-turn conversations persist across requests via `thread_id`, and task mutations emit valid `STATE_DELTA` events.

---

## Stage 3: Python CLI Client
- **Git Branch:** `feat/stage-3-cli-client`
- **Goal:** Build a terminal client to interact with the backend and validate shared state visually.
- **Tasks:**
  1. Set up `clients/cli/` with `httpx` and `rich`.
  2. Implement an async SSE event parser capable of handling `TEXT_MESSAGE_*`, `TOOL_CALL_*`, `STATE_SNAPSHOT`, and `STATE_DELTA` (applying JSON patches locally).
  3. Build a split terminal UI using `rich.live.Live`: streaming chat output on the left/top and live task table on the right/bottom.
  4. Support interactive Human-in-the-loop (`y/n`) prompts in the terminal.
- **Acceptance Criteria:** Running the CLI lets the user chat with the agent, watch tokens stream live, and see the task table update immediately on `STATE_DELTA`.

---

## Stage 4: Telegram Bot Client (`aiogram`)
- **Git Branch:** `feat/stage-4-telegram-bot`
- **Goal:** Connect a Telegram bot to the exact same AG-UI backend endpoint.
- **Tasks:**
  1. Set up `clients/telegram_bot/` with `aiogram` 3.x and `httpx`.
  2. Reuse or adapt the async SSE client to consume events from `BACKEND_URL`.
  3. Implement buffered message streaming: send an initial placeholder message and update it via `bot.edit_message_text` throttled at `TELEGRAM_THROTTLE_SECONDS` (`~0.9s`).
  4. Render inline keyboard buttons (`Approve` / `Cancel`) when the agent triggers a Human-in-the-loop confirmation.
  5. Add `/tasks` command and automatic task summary formatting on `STATE_DELTA`.
- **Acceptance Criteria:** Bot streams answers smoothly without hitting Telegram flood limits and updates tasks in the shared `thread_id`.

---

## Stage 5: React Web Dashboard (`@ag-ui/client`)
- **Git Branch:** `feat/stage-5-web-dashboard`
- **Goal:** Build a modern web UI demonstrating real-time state sync across interfaces.
- **Tasks:**
  1. Scaffold `clients/web/` using Vite + React + TypeScript + Tailwind CSS.
  2. Install `@ag-ui/client` and connect `HttpAgent` to `VITE_BACKEND_URL`.
  3. Build a two-column dashboard:
     - Left: Streaming chat interface with tool-call status badges and HITL confirmation modal.
     - Right: Live task board (`Todo` / `Done`) subscribed to `STATE_SNAPSHOT` and `STATE_DELTA`.
  4. Allow configuring `thread_id` in the UI so users can connect to the same session as Telegram/CLI.
- **Acceptance Criteria:** Adding a task via Telegram or CLI updates the React board in real time (on next sync/run) and vice versa.

---

## Stage 6: Dockerization, CI & Polish
- **Git Branch:** `chore/stage-6-docker-ci`
- **Goal:** Make the entire stack reproducible with one command and verify CI pipeline.
- **Tasks:**
  1. Write clean multi-stage `Dockerfile`s for `backend`, `clients/telegram_bot`, and `clients/web`.
  2. Verify `docker compose up --build` starts Redis, Backend, Bot, and Web cleanly.
  3. Ensure `.github/workflows/ci.yml` passes all checks on GitHub Actions.
  4. Update `README.md` with final setup instructions and architecture notes.
- **Acceptance Criteria:** Fresh clone runs cleanly via `docker compose up` using `.env.example` template.
