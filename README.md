# omni-agui

**One AI Agent, Three Interfaces: Web, Telegram & CLI powered by the AG-UI Protocol.**

`omni-agui` is a reference full-stack architecture demonstrating the core value of the **Agent-User Interaction Protocol (AG-UI)**: decoupling AI agent logic from the user interface. A single Python backend streams standardized, typed events over Server-Sent Events (SSE) to simultaneously power a React web dashboard, a Telegram bot, and a terminal CLI client.

---

## Architecture

```text
+-------------------------------------------------------------------+
|                        Clients (UI Layer)                         |
|                                                                   |
|  +------------------+   +------------------+   +---------------+  |
|  |   Web Dashboard  |   |   Telegram Bot   |   |  CLI Client   |  |
|  | (React + Vite)   |   |  (aiogram 3.x)   |   | (httpx+rich)  |  |
|  +--------+---------+   +--------+---------+   +-------+-------+  |
+-----------|----------------------|---------------------|----------+
            |                      |                     |
            +----------------------+---------------------+
                                   |
                   Bi-directional AG-UI Event Stream
               (SSE: TEXT_MESSAGE_*, TOOL_CALL_*, STATE_*)
                                   |
+----------------------------------v--------------------------------+
|                     Python Backend (FastAPI)                      |
|                                                                   |
|  +---------------------+  +------------------------------------+  |
|  |   AG-UI Encoder     |  |         Task & Note Agent          |  |
|  | (ag-ui-protocol)    |  | (LLM Orchestration + JSON Patch)   |  |
|  +---------------------+  +------------------+-----------------+  |
+----------------------------------------------|--------------------+
                                               |
                                  +------------v------------+
                                  |          Redis          |
                                  | (thread_id & Shared State)|
                                  +-------------------------+
```

---

## Key Features

- **Protocol-First Design:** Built strictly on `ag-ui-protocol` 1.0. Any AG-UI compatible client can connect to the backend out of the box.
- **Real-Time Shared State:** Synchronizes a task and note store across clients using `STATE_SNAPSHOT` on connect and RFC 6902 JSON Patch `STATE_DELTA` events on updates.
- **Human-in-the-Loop (HITL):** Destructive operations (such as clearing all completed tasks) pause execution and require explicit user confirmation in the active interface.
- **Rate-Limit Aware Telegram Streaming:** The `aiogram` client buffers incoming SSE text deltas and throttles `edit_message_text` calls to comply with Telegram Flood Control limits.
- **Rich Terminal Experience:** The CLI client renders live streaming markdown responses and real-time task tables using `rich`.
- **Modern Engineering Stack:** Managed with `uv`, strictly typed with `mypy --strict`, linted with `ruff`, and tested via GitHub Actions CI.

---

## Tech Stack

| Layer | Technologies |
| :--- | :--- |
| **Backend & Protocol** | Python 3.12+, FastAPI, `ag-ui-protocol`, Pydantic v2, `jsonpatch`, `uv` |
| **State & Storage** | Redis (async `redis.asyncio`) |
| **Web Client** | React, TypeScript, Vite, Tailwind CSS, `@ag-ui/client` |
| **Telegram Client** | `aiogram` 3.x, `httpx` (async SSE streaming) |
| **CLI Client** | `httpx`, `rich` |
| **DevOps & Quality** | Docker, Docker Compose, GitHub Actions, `ruff`, `mypy --strict`, `pytest` |

---

## Project Structure

```text
omni-agui/
├── AGENTS.md                 # System instructions for AI coding agents (Opencode)
├── SPEC.md                   # Detailed technical & protocol specification
├── PLAN.md                   # Phased implementation roadmap
├── docker-compose.yml        # Full-stack orchestration (Redis, Backend, Bot, Web)
├── .env.example              # Environment variable template
├── backend/                  # FastAPI + ag-ui-protocol server
├── clients/
│   ├── cli/                  # Python terminal client
│   ├── telegram_bot/         # aiogram 3.x bot client
│   └── web/                  # React + TypeScript dashboard
└── .github/workflows/ci.yml  # Automated linting, type-checking, and tests
```

---

## Quick Start

### 1. Configure Environment
Clone the repository and create your local `.env` file from the template:

```bash
git clone https://github.com/skislyakow/omni-agui.git
cd omni-agui
cp .env.example .env
```

Fill in your `LLM_API_KEY` (and `TELEGRAM_BOT_TOKEN` if running the bot) inside `.env`.

### 2. Run with Docker Compose (Recommended)
Start Redis, the FastAPI backend, the Telegram bot, and the React web app with a single command:

```bash
docker compose up --build
```

- **Web Dashboard:** `http://localhost:5173`
- **Backend API / Swagger:** `http://localhost:8000/docs`

### 3. Run the CLI Client Locally
Connect to the running backend from your terminal using `uv`:

```bash
cd clients/cli
uv run python main.py --thread-id demo-session
```

---

## Development & Quality Checks

To run the backend locally without Docker and verify code quality:

```bash
cd backend
uv sync --dev
uv run ruff check .
uv run mypy src
uv run pytest
```

---

## Demo

> *Add a screen recording or GIF here demonstrating a task created in Telegram appearing instantaneously in the React Web UI via `STATE_DELTA`.*

---

## License

This project is licensed under the MIT License.
