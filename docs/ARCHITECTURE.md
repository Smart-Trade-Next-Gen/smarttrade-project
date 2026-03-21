# SmartTrade Platform — System Architecture

**Version:** 1.0
**Date:** 2026-03-21
**Status:** Production

---

## 1. Overview

SmartTrade is a broker-agnostic algorithmic trading platform built on a microservices architecture. It provides real-time market data, order execution, risk management, portfolio analytics, and an intelligent automation layer (PIE — Position Intelligence Engine) through a normalized API that abstracts away broker-specific APIs.

### Core Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Financial correctness** | `Decimal` throughout; no floats in any monetary calculation |
| **ACID guarantees** | SQLAlchemy transactions on all order/trade/position mutations |
| **User isolation** | Every DB query filtered by `user_id`; validated at service layer |
| **Async-first** | AsyncIO + asyncpg; zero blocking I/O paths |
| **Broker agnostic** | Plugin architecture; broker-specific logic fully encapsulated |
| **Event-driven** | Services communicate via typed Pydantic events on Redis |
| **Immutable audit** | All order/trade/portfolio mutations emit append-only audit records |

---

## 2. Service Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Client Browser                              │
│                     Frontend (Vite/React 18)                        │
│                          Port 5173                                   │
└───────┬──────────────────────────┬──────────────────────────────────┘
        │ REST/WebSocket           │ REST/WebSocket
        ▼                          ▼
┌───────────────┐        ┌──────────────────┐
│ Authentication│        │  Market Data     │
│   Service     │        │  Service (MDS)   │
│  Port 8001    │        │  Port 8004       │
│  PostgreSQL   │        │  PostgreSQL       │
└───────┬───────┘        └──────────┬───────┘
        │ Events                    │ Events
        ▼                           ▼
        ┌──────────────────────────────┐
        │         Redis Event Bus      │
        │    (Pub/Sub + Streams)       │
        └──────────────┬───────────────┘
                       │ Events
                       ▼
        ┌──────────────────────────────┐
        │   Broker Adapter Service     │
        │         (BAS)                │
        │        Port 8005             │
        │        PostgreSQL            │
        └──────────────┬───────────────┘
                       │ Plugin interface
               ┌───────┴────────┐
               ▼                ▼
        ┌────────────┐  ┌──────────────┐
        │   Fyers    │  │ Paper/Mock   │
        │   Plugin   │  │   Service    │
        │  (Live)    │  │  Port 8002   │
        └────────────┘  └──────────────┘
```

### Service Responsibilities Summary

| Service | Port | DB | Purpose |
|---------|------|----|---------|
| Authentication | 8001 | `smarttrade_authentication_service` | Identity, JWT lifecycle, RBAC |
| Market Data (MDS) | 8004 | `smarttrade_market_data_service` | Quotes, candles, options, Greeks |
| Broker Adapter (BAS) | 8005 | `smarttrade_broker_adapter_service` | Orders, positions, risk, PIE |
| Mock Service | 8002 | `smarttrade_mock_service` | Paper trading, test simulation |
| Frontend | 5173 | — | React SPA, dashboard, trading UI |

---

## 3. Service Architecture

### 3.1 Authentication Service

**Responsibility:** Identity management — user registration, login, JWT issuance, refresh token rotation, RBAC.

```
API Layer (FastAPI routes)
    └── user.py             POST /register, /login, /logout, /refresh, /change-password

Service Layer
    └── UserService         Business logic: bcrypt hashing, token generation, audit

Repository Layer
    ├── UserRepository      CRUD for User model
    └── RefreshTokenRepository  Token rotation + revocation

Models (PostgreSQL)
    ├── User                id, email, hashed_password, role, is_active, created_at
    └── RefreshToken        token_hash, user_id, expires_at, revoked
```

**Token Strategy:**
- Access token: HS256 JWT, 15-minute expiry, carries `user_id` + `role` claims
- Refresh token: random UUID (SHA-256 hashed in DB), 7-day expiry, rotation on every use
- Token encryption key separate from JWT secret

**Security Controls:**
- bcrypt password hashing (cost factor 12)
- Refresh token rotation (old token invalidated on use)
- Account lockout after configurable failed attempts
- Audit log for all auth events (login, logout, password change)

---

### 3.2 Broker Adapter Service (BAS)

The most complex service — manages all broker interactions, order lifecycle, risk enforcement, and the PIE automation engine.

```
API Layer
    ├── routes_orders.py           POST/GET/PUT/DELETE /api/v1/orders
    ├── routes_portfolio.py        GET /api/v1/portfolio, /funds
    ├── routes_position_group.py   Position group CRUD + exit
    ├── routes_risk.py             Risk snapshot, settings
    ├── routes_session.py          Broker session management
    ├── routes_auto_entry.py       PIE auto-entry rules
    ├── routes_strategies.py       PIE strategy CRUD
    ├── routes_kill_switch.py      Emergency kill switch
    ├── routes_actions.py          Action log query
    ├── routes_pie_status.py       PIE real-time status
    ├── routes_preferences.py      User preferences
    └── connection/
        ├── broker_connection_routes.py   Broker connection CRUD
        ├── trading_account_routes.py     Trading account CRUD
        └── routes_oauth.py              Fyers OAuth flow

Core
    ├── AdapterManager         Per-user broker session registry
    ├── BrokerAdapterService   Orchestrates order placement through plugins
    ├── PositionGraphEngine    Tracks fill-level position state (graph model)
    ├── UserBrokerSession      Per-user, per-broker session wrapper
    └── AccountStateGuard      Validates account state before trading operations

Plugin System
    ├── base.py               Abstract BrokerPlugin interface
    ├── plugin_registry.py    Plugin lookup by broker type
    ├── fyers/
    │   ├── plugin.py         Fyers implementation of BrokerPlugin
    │   ├── dto_to_fyers_mapper.py
    │   └── fyers_to_dto_mapper.py
    └── paper/
        ├── plugin.py         Paper trading plugin (routes to Mock Service)
        ├── dto_to_paper_mapper.py
        └── paper_to_dto_mapper.py

Risk Engine
    ├── engine.py             Rule evaluation orchestrator
    ├── risk_monitor_service.py   Continuous monitoring + alerting
    ├── risk_snapshot_builder.py  Snapshot computation
    ├── risk_repository.py    Persisted risk state
    ├── estimate_risk.py      Pre-trade risk estimation
    ├── action_orchestrator.py    Risk-triggered action executor (PIE integration)
    └── rules/
        ├── max_daily_loss.py
        ├── max_open_positions.py
        └── max_risk_per_trade.py

Services
    ├── OrderHandler           Order placement, modification, cancellation
    ├── PortfolioService       Holdings + P&L aggregation
    ├── PositionManagementService  Position group lifecycle
    ├── SettlementService      T+1 settlement processing
    ├── SettlementProcessor    Settlement job runner
    ├── TradingAccountService  Account CRUD + state transitions
    ├── BrokerConnectionService  Credential management + OAuth
    ├── AutoEntryService       PIE auto-entry rules CRUD
    ├── KillSwitchService      Emergency stop management
    ├── StrategyService        PIE strategy CRUD
    ├── PieStatusService       Real-time PIE status aggregation
    ├── ActionLogService       Immutable action audit log
    └── InstrumentCache        Local cache of instrument metadata from MDS

Models (PostgreSQL)
    ├── TradingAccount         Broker account record + state machine
    ├── BrokerConnection       Encrypted credentials
    ├── Order                  Order lifecycle record
    ├── Trade                  Fill records (immutable)
    ├── Position               Current position snapshot
    ├── PositionManagement     Position group definition + rules
    ├── Settlement             T+1 settlement records
    ├── ActiveStrategy         PIE strategy activation record
    ├── AutoEntry              Auto-entry rule configuration
    ├── KillSwitch             Kill switch state
    ├── ActionLog              Immutable action audit
    └── Preferences            User trading preferences
```

**Order Lifecycle:**
```
POST /orders → OrderHandler
    → AccountStateGuard.validate()
    → RiskEngine.pre_trade_check()
    → BrokerAdapterService.place_order()
        → PluginRegistry.get_plugin(broker_type)
        → plugin.place_order(dto)
        → broker API call
    → PositionGraphEngine.on_fill()
    → EventBus.publish(order.placed)
    → AuditLog.record()
```

---

### 3.3 Market Data Service (MDS)

```
API Layer
    ├── route_instrument.py        GET /instruments (search, lookup)
    ├── route_broker_instrument.py GET /broker_instruments (broker-symbol mapping)
    ├── route_data.py              GET /historical (OHLCV candles)
    ├── route_margin.py            GET /margin (margin requirements)
    ├── route_notification.py      GET /notifications
    └── routes/
        ├── option_chains.py       GET /option_chain
        ├── greeks.py              GET /greeks
        └── iv_metrics.py         GET /iv_metrics, /iv_history

WebSocket
    └── main_ws.py                 WS /ws/{broker_id}/{user_id}
        ├── WsManager              Client connection registry
        ├── BrokerScopedManager    Per-broker subscription management
        ├── OptionEventBroadcaster Fan-out option data to subscribers
        └── OptionSubscriptionHandler  Subscribe/unsubscribe messages

Plugin System
    ├── fyers/
    │   ├── plugin.py             Fyers data fetcher
    │   └── websocket.py          Fyers WebSocket adapter
    └── paper/
        ├── paper_plugin.py       Mock data plugin
        └── websocket.py          Mock WebSocket (random walk)

Core Systems
    ├── InstrumentResolverManager  Async queue for symbol → instrument resolution
    ├── InstrumentResolverWorker   Worker that resolves pending instruments
    ├── PendingEventBuffer         Buffer events while instrument resolves
    ├── OptionChainCache           Redis-backed option chain snapshot cache
    ├── OptionChainRefreshScheduler  Scheduled option chain refresh
    ├── GreeksCalculator           Black-Scholes delta/gamma/theta/vega
    ├── VolatilitySurface          IV surface modeling
    ├── IVMetricsCalculator        IV rank, IV percentile computation
    └── TradingCalendar            Exchange holiday + session time management

Models (PostgreSQL)
    ├── Instrument                 NSE/BSE instrument master
    ├── BrokerInstrument           Broker-specific symbol mapping
    ├── HistoricalCandle           OHLCV time-series
    ├── ExchangeHoliday            Trading calendar
    └── OptionChain                Option chain snapshot (DB-cached)
```

---

### 3.4 smarttrade-common (Shared Library)

All services depend on this package. It enforces cross-cutting concerns uniformly.

```
smarttrade_common/
    app_factory.py          FastAPI app creator: CORS, middleware, exception handlers, Bearer auth
    config.py               CommonSettings base (Pydantic Settings v2)
    lifespan.py             Startup/shutdown: DB init, Redis connect, event bus subscribe
    auth.py                 JWT decode middleware, get_current_user dependency
    errors.py               SmartTradeError hierarchy, standardized error codes
    audit.py                Immutable audit event emission
    idempotency.py          Idempotency key check (Redis-backed)
    locking.py              Distributed locking (Redis)

    database/
        session.py          AsyncSessionFactory, get_db dependency
        repository.py       Generic async repository (CRUD + filter)
        concurrent_repository.py  Optimistic locking + retry
        transaction.py      @transactional decorator (ACID)
        audit_models.py     Audit trail base models
        audit_repository.py Append-only audit write

    events/
        event_bus.py        Redis pub/sub + stream producer/consumer
        dispatcher.py       Route events to registered handlers

    security/
        jwt_utils.py        HS256 sign/verify, claims extraction
        rbac.py             Role-based access control (YAML policy)
        cryptography.py     AES-256 credential encryption/decryption
        password_utils.py   bcrypt hash/verify

    resilience/
        retry.py            Exponential backoff decorator
        circuit_breaker.py  Circuit breaker (half-open, open, closed states)
        timeout.py          Async operation timeout wrapper
        distributed_rate_limiter.py  Redis token-bucket rate limiter
        rate_limiter_registry.py     Per-user, per-endpoint rate limit registry

    observability/
        metrics.py          Prometheus metrics: counters, histograms, gauges
        tracing.py          OpenTelemetry trace context propagation

    middleware/
        rate_limit.py       Request-level rate limiting middleware
        request_id.py       X-Request-ID injection + propagation

    rule_engine/
        engine.py           Config-driven rule evaluation
        expressions.py      Expression AST evaluation
        loader.py           YAML rule config loader
        transform.py        Data transformation pipeline

    schemas/
        core/               Base Pydantic models (paginated response, error response)
        services/trading/   Order DTOs, order events, PIE events, risk events, session events
        services/authentication/  Auth DTOs, user events
        types/              Enums (OrderSide, OrderType, ProductType, SegmentType, BrokerType)
```

---

## 4. Data Flow

### 4.1 Real-Time Market Data Flow

```
Fyers WebSocket (external)
    ↓
MDS Fyers WebSocket Adapter
    ↓
InstrumentResolverManager
    ↓ (resolved)
BrokerScopedManager
    ↓
WsManager (fan-out to all subscribed clients)
    ↓
Frontend WebSocketProvider
    ↓
marketDataStore (Zustand) → Chart candles, QuotePanel, OrderBook
```

### 4.2 Order Placement Flow

```
Frontend OrderPlacementModal
    → POST /api/v1/orders (BAS)
        → JWT middleware (Auth validation)
        → AccountStateGuard.assert_can_trade()
        → RiskEngine.pre_trade_estimate()  [blocks if limit breach]
        → OrderHandler.place_order()
            → BrokerAdapterService.route(order)
                → FyersPlugin.place_order()
                    → Fyers REST API
        → Order saved to DB (ACID)
        → EventBus.publish("order.placed")
    ← 201 Created { order_id, status }
```

### 4.3 Position Update Flow

```
Fyers Order Socket (WebSocket)
    ↓
BAS PositionSyncConsumer
    ↓
PositionGraphEngine.on_fill(fill_event)
    ↓  (ACID transaction)
Position updated in DB
    ↓
EventBus.publish("position.updated")
    ↓
Frontend (via BAS WebSocket or polling)
    ↓
positionsStore updated → PositionsPanel re-renders
```

### 4.4 PIE (Position Intelligence Engine) Flow

```
RiskMonitorService (polling loop)
    ↓
RiskSnapshotBuilder.build(user_id)
    ↓
RiskEngine.evaluate(snapshot, rules)
    ↓ (rule breach detected)
ActionOrchestrator.execute(actions)
    ├── KillSwitchService.trigger()       → close all positions
    ├── OrderHandler.cancel_all()
    └── EventBus.publish("risk.breach")
        ↓
    Frontend NotificationCenter (real-time alert)
```

---

## 5. Event Catalog

All events are Pydantic models published to Redis Pub/Sub channels.

| Event Topic | Publisher | Subscribers | Payload |
|-------------|-----------|-------------|---------|
| `order.placed` | BAS | MDS (position sync) | order_id, user_id, symbol, qty, price |
| `order.filled` | BAS | BAS (settlement trigger) | order_id, fill_price, fill_qty |
| `order.cancelled` | BAS | — | order_id, reason |
| `order.rejected` | BAS | — | order_id, rejection_reason |
| `position.updated` | BAS | Frontend (WS) | user_id, symbol, net_qty, avg_price, pnl |
| `trade.executed` | BAS | BAS (settlement) | trade_id, order_id, price, qty, timestamp |
| `settlement.initiated` | BAS | — | settlement_id, trade_ids |
| `settlement.completed` | BAS | — | settlement_id, settled_at |
| `risk.breach` | BAS | BAS (action orchestrator) | user_id, rule_name, current_value, limit |
| `risk.alert` | BAS | Frontend (WS) | user_id, message, severity |
| `pie.strategy.activated` | BAS | BAS (monitor loop) | strategy_id, user_id, config |
| `pie.strategy.stopped` | BAS | — | strategy_id, reason |
| `pie.kill_switch.triggered` | BAS | BAS (order handler) | user_id, trigger_reason |
| `session.started` | BAS | MDS | user_id, broker_id |
| `session.ended` | BAS | MDS | user_id, broker_id |
| `user.registered` | Auth | — | user_id, email |
| `user.login` | Auth | — | user_id, ip_address |

---

## 6. Database Schema Overview

### Per-Service Isolation

Each service owns its own PostgreSQL database. There are no cross-service joins.

### BAS Key Tables

```sql
trading_accounts       (id, user_id, broker_id, account_code, state, credentials_encrypted)
orders                 (id, user_id, account_id, symbol, side, type, qty, price, status, broker_order_id, created_at)
trades                 (id, order_id, user_id, symbol, fill_price, fill_qty, timestamp)  -- immutable
positions              (id, user_id, account_id, symbol, net_qty, avg_price, pnl, updated_at)
position_groups        (id, user_id, name, rules_json, created_at)
position_group_members (position_id, group_id)
settlements            (id, trade_id, user_id, status, settlement_date)
risk_snapshots         (id, user_id, snapshot_json, evaluated_at)
active_strategies      (id, user_id, strategy_config_json, started_at, status)
auto_entries           (id, user_id, instrument, trigger_config_json, is_active)
kill_switches          (id, user_id, is_active, triggered_at, reason)
action_logs            (id, user_id, action_type, payload_json, created_at)  -- immutable
preferences            (user_id, settings_json, updated_at)
```

### MDS Key Tables

```sql
instruments            (id, symbol, exchange, instrument_type, lot_size, tick_size)
broker_instruments     (id, instrument_id, broker_id, broker_symbol, broker_token)
historical_candles     (id, instrument_id, resolution, open, high, low, close, volume, timestamp)
option_chains          (id, underlying, expiry, snapshot_json, fetched_at)
exchange_holidays      (id, exchange, date, description)
```

---

## 7. Security Architecture

### Authentication & Authorization

```
Request → TLS termination (nginx/ingress)
    → X-Request-ID injection (request_id middleware)
    → JWT Bearer validation (app_factory auth middleware)
        → decode(token, JWT_SECRET_KEY)
        → validate expiry, user_id, role
    → RBAC policy check (rbac.py)
        → load YAML policy (rbac_policies.yaml)
        → assert role has permission for route
    → Route handler (user_id injected via Depends)
```

### Credential Encryption

Broker credentials (API keys, secrets) are encrypted with AES-256-GCM before storage using `TOKEN_ENCRYPTION_KEY`. Decryption happens only in-memory at session start; plaintext never persisted.

### Input Validation

All API inputs validated via Pydantic v2 models. No raw dicts cross service boundaries. SQLAlchemy ORM prevents SQL injection. Rate limiting enforced per user per endpoint via Redis token bucket.

---

## 8. Observability

### Metrics (Prometheus)

Exposed at `GET /metrics` on each service:

| Metric | Type | Description |
|--------|------|-------------|
| `http_requests_total` | Counter | By method, path, status |
| `http_request_duration_seconds` | Histogram | P50/P95/P99 latency |
| `orders_placed_total` | Counter | By user, broker, status |
| `risk_breaches_total` | Counter | By rule_name |
| `websocket_connections_active` | Gauge | Active WS connections |
| `event_bus_published_total` | Counter | By topic |
| `event_bus_consumed_total` | Counter | By topic, consumer_group |
| `settlement_lag_seconds` | Histogram | T+1 processing delay |

### Tracing (OpenTelemetry)

Distributed traces propagated via `X-B3-TraceId` / `traceparent` headers. All service-to-service HTTP calls include trace context. Critical trading paths (order placement, risk evaluation) have explicit span creation.

### Structured Logging

All logs are JSON-structured with fields:

```json
{
  "timestamp": "ISO8601",
  "level": "INFO|WARN|ERROR",
  "service": "broker-adapter-service",
  "request_id": "uuid",
  "user_id": "uuid",
  "trace_id": "hex",
  "message": "...",
  "extra": {}
}
```

### Health Probes

| Endpoint | Purpose |
|----------|---------|
| `GET /` | Liveness probe (always 200 if process alive) |
| `GET /ready` | Readiness probe (checks DB + Redis connectivity) |

---

## 9. Resilience Patterns

### Circuit Breaker

Applied to all external broker API calls (Fyers REST, Fyers WebSocket). States: `CLOSED → OPEN → HALF_OPEN → CLOSED`. Opens after 5 consecutive failures; probes after 30s.

### Retry with Exponential Backoff

Applied to Redis event publishing and broker HTTP calls. Max 3 retries; jitter applied to prevent thundering herd.

### Distributed Rate Limiting

Per-user, per-endpoint Redis token bucket. Prevents API abuse and broker rate limit violations.

### Idempotency Keys

All order placement operations check an idempotency key (SHA-256 of `user_id + order_params + client_nonce`) against Redis before forwarding to broker. Prevents duplicate order submission on client retry.

---

## 10. Deployment Architecture

### Container Stack

```yaml
services:
  postgres:       PostgreSQL 15 (one instance; separate DBs per service)
  redis:          Redis 7 (event bus + cache + rate limiting + locks)
  auth:           Authentication Service (:8001)
  mds:            Market Data Service (:8004)
  bas:            Broker Adapter Service (:8005)
  mock:           Mock Service (:8002)
  frontend:       Vite/Nginx (:5173/80)
```

### Start Order (Dependency Chain)

```
PostgreSQL → Redis → Auth Service → MDS → BAS → Mock Service → Frontend
```

### Environment Configuration

All configuration via environment variables. Key groups:

| Group | Variables |
|-------|-----------|
| Runtime | `ENV`, `SERVICE_NAME`, `LOG_LEVEL` |
| Database | `DATABASE_URL` (asyncpg), `DATABASE_POOL_SIZE` |
| Security | `JWT_SECRET_KEY`, `TOKEN_ENCRYPTION_KEY` |
| Events | `EVENT_BUS=redis`, `EVENT_BUS_URL` |
| Broker | `FYERS_APP_ID`, `FYERS_APP_SECRET`, `BROKER_REDIRECT_URI` |
| Observability | `SENTRY_ENABLED`, `SENTRY_DSN`, `PROMETHEUS_ENABLED` |
| Frontend | `VITE_API_BASE`, `VITE_WAS_BASE` (WebSocket aggregator) |

---

## 11. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| API P99 latency | < 100ms | Excluding broker round-trip |
| Order placement | < 500ms | End-to-end (auth + risk + broker) |
| Market data fan-out | < 10ms | WebSocket broadcast to N clients |
| DB query | < 100ms | All indexed queries |
| Risk evaluation | < 50ms | Per-user snapshot computation |
| Option chain refresh | < 2s | Full chain from Fyers API |

---

## 12. Frontend Architecture

```
smarttrade-frontend/src/
    App.tsx              Router setup, auth guard
    main.tsx             React 18 root mount

    api/                 Axios clients per service
    ws/                  WebSocket connection + subscription registry
    store/               Zustand stores (20+ domain stores)
    hooks/               React hooks wrapping stores + API calls
    components/          Reusable UI components
        Chart/           EnhancedChart: realtime, replay, indicators, crosshair sync
        panels/          60+ trading panel components
        pie/             PIE dashboard (auto-entry, kill switch, monitor)
        positionManagement/  Position groups UI
    pages/               Route-level page components
    types/               TypeScript type definitions
    utils/               Formatters, chart utilities, WS helpers
```

**State Management:** Zustand stores are domain-scoped (orderStore, positionsStore, marketDataStore, etc.) with no shared mutable state between stores. WebSocket data is fanned out to relevant stores via `websocketDataFanout.ts`.

**Chart System:** Built on `lightweight-charts`. Supports multi-pane layouts, crosshair sync across charts, timeframe sync, realtime candle building from tick data, and historical replay mode.
