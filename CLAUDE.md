# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SmartTrade is a **microservices-based trading platform** providing a normalized, broker-agnostic interface for executing trades, managing market data, and authentication. The primary broker integration is Fyers, with architecture supporting others (Zerodha, Interactive Brokers).

## Services & Ports

| Service | Port | Database |
|---|---|---|
| Authentication Service | 8001 | `smarttrade_authentication_service` |
| Mock Service | 8002 | `smarttrade_mock_service` |
| Market Data Service (MDS) | 8004 | `smarttrade_market_data_service` |
| Broker Adapter Service (BAS) | 8005 | `smarttrade_broker_adapter_service` |
| Frontend | 5173 | — |

All backend services are FastAPI + Python 3.12, using `uv` as the package manager. PostgreSQL is used in production, SQLite for local dev. Redis is the event bus.

## Commands

### Python Services (run inside each service directory)

```bash
uv sync                        # Install dependencies
uv sync --extra dev            # Include dev dependencies
uv run uvicorn <module>.main:app --reload  # Run service

uv run pytest                  # Run all tests
uv run pytest tests/path/to/test.py::test_name  # Run single test
uv run pytest -m unit          # Run only unit tests
uv run pytest --cov            # With coverage

uv run ruff check src/         # Lint
uv run mypy src/               # Type check

uv run alembic upgrade head    # Apply DB migrations
uv run alembic revision -m "description"  # Create migration
```

pytest is configured with `asyncio_mode = auto` and `--maxfail=1` by default.

### Frontend (`smarttrade-frontend/`)

```bash
npm install
npm run dev      # Vite dev server with HMR
npm run build    # Production build
npm run lint     # ESLint on .ts/.tsx files
```

### Docker (all services together)

```bash
docker-compose up                              # Start everything
docker-compose -f docker-compose.local.yml up  # Alternative local config
docker-compose logs -f <service-name>          # Follow logs
```

## Architecture

### `smarttrade-common` — Shared Library

All services depend on this local package. It provides:
- `app_factory.py` — Creates FastAPI app with standard middleware, CORS, exception handlers, Bearer auth
- `lifespan.py` — Startup/shutdown hooks (DB init, event bus connection)
- `config.py` — `CommonSettings` base class (Pydantic Settings); each service extends it
- `auth.py` / `security/` — JWT (HS256) generation/validation, RBAC, bcrypt password hashing
- `database/` — Async SQLAlchemy session management, generic repository pattern, Alembic support
- `events/` — Redis/Kafka event bus, publish/subscribe, Pydantic event schemas
- `middleware/` — Rate limiting, request ID tracking, auth enforcement
- `resilience/` — Retry with exponential backoff, circuit breaker, timeouts
- `rule_engine/` — Config-driven validation rules (used by risk engine in BAS)
- `errors.py` — `SmartTradeError` base exception with standardized error codes (e.g., `VAL_001`, `AUTH_004`)
- `http_client/` — Service-to-service HTTP with retry/timeout/circuit breaker
- `observability/` — Prometheus metrics, OpenTelemetry tracing
- `health.py` — Liveness (`GET /`) and readiness (`GET /ready`) probes

### Service Responsibilities

**Authentication Service** — User registration/login, JWT access + refresh token lifecycle, bcrypt password hashing, RBAC role assignment, audit logging.

**Broker Adapter Service (BAS)** — Translates SmartTrade order models ↔ Fyers API calls, risk validation via rule engine, position/fund aggregation, session management for broker connections, publishes domain events (order.placed, order.filled, etc.).

**Market Data Service (MDS)** — Subscribes to Fyers WebSocket data streams, fans out market data to connected clients, instrument resolution & broker mapping, trading calendar management.

**Mock Service** — Mirrors BAS API for testing without real broker connections; simulates fills and price movements.

**Frontend** — React 18 + TypeScript, Vite, Tailwind CSS. State via Zustand, charts via `lightweight-charts`, dashboard layout via `react-grid-layout`. Connects to backend via axios and WebSocket.

### Key Patterns

- **Database-per-service**: Each service owns its own PostgreSQL database; no cross-service DB queries.
- **Event-driven**: Services communicate asynchronously via Redis (or Kafka) using Pydantic-typed domain events.
- **Async-first**: All services use AsyncIO + asyncpg; all DB queries and HTTP calls are async.
- **RBAC**: Roles enforced via JWT claims and decorators from `smarttrade-common`.
- **Config**: All settings via environment variables; `CommonSettings` base class using Pydantic Settings.

### Service Start Order

PostgreSQL → Redis → Auth Service → MDS → BAS → Mock Service → Frontend

### Environment Variables (key ones)

```
ENV=local|dev|staging|prod
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db
JWT_SECRET_KEY=<base64>
TOKEN_ENCRYPTION_KEY=<base64>
EVENT_BUS=redis
EVENT_BUS_URL=redis://redis:6379/0
FYERS_APP_ID=...
FYERS_APP_SECRET=...
BROKER_REDIRECT_URI=http://www.smarttrade.asia/
SENTRY_ENABLED=true|false
PROMETHEUS_ENABLED=true
VITE_API_BASE=...          # Frontend: backend API URL
VITE_WAS_BASE=...          # Frontend: WebSocket aggregator URL
```

## Testing

Integration tests are in Postman (`SmartApp Integration Tests.postman_collection.json` + `Smartapp Local.postman_environment.json` at the repo root).

Each service has its own `tests/` directory with unit and integration tests. pytest markers: `unit`, `slow`.
