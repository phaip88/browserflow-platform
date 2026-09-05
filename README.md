# BrowserFlow Platform

[![CI](https://github.com/phaip88/browserflow-platform/actions/workflows/ci.yml/badge.svg)](https://github.com/phaip88/browserflow-platform/actions/workflows/ci.yml)
[![Docker Publish](https://github.com/phaip88/browserflow-platform/actions/workflows/docker-publish.yml/badge.svg)](https://github.com/phaip88/browserflow-platform/actions/workflows/docker-publish.yml)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

[English](README.md) | [中文说明](README_zh.md)

**BrowserFlow Platform** is a production-grade, self-hosted, modular browser automation and workflow orchestration platform. It is engineered with process-level isolation between the control plane and browser execution engines, preventing automated crawler and browser crashes from destabilizing the Web control plane.

---

## Key Highlights

- **Process-Level Isolation**: Playwright Chromium runs in dedicated worker processes (`apps/browser_worker`), decoupled from the FastAPI control plane (`apps/api`).
- **Deterministic DAG Compiler**: Validates, topologically sorts, and compiles visual workflows into immutable, deterministic execution plans.
- **Lease & Heartbeat Protocol**: Database-backed task leases (`attempt_id`, `lease_token`) with heartbeat reporting and automatic stale-worker reclaim guards.
- **Enterprise-Grade Security**:
  - **SSRF Prevention**: `BrowserRequestNetworkPolicy` blocks unauthorized private network traversal (RFC 1918 / loopback / cloud metadata endpoints).
  - **SafePath Guard**: Enforces strict directory containment, preventing path traversal attacks.
  - **Credential Isolation**: Secrets are encrypted using AES-256-GCM via a dedicated master key and dynamically redacted in execution logs.
  - **Authentication**: Argon2id password hashing, HttpOnly session cookies, and CSRF token protection.
- **Durable Scheduler**: Crontab and one-off scheduling engine supporting IANA timezones, misfire recovery, and trigger deduplication.
- **Extensible Node SDK**: Modular node architecture with dedicated packs for Browser, Control Flow, Data Processing, and Webhook/HTTP Integrations.
- **Bilingual Interface**: Built-in English and Simplified Chinese localization with persistent preference switching.

---

## Architecture Overview

```
                          ┌───────────────────────────┐
                          │   Web UI (@browserflow)   │
                          │  Vite + React + React Flow │
                          └─────────────┬─────────────┘
                                        │ HTTP / WS
                                        ▼
                          ┌───────────────────────────┐
                          │   FastAPI Control Plane   │
                          │        (apps/api)         │
                          └─────────────┬─────────────┘
                                        │
           ┌────────────────────────────┼───────────────────────────┐
           │                            │                           │
           ▼                            ▼                           ▼
┌───────────────────────┐   ┌───────────────────────┐   ┌───────────────────────┐
│  PostgreSQL Database  │   │  Browser Worker Pool  │   │   Durable Scheduler   │
│ (Single Source of Truth│   │ (Playwright Chromium) │   │   (apps/scheduler)    │
└───────────────────────┘   └───────────────────────┘   └───────────────────────┘
```

| Component | Technology | Role |
| :--- | :--- | :--- |
| `apps/web` | Vite, React 18, React Flow, Tailwind CSS | Visual DAG flow editor, execution monitor, credential manager |
| `apps/api` | Python 3.12+, FastAPI, SQLAlchemy (async) | RESTful API, flow compiler, user management, audit logging |
| `apps/browser_worker` | Playwright Python, Chromium | Isolated browser execution engine, task lease consumer |
| `apps/scheduler` | Python 3.12+, Croner | Crontab trigger evaluator and execution dispatcher |
| `packages/domain` | Python (Pure Domain Models) | Domain entities, state machines, and business rules |
| `packages/flow_compiler`| Python (DAG Analysis) | Flow syntax validation, cycle detection, deterministic plans |
| `packages/node_sdk` | Python | Standardized lifecycle interface for workflow nodes |

---

## Node Catalog (Release 1)

### 1. Browser Pack (`node_pack_browser`)
- `page.goto`: Navigate to URL with network policy validation.
- `page.click`: Click elements with smart waiting.
- `page.fill`: Input text into form elements.
- `page.screenshot`: Capture full-page or element screenshots into artifact storage.
- `page.evaluate`: Execute custom JavaScript within page context.
- `page.wait_for_selector`: Wait for DOM selectors to appear/disappear.
- `page.scrape_text`: Extract inner text or regex values from selectors.
- `page.press` / `page.hover`: Keyboard input and mouse hovering.
- `page.reload`: Reload the active browser page.

### 2. Control Flow Pack (`node_pack_control`)
- `control.branch`: Conditional logic branching based on execution context.
- `control.loop`: Iterate over arrays or numeric ranges.
- `control.delay`: Explicit pause / sleep with cancellation support.

### 3. Data Processing Pack (`node_pack_data`)
- `data.transform`: JSON projection and property mapping.
- `data.extract_regex`: Regular expression pattern matching on text.
- `data.json_parse`: Parse and validate raw JSON payloads.

### 4. Integration Pack (`node_pack_integration`)
- `integration.http_request`: Outbound HTTP/HTTPS requests with header and payload mapping.
- `integration.webhook`: Inbound and outbound webhook dispatchers.

---

## Quick Start (Docker Compose Production)

### 1. Prerequisites
- Docker Engine 24+ & Docker Compose v2+
- Linux, macOS, or Windows (WSL2)

### 2. Clone and Configure
```bash
git clone https://github.com/phaip88/browserflow-platform.git
cd browserflow-platform

# Generate cryptographically secure secrets
mkdir -p secrets
python3 -c "import os, secrets; \
  open('secrets/postgres_password', 'w').write(secrets.token_hex(16)); \
  open('secrets/master.key', 'wb').write(os.urandom(32)); \
  open('secrets/session.secret', 'wb').write(os.urandom(48))"
```

### 3. Launch Services
```bash
docker compose -f docker-compose.production.yml up -d --build
```

### 4. Initialize Database & Admin Account
```bash
# Apply database migrations
docker compose -f docker-compose.production.yml exec api alembic upgrade head

# Create the initial administrator user
docker compose -f docker-compose.production.yml exec api browserflow admin create
```

### 5. Access
- **Web UI**: [http://localhost:8080](http://localhost:8080)
- **API Documentation**: [http://localhost:8000/docs](http://localhost:8000/docs)
- **Health Endpoint**: `curl http://localhost:8000/health/live`

---

## Local Development Setup

### 1. Environment Setup
```bash
# Python virtual environment (Python 3.12+)
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\Activate.ps1
pip install -U pip
pip install -e ".[dev]"

# Install Playwright browser
python -m playwright install chromium

# Node.js dependencies for frontend (pnpm 9+)
pnpm install
```

### 2. Run Database
```bash
# Start PostgreSQL via docker or local service
docker run -d --name browserflow-pg -p 5432:5432 \
  -e POSTGRES_USER=browserflow \
  -e POSTGRES_PASSWORD=browserflow_dev \
  -e POSTGRES_DB=browserflow \
  postgres:16-alpine
```

### 3. Run Development Services
```bash
# Terminal 1: API Server
uvicorn browserflow.api.main:app --host 127.0.0.1 --port 8000 --reload

# Terminal 2: Browser Worker
python -m browserflow.browser_worker

# Terminal 3: Scheduler
python -m browserflow.scheduler

# Terminal 4: Frontend UI
pnpm --filter @browserflow/web dev
```

### 4. Testing
```bash
# Backend pytest suite (unit, contracts, integration, security, stability)
pytest tests -q

# Frontend tests & linting
pnpm --filter @browserflow/web test
pnpm --filter @browserflow/web typecheck
```

---

## Container Registry Images

Pre-built Docker images are automatically compiled and published via GitHub Actions to GitHub Packages (GHCR):

- **API & Scheduler**: `ghcr.io/phaip88/browserflow-platform-api:latest`
- **Browser Worker**: `ghcr.io/phaip88/browserflow-platform-worker:latest`
- **Frontend Web**: `ghcr.io/phaip88/browserflow-platform-web:latest`

---

## Security Policy

- **No Plaintext Fallbacks**: All credentials and tokens must be injected via runtime environment variables or Docker Secrets.
- **Reporting Vulnerabilities**: Please review [SECURITY.md](SECURITY.md) for vulnerability disclosure guidelines.

---

## License

This project is licensed under the Apache 2.0 License. See the [LICENSE](LICENSE) file for details.
