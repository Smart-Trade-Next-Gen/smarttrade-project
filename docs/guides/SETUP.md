# How to: Set Up SmartTrade Locally

---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Python | 3.12 | All backend services |
| `uv` | latest | Python package manager |
| Node.js | 18+ | Frontend |
| npm | 9+ | Frontend packages |
| PostgreSQL | 15 | Primary database |
| Redis | 7 | Event bus + cache |
| Git | any | Version control |

---

## 1. Clone the Monorepo

```bash
git clone https://github.com/your-org/Smart-Trade.git
cd Smart-Trade

# Initialize all submodules
git submodule update --init --recursive
```

---

## 2. Start Infrastructure (PostgreSQL + Redis)

**Option A: Docker (recommended)**

```bash
docker-compose -f docker-compose.local.yml up postgres redis -d
```

**Option B: Local install**

```bash
# PostgreSQL: create databases
createdb smarttrade_authentication_service
createdb smarttrade_market_data_service
createdb smarttrade_broker_adapter_service
createdb smarttrade_mock_service

# Redis: start
redis-server --daemonize yes
```

---

## 3. Set Up smarttrade-common (shared library)

```bash
cd smarttrade-common
uv sync
```

This installs the shared library in editable mode. All other services reference it as a local dependency.

---

## 4. Set Up Each Service

### Authentication Service (Port 8001)

```bash
cd authentication-service
cp .env.template .env
# Edit .env: set DATABASE_URL, JWT_SECRET_KEY
uv sync

# Run migrations
uv run alembic upgrade head

# Start service
uv run uvicorn authentication_service.main:app --port 8001 --reload
```

### Market Data Service (Port 8004)

```bash
cd market-data-service
cp .env.template .env
# Edit .env: set DATABASE_URL, FYERS_APP_ID, FYERS_APP_SECRET, AUTH_SERVICE_URL
uv sync

uv run alembic upgrade head
uv run uvicorn market_data_service.main:app --port 8004 --reload
```

### Broker Adapter Service (Port 8005)

```bash
cd broker-adapter-service
cp .env.template .env
# Edit .env: set DATABASE_URL, JWT_SECRET_KEY, TOKEN_ENCRYPTION_KEY, MDS_URL
uv sync

uv run alembic upgrade head
uv run uvicorn broker_adapter_service.main:app --port 8005 --reload
```

### Mock Service (Port 8002)

```bash
cd mock-service
cp .env.template .env
uv sync

uv run alembic upgrade head
uv run uvicorn mock_service.main:app --port 8002 --reload
```

### Frontend (Port 5173)

```bash
cd smarttrade-frontend
cp .env.example .env
# Verify VITE_* variables point to local service ports
npm install
npm run dev
```

---

## 5. Environment Variables Reference

### Required for all services

```bash
ENV=local
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/<service_db>
EVENT_BUS=redis
EVENT_BUS_URL=redis://localhost:6379/0
JWT_SECRET_KEY=<generate: openssl rand -base64 32>
TOKEN_ENCRYPTION_KEY=<generate: openssl rand -base64 32>
LOG_LEVEL=INFO
```

### BAS additional

```bash
FYERS_APP_ID=<your Fyers App ID>
FYERS_APP_SECRET=<your Fyers App Secret>
BROKER_REDIRECT_URI=http://localhost:5173/oauth/callback
MDS_URL=http://localhost:8004
MOCK_SERVICE_URL=http://localhost:8002
```

### Frontend

```bash
VITE_AUTH_BASE=http://localhost:8001
VITE_BAS_BASE=http://localhost:8005
VITE_MDS_BASE=http://localhost:8004
VITE_WAS_BASE=ws://localhost:8004
VITE_MOCK_BASE=http://localhost:8002
```

---

## 6. Generate JWT Secret Keys

```bash
# JWT Secret (for signing access tokens)
python -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"

# Token Encryption Key (for AES-256 credential encryption)
python -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

Both values go into `.env` for each service that needs them.

---

## 7. Seed Initial Data

```bash
# Create the initial admin user (Authentication Service)
cd authentication-service
uv run python -c "
from authentication_service.seeder import seed_admin
import asyncio
asyncio.run(seed_admin(email='admin@smarttrade.local', password='Admin@123'))
"

# Sync Fyers instrument master (Market Data Service)
cd market-data-service
uv run python -c "
from market_data_service.services.instrument_sync_service import sync_instruments
import asyncio
asyncio.run(sync_instruments(broker_id='fyers'))
"
```

---

## 8. Verify Everything Is Running

```bash
# Health checks
curl http://localhost:8001/ready      # Auth Service
curl http://localhost:8004/ready      # MDS
curl http://localhost:8005/ready      # BAS
curl http://localhost:8002/ready      # Mock Service
```

All should return `{"status": "ok"}`.

Open `http://localhost:5173` in your browser. Login with the seeded admin credentials.

---

## 9. Running Tests

### All tests in a service

```bash
cd broker-adapter-service
uv run pytest                         # All tests
uv run pytest -m unit                 # Unit tests only
uv run pytest --cov src/ --cov-report=html  # With coverage
```

### Specific test file

```bash
uv run pytest tests/unit/test_order_handler.py -v
```

### Integration tests (requires running services)

```bash
uv run pytest tests/integration/ -v
```

---

## 10. Service Start Order

If starting all services manually, follow this order:

```
1. PostgreSQL
2. Redis
3. Authentication Service  (other services validate JWTs — start first)
4. Market Data Service     (BAS uses MDS for instrument resolution + quotes)
5. Broker Adapter Service  (main trading service)
6. Mock Service            (paper trading, optional)
7. Frontend
```

---

## 11. Common Issues

### `asyncpg.exceptions.InvalidCatalogNameError`
Database doesn't exist. Run `createdb <db_name>` first.

### `redis.exceptions.ConnectionError`
Redis not running. Start with `redis-server` or `docker-compose up redis`.

### `JWT_SECRET_KEY not set`
Copy `.env.template` to `.env` and fill in required values.

### Fyers OAuth redirect fails
Ensure `BROKER_REDIRECT_URI` matches what you registered in the Fyers app dashboard.

### `alembic.util.exc.CommandError: Can't locate revision`
Migrations out of sync. Run `uv run alembic stamp head` then `uv run alembic upgrade head`.
