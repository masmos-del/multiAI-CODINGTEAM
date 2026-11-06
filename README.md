# Project 1 — Multi-AI Coding Team v4

Production-oriented local autonomous software-engineering platform. Multiple AI roles collaborate over a real workspace, run actual checks, repair failures, and can only release when deterministic quality gates pass.

## v4 improvements

- Persistent SQLite-backed run queue; work is no longer launched as an in-memory `asyncio.create_task()` from the HTTP request.
- Atomic project locking and queue claiming.
- Interrupted-run recovery after backend restart.
- Git repository initialized per workspace with automatic commits at important stages.
- Strict Pydantic contracts for agent outputs.
- Atomic filesystem changes with path traversal protection and change-size limits.
- Checkpoint archives + SHA-256 manifests + rollback API.
- Real test execution in Docker by default, with network disabled and resource limits.
- Security scanner for common dangerous patterns and hardcoded secrets.
- Static-analysis/build checks for Python and JavaScript projects.
- Optional browser acceptance-test hook for Playwright/Cypress/etc.
- Token usage and estimated cost ledger per agent/model.
- Deterministic release gate: tests + review + security + static analysis + browser QA + Finalizer.
- Full audit trail of agent messages and lifecycle events.
- Retry/cancel/retry-run APIs.
- React/Vite operator UI.

## Architecture

```text
                  HTTP / React UI
                        |
                        v
                 FastAPI API layer
                        |
                        v
                SQLite durable queue
                        |
                        v
                  Worker process
                        |
              +---------+---------+
              |                   |
              v                   v
         Agent Orchestrator    Audit/Usage DB
              |
      +-------+-------+----------------+
      |       |       |       |        |
   Manager Developer Reviewer Fixer Finalizer
              |       |
              v       v
            Git   Workspace tools
              |       |
              +--- Quality Gate ---+
                  |       |        |
                Tests  Security  Static/Browser
                  \       |        /
                   +------v-------+
                       RELEASE
```

## Queue semantics

`POST /api/projects/{id}/runs` only creates a durable `queued` row. The worker claims it transactionally. A server restart marks active work `interrupted`; it never falsely reports a lost job as completed.

## Quality gate

A run can be `completed` only if:

1. configured tests exit with code 0;
2. Reviewer has zero blocker/major findings;
3. security scan has no critical/high findings;
4. static analysis/build checks pass;
5. configured browser QA passes;
6. Finalizer returns `ready=true`.

The LLM cannot override this gate.

## Quick start

### Backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload --port 8000
```

### Frontend

```bash
cd frontend
npm install
npm run dev
```

Open `http://localhost:5173`.

### Secure execution

Use Docker for AI-generated code:

```env
EXECUTION_MODE=docker
DOCKER_NETWORK=none
DOCKER_MEMORY=768m
DOCKER_CPUS=2
```

Host execution is disabled by default and should only be enabled for trusted local projects.

## API additions

- `POST /api/runs/{id}/rollback`
- `POST /api/runs/{id}/retry`
- `GET /api/runs/{id}/usage`
- `GET /api/runs/{id}/security`
- `GET /api/projects/{id}/git`

## AI providers

Any OpenAI-compatible `/chat/completions` endpoint can be used. Ollama can be used by setting `AI_PROVIDER=ollama` and its base URL.

## Important security note

Docker execution here is a practical local sandbox, not a hostile-adversary security boundary. Never mount cloud credentials, SSH keys, production secrets, Docker socket, or sensitive host directories into generated-code workloads. For hostile/untrusted code use a dedicated VM or hardened sandbox service.

## Tests

```bash
cd backend
pytest -q
```
