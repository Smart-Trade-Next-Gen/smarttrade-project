# SmartTrade Architecture v3.4 — CURRENT + Market Data Distribution Update

**Status**: In Production (Core Complete) | Phase 10: Production Hardening + Phase 11: Market Data Refactor (Architecture Spec Complete)
**Date**: 2026-04-10 (original) → 2026-04-20 (Market Data Distribution Architecture v1.0)
**Version**: 3.4 — CURRENT + Market Data Distribution Pattern v3.5+ (Architecture Spec Phase)
**Previous Milestone**: v3.2 (March 2026 — Core implementation complete)
**Current Phase**: Phase 10: Production Hardening (3/12 tasks complete)
**Production Status**: All critical components deployed and tested. Core architecture stable. Phase 10 focuses on monitoring, observability, and deployment hardening.

---

## Executive Summary

SmartTrade v3.4 is a **microservices-based, broker-agnostic trading platform** with:

- **Core**: Centralized execution layer (Execution Orchestrator) ensuring atomic order handling, idempotency, and audit trails
- **Intelligence**: Signal Engine (cross-market pattern detection) → Strategy Runtime (user-defined algos) → Execution
- **Safety**: Actor model concurrency (per-user serialization), broker sync (drift detection), external trade handling, portfolio-level controls
- **Advisory** (NEW in v3.4): AI Orchestrator (LLM-powered insights, advisory only) + Notification Service (real-time alerts)
- **Support**: Market data (real-time quotes, instrument resolution), risk engine (limits, daily loss), position management, execution tracking, Journal Service (trade history + behavioral learning)

**Key Differences from v3.3**:
- Adds AI Orchestrator (advisory layer, strictly non-trading)
- Adds Notification Service (async alert delivery)
- Expands Journal Service (trade history + AI learning)
- All new services are event-driven and non-blocking

**Key Difference from v3.1**: Separates Portfolio Engine from Position Engine; adds explicit Strategy Runtime Engine; formalized Signal Engine with indicator + options + event signals.

**Implementation Status** (As of 2026-04-10):
- ✅ **Core Infrastructure**: Authentication, PostgreSQL (per-service), Redis, Event Bus (Redis Streams), smarttrade-common library
- ✅ **Broker Adapter Service (BAS)**: 
  - Order State Machine (Phase 4, ✅ v1.0 production-ready)
  - Idempotency & Deduplication (Phase 3, ✅ v1.0 production-ready)
  - Execution Orchestrator (Phase 8, ✅ v2.1 production-ready)
  - Outbox Pattern & OutboxProcessor (Phases 5-6, ✅ v1.0 production-ready)
  - Risk Engine with enhanced validation
  - Position Intelligence Engine (PIE)
  - Broker Adapters (Fyers + Paper Broker Service)
- ✅ **Market Data Service (MDS)**: Real-time quotes, instrument resolution, trading calendar, WebSocket feeds
- ✅ **Paper Broker Service** (renamed from Mock Service): Paper trading execution backend for integration testing
- ✅ **Test Coverage**: 250+ unit tests, 100+ integration tests, E2E test suite (smarttrade-tests)
- 🔄 **Phase 10 Progress**: 
  - ✅ TASK_10_1: Load testing baseline complete
  - ✅ TASK_10_2: Performance tuning complete
  - ✅ TASK_10_3: Chaos engineering complete
  - ⏳ TASK_10_4-10_8: Monitoring, observability, deployment runbooks (in progress)
- ❌ **Future (Phase 11+)**: Strategy Engine, Signal Engine, AI Orchestrator (advisory-only, non-trading)

---

## 1. Current State vs Target State

### Current Implementation (v3.1 Actual)

| Component | Status | Notes |
|-----------|--------|-------|
| Order Placement | ✅ | Ad-hoc; no state machine |
| Position Tracking | ✅ | Complete; P&L calculation done |
| Risk Engine | ✅ | YAML-driven; daily loss, position limits |
| PIE (Position Intelligence) | ✅ | Auto-entry, kill-switch, rule triggering |
| Market Data | ✅ | Quotes, instrument resolution, trading calendar |
| Authentication | ✅ | JWT, RBAC, bcrypt |
| **Quote Cache (BAS)** | ⚠️ | In-memory local cache; should consume from Redis Streams (market.quote.v1) + Redis KV with freshness decision logic |
| **Instrument Cache (BAS)** | ⚠️ | Local metadata cache; duplicates MDS responsibility; should fetch from MDS API with 24h TTL cache |
| **Quote Distribution (MDS→BAS/PBS)** | ⏳ | WebSocket-based (non-durable); should use Redis Streams + KV pattern (durable, ordered, idempotent, replay-safe) |
| **Action Logging (BAS)** | ⚠️ | Stores PIE logs in BAS DB; should be event-driven (Journal Service) |
| **Execution Orchestrator** | ❌ | No centralized queue |
| **Idempotency** | ❌ | Not integrated into endpoints |
| **Order State Machine** | ❌ | No formal states |
| **Broker Sync** | ❌ | Basic position sync; no external trades |
| **Strategy Engine** | ❌ | PIE-only; no user DSL |
| **Signal Engine** | ❌ | Not implemented |
| **Portfolio Engine** | ⚠️ | Mixed with Position Engine |

**Test Coverage**: 250+ (unit + integration); production-ready core.
**Deployment**: Docker Compose, Redis event bus, PostgreSQL (per-service).

### Target State (v3.2 Complete)

| Component | Status | Effort |
|-----------|--------|--------|
| All of v3.1 | ✅ | — |
| + Order State Machine | Phase 1 | 3 hrs |
| + Idempotency integration | Phase 1 | 2 hrs |
| + Execution Orchestrator | Phase 2 | 30 hrs |
| + Broker Sync Engine | Phase 3 | 80 hrs |
| + Strategy Engine | Phase 4 | 100 hrs |
| + Strategy Runtime | Phase 4 | 50 hrs |
| + Signal Engine | Phase 5 | 80 hrs |
| + Portfolio Engine (separated) | Phase 2-3 | 40 hrs |
| + Production stabilization | Phase 2.5 | 60 hrs |

**Total New Effort**: ~435 hours (≈24 weeks @ 18 hrs/week).

---

## 1.5 Architectural Debt & Phase 1 Cleanup Items

**Scope**: Issues in current implementation that must be cleaned up before full production rollout.

### Phase 1 Cleanup (Before Go-Live)

| Issue | Location | Impact | Fix Effort | Priority |
|-------|----------|--------|-----------|----------|
| **Quote Cache Duplication** | MDS + BAS + PBS | Violates MDS ownership; non-durable (no replay); creates stale price risk | Implement Redis Streams + KV pattern; BAS/PBS consumer groups | **HIGH** |
| **Instrument Cache Duplication** | `broker_adapter_service/instrument_cache.py` | Duplicates MDS; metadata divergence risk | 6 hrs | **HIGH** |
| **Action Logging Sync** | `broker_adapter_service/action_log_service.py` | Stores logs in BAS DB; should be event-driven | 8 hrs | **MEDIUM** |
| **Portfolio Mixed with Position** | `broker_adapter_service/portfolio_service.py` | Violates single responsibility; blocks portfolio-level rules | Refactor Phase 2-3 | **MEDIUM** |

**Total Phase 1 Cleanup Effort**: ~18 hours (reduce by moving quote/instrument cache to consumption model)

**Recommended Approach** (Redis Streams + KV Pattern):
```
BEFORE: BAS maintains local cache of quotes & instruments via WebSocket
  quote_store.py (in-memory dict, non-durable)
  MDS WebSocket subscription (per-user, fragile)
  No event trail or replay capability

AFTER: BAS consumes from durable Redis Streams + KV pattern (v3.5+)
  
MDS PUBLISHES (Redis → KV first, then Stream):
  1. Write Redis KV: key=quote:{instrument_id}, TTL=60s
     Value: { instrument_id, ltp, bid, ask, timestamp, sequence_number }
  2. Publish Redis Stream: stream=market.quote.v1
     Fields: { instrument_id, ltp, bid, ask, timestamp, sequence_number }
  
  → Guarantees: KV-first ordering; sequence_number per instrument (Redis INCR on seq:{id})

BAS CONSUMES (Redis Stream + KV for freshness):
  1. Consumer group: bas-quote-consumer on market.quote.v1
  2. On each event:
     - Check sequence_number for idempotency (skip if already seen)
     - Fetch Redis KV for latest snapshot
     - Compare event.ltp vs kv.ltp (prefer KV if lag > 100ms OR |divergence| > 0.1%)
     - Update in-memory quote_store
     - ACK stream message
  
  → Guarantees: Durable, ordered, idempotent, replay-safe; stale-price protection

PBS FOLLOWS SAME PATTERN:
  - Consumer group: pbs-quote-consumer on market.quote.v1
  - Updates price_cache → triggers order execution
  - Same freshness decision logic as BAS
```

**Key Benefits**:
- ✅ Durable event log (Redis Streams) enables replay and audit
- ✅ Latest snapshot in Redis KV ensures sub-100ms freshness
- ✅ Sequence numbers prevent duplicate fills and enable idempotency
- ✅ Eliminates per-user WebSocket clients (simplified, centralized)
- ✅ Solves stale-price risk by comparing stream event vs KV snapshot


### Phase 2-3 Refactoring (Blocking Future Features)

| Issue | Component | Target Phase | Impact | Details |
|-------|-----------|--------------|--------|---------|
| **Portfolio Engine Separation** | `portfolio_service.py` (currently in BAS) | Phase 2-3 | Blocks portfolio-level risk rules, Greeks aggregation | Currently mixed with PositionEngine; needs clear boundary |
| **Action Log Event Model** | `action_log_service.py` (currently sync writes) | Phase 2-3 | Blocks Journal Service integration | Switch from direct DB writes to event publishing |
| **PIE/Strategy Boundary** | PIE + Strategy (~1,042 lines) | Phase 4 Pre-Work | Blocks Strategy Engine launch | Extract PIE to separate policy engine; clarify defensive vs. offensive logic |

---

## 2. Architecture Overview (v3.2)

### High-Level System Design

```
┌──────────────────────────────────────────────────────────────┐
│ FRONTEND (React 18 + TypeScript)                             │
│ Dashboard, charts, orders, positions, strategy controls      │
└──────┬───────────────────────────────────────────────────────┘
       │ HTTP / WebSocket
       ▼
┌──────────────────────────────────────────────────────────────┐
│ API GATEWAY (future; currently direct service calls)         │
└──┬───────────────────────────────────────────────────────────┘
   │
   ├─────────────────┬──────────────────┬──────────────────┐
   │                 │                  │                  │
   ▼                 ▼                  ▼                  ▼
┌──────────┐  ┌─────────────┐  ┌──────────────┐  ┌──────────┐
│ Auth     │  │ Market Data │  │ Broker       │  │ Strategy │
│ Service  │  │ Service     │  │ Adapter      │  │ Service  │
│ (8001)   │  │ (8004)      │  │ Service      │  │ (future) │
│          │  │             │  │ (8005)       │  │          │
│ • JWT    │  │ • Quotes    │  │ • Orders     │  │ • Signal │
│ • RBAC   │  │ • Instruments│  │ • Risk       │  │ • Eval   │
│ • Users  │  │ • Calendar  │  │ • Positions  │  │ • State  │
└──────┬───┘  │ • Trading   │  │ • Execution  │  └──────┬───┘
       │      │   Times     │  │ • PIE        │         │
       │      └──────┬──────┘  │ • Adapter    │         │
       │             │         └────────┬─────┘         │
       │             │                  │               │
       └─────────────┼──────────────────┼───────────────┘
                     │                  │
    ┌────────────────▼──────────────────▼─────────────────┐
    │ EVENT BUS (Kafka or Redis Streams)                  │
    │ Durable, at-least-once event streaming              │
    │ Versioned events + idempotent handling               │
    ├────────────────┬──────────────────┬────────────────┤
    │ order.*.v1        │ position.*.v1      │ market_data.*.v1 │
    │ trade.*.v1        │ portfolio.*.v1     │ signal.*.v1      │
    │ execution.*.v1    │ risk.breach.v1     │ ai.*.v1          │
    │ notification.*.v1 │ journal.*.v1       │ system.*.v1      │
    └────────────────┴──────────────────┴────────────────┘
       │
       ├──────────────────┬─────────────────┬──────────────────┐
       │                  │                 │                  │
       ▼                  ▼                 ▼                  ▼
    ┌─────────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ PostgreSQL      │ │ AI           │ │ Notification │ │ Journal      │
    │ (per-service)   │ │ Orchestrator │ │ Service      │ │ Service      │
    │                 │ │ (Port 8007)  │ │ (Port 8008)  │ │ (Port 8009)  │
    │ • auth_db       │ │              │ │              │ │              │
    │ • bas_db        │ │ • Insights   │ │ • Push       │ │ • Trades     │
    │ • mds_db        │ │ • Scores     │ │ • Email      │ │ • Notes      │
    │ • strategy_db   │ │ • Advisory   │ │ • WebSocket  │ │ • Analytics  │
    └─────────────────┘ │ (non-trading)│ │ • DLQ retry  │ │ • Features   │
                        └──────────────┘ └──────────────┘ └──────────────┘
```

### Event Bus Guarantees (FIX-01: Durability Upgrade)

**Technology**: Kafka (preferred) or Redis Streams (minimum).

**Delivery Guarantees**:
- **At-least-once delivery**: Every event is delivered to subscribers at least once (never lost)
- **Message persistence**: All events persisted to disk before acknowledgment
- **Consumer groups**: Multiple services consume same event independently via consumer groups
- **Replay capability**: Events can be replayed from any offset (debugging, recovery)
- **Idempotency requirement**: All services must handle event deduplication (same event processed twice)

**Rules**:
- ✅ All critical events (order.*, position.*, risk.*) → Kafka/Redis Streams (durable)
- ✅ All events must be versioned (e.g., `order.placed.v1`)
- ✅ No service can bypass event bus for critical operations
- ❌ Pub/Sub allowed only for non-critical ephemeral signals (market ticks not needed for audit)
- ✅ Retention: All trading events retained for 90 days minimum

**Example: Durable Event Publishing**:
```python
# Before: fire-and-forget (Redis Pub/Sub)
# await redis.publish("order_channel", json.dumps(order_data))

# After: durable streaming (Kafka / Redis Streams)
await kafka_producer.send(
    topic="order.placed.v1",
    key=f"user_{user_id}",
    value=OrderPlacedEvent(**order_data).model_dump_json(),
    timestamp_ms=int(datetime.utcnow().timestamp() * 1000)
)
# Guarantees: persisted, ordered per user, replayable, idempotent
```

---

### Service Responsibilities Matrix

| Service | Responsibility | Consumes | Produces |
|---------|---|---|---|
| **Auth** | User identity, JWT, RBAC | — | user.registered.v1, user.logged_in.v1 |
| **MDS** | Real-time quotes, instrument resolution, trading calendar | broker.session.v1 | market.quote.v1 |
| **BAS** | Orders, positions, risk, execution | user requests + events | order.*.v1, position.*.v1, portfolio.*.v1, trade.*.v1 |
| **Strategy** (future) | User-defined algos, backtesting | market.quote.v1, signal.*.v1 | strategy.signal.v1 |
| **AI Orchestrator** (NEW) | LLM insights, trade scoring, nudges | trade.*.v1, order.*.v1, strategy.*.v1 | ai.trade.score.v1, ai.nudge.v1, ai.warning.v1 |
| **Notification Service** (NEW) | Alert delivery (push, email, WS) | ai.*.v1, risk.*.v1, order.*.v1 | notification.sent.v1, notification.failed.v1 |
| **Journal Service** (NEW) | Trade history, behavioral analytics | trade.*.v1, ai.*.v1 | journal.entry.created.v1, journal.insight.generated.v1 |
| **smarttrade-common** | Shared infrastructure | — | (used by all services) |

---

## 2.5 Global Contract Registry (FIX-02: API & Event Contracts)

**Purpose**: Define, version, and validate all contracts (APIs, events) in one place. Prevents undocumented APIs or event schemas.

**Directory Structure** (in smarttrade-project):
```
contracts/
├─ events/
│  ├─ order/
│  │  ├─ order.placed.v1.json
│  │  ├─ order.filled.v1.json
│  │  └─ order.cancelled.v1.json
│  ├─ position/
│  │  ├─ position.changed.v1.json
│  │  └─ position.closed.v1.json
│  └─ portfolio/
│     ├─ portfolio.updated.v1.json
│     └─ risk.breach.v1.json
│
└─ api/
   ├─ broker-adapter-service/
   │  ├─ orders.v1.openapi.json
   │  ├─ positions.v1.openapi.json
   │  └─ risk.v1.openapi.json
   ├─ market-data-service/
   │  ├─ instruments.v1.openapi.json
   │  └─ quotes.v1.openapi.json
   └─ auth-service/
      └─ auth.v1.openapi.json
```

**Rules**:
- ✅ Every event must have a JSON schema (Pydantic model → JSON Schema)
- ✅ Every REST API must have OpenAPI spec (auto-generated by FastAPI `openapi.json`)
- ✅ No service can publish/consume events without registered schema
- ✅ No API endpoint can exist without documented contract
- ✅ All contracts versioned (v1, v2, v3); breaking changes require version bump
- ❌ Backward compatibility required for minor changes

**Example: Event Contract (JSON Schema)**:
```json
// contracts/events/order/order.placed.v1.json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "OrderPlaced",
  "type": "object",
  "required": ["order_id", "user_id", "symbol", "side", "quantity", "timestamp"],
  "properties": {
    "order_id": {"type": "string"},
    "user_id": {"type": "string"},
    "symbol": {"type": "string"},
    "side": {"enum": ["BUY", "SELL"]},
    "quantity": {"type": "number", "minimum": 0},
    "price": {"type": "number", "minimum": 0},
    "timestamp": {"type": "string", "format": "date-time"},
    "idempotency_key": {"type": "string"}
  }
}
```

**Validation**: Before deploying, run schema validation:
```bash
# Validate all event schemas
uv run pytest tests/test_contract_validation.py

# Check OpenAPI compatibility
uv run pytest tests/test_api_contracts.py
```

---

## 3. Core Components

### 3.1 Authentication Service (Port 8001)

**Responsibility**: User registration, login, JWT token lifecycle, RBAC.

**Routes**:
```
POST   /api/v1/auth/register        — Register new user
POST   /api/v1/auth/login           — Login, get JWT
POST   /api/v1/auth/refresh         — Refresh access token
POST   /api/v1/auth/logout          — Revoke refresh token
POST   /api/v1/auth/change-password — Change password
```

**JWT Structure**:
```json
{
  "sub": "user_id",
  "email": "user@example.com",
  "roles": ["trader"],
  "iat": 1640000000,
  "exp": 1640003600
}
```

**RBAC Roles**: `admin`, `trader`, `viewer` (enforced in routes via `@require_role` decorator).

**Events Published** (FIX-07: Versioned):
- `user.registered.v1` (new user)
- `user.logged_in.v1` (successful login)
- `user.logged_out.v1` (logout)
- `user.password_changed.v1` (password update)

**Event Schema** (Pydantic):
```python
class UserRegisteredV1(BaseEvent):
    event_type: Literal["user.registered.v1"]
    user_id: str
    email: str
    timestamp: datetime
    idempotency_key: str
```

---

### 3.2 Broker Adapter Service (Port 8005)

**Responsibility**: Order execution, position management, risk validation, execution tracking, PIE controls, broker integration.

**Core Layers** (bottom-up):

#### Layer 1: Broker Adapter Plugins
- **Responsibility**: Translate SmartTrade → Broker API calls
- **Implementations**: 
  - `FyersAdapter` (live broker integration)
  - `MockAdapter` → **Paper Broker Service** (paper trading backend; see [`paper-broker-service/docs/Paper_Broker_Service_HLD_v1.md`](../paper-broker-service/docs/Paper_Broker_Service_HLD_v1.md))
  - (future: Zerodha, IBKR, Alpaca)
- **Patterns**: Plugin architecture; adapter interface standardized
- **Design Note**: PBS behaves like external broker (no SmartTrade event bus access); BAS translates WebSocket updates to order.*.v1 events

#### Layer 2: Broker Session Management
- **Classes**: `UserBrokerSession`, `UserBrokerAccountSession`
- **Responsibility**: Per-user, per-account broker authentication
- **Persistence**: Encrypted token storage in DB

#### Layer 3: Position Management
- **Class**: `PositionManagementService`
- **Responsibility**: Position aggregation, P&L calculation, position graphs
- **Key Feature**: Relationship tracking (legs, spreads, hedges)
- **Output**: Real-time position updates via WebSocket

#### Layer 4: Risk Engine
- **Class**: `RiskEngine`
- **Responsibility**: Position limits, daily loss limits, per-trade risk
- **Features**: YAML-driven rules, real-time breach detection
- **Integration**: Blocks risky orders; triggers alerts

#### Layer 5: Portfolio Engine (Phase 2-3 Refactor)
- **Current State**: ⚠️ **MIXED WITH POSITION ENGINE** (portfolio_service.py lives in BAS, not separated)
- **Target Responsibility** (Phase 2-3): Separate from Position Engine
  - Total MTM (mark-to-market) tracking
  - Global stop-loss (portfolio-level)
  - Capital allocation across strategies
  - Exposure management (sector, instrument type)
  - Portfolio-level Greeks (when Phase 2 Greeks available)
- **Output**: Portfolio summary API, events on state change
- **Refactor Note**: Extract PortfolioService from BAS into separate logical layer; currently conflated with position tracking

#### Layer 6: Execution Layer
- **Execution Orchestrator** (Phase 2): Centralized order queue (asyncio actor model)
- **Order State Machine** (Phase 1): Formal states (pending → accepted → filled → settled)
- **Idempotency Handler** (Phase 1): Request deduplication via idempotency keys
- **Audit Logger**: Immutable log of all operations
- **Retry & Resilience**: Exponential backoff, circuit breaker for broker calls

#### Layer 7: PIE (Position Intelligence Engine) (Phase 4 Refactor Needed)
- **Current Responsibility**: Auto-entry, kill-switch, rule triggering (defensive position controls)
- **Components**: `AutoEntryService`, `KillSwitchService`, `ActionOrchestrator` (~1,042 lines in BAS)
- **Integration**: Consumes market data → evaluates rules → triggers actions
- **⚠️ ARCHITECTURAL DEBT**:
  - PIE currently tightly coupled in BAS (1,042 lines of business logic)
  - Overlaps with Phase 4 Strategy Engine (both do DSL evaluation, trigger detection, state tracking)
  - Action logging should be event-driven (publishes `action.executed.v1` → Journal Service) not stored in BAS DB
  - **Phase 4 Pre-Work**: Extract PIE to separate boundary (pie-service or refactor to pure Policy Engine)
    - Keep: Position-based rules (kill-switch on loss, take-profit)
    - Defer to Strategy: User-defined algo entry/exit
    - Move: Action logs to event-driven model

#### Layer 8: Broker Sync Engine (Phase 3 - NEW)
- **Responsibility**: Reconciliation, external trade capture, drift detection
- **Features**:
  - Polling + WebSocket hybrid model
  - External trade detection (trades entered outside SmartTrade)
  - Drift classification (LOW/MEDIUM/HIGH)
  - Selective strategy pause (symbol-level, not global)
  - Reconciliation strategy (broker as source of truth)

**Routes** (HTTP API):
```
POST   /api/v1/orders/place         — Create order
DELETE /api/v1/orders/{id}          — Cancel order
PUT    /api/v1/orders/{id}          — Modify order
GET    /api/v1/orders               — List orders
GET    /api/v1/orders/{id}          — Get order detail

GET    /api/v1/positions            — Get all positions
GET    /api/v1/positions/{symbol}   — Get position detail
GET    /api/v1/portfolio            — Portfolio summary

GET    /api/v1/risk/daily-loss      — Daily loss status
GET    /api/v1/risk/exposure        — Exposure summary

POST   /api/v1/broker_sync/ingest   — Ingest external trade (Phase 3)
GET    /api/v1/broker_sync/drift    — Drift detection results (Phase 3)

POST   /api/v1/strategies           — Create strategy (Phase 4)
PUT    /api/v1/strategies/{id}      — Update strategy
GET    /api/v1/strategies           — List strategies

WS     /api/v1/ws                   — Account event stream (orders, trades, positions)
```

---

### 3.3 Market Data Service (Port 8004)

**Responsibility**: Real-time quotes, instrument resolution, trading calendar, OHLC aggregation, IV calculation, backtest data feed.

**Core Principle (v2.1 Production Hardening)**: Deterministic, idempotent, memory-safe market data platform.

**Implemented Components**:
1. **Instrument Service** ✅ — Lookup, resolution, broker mapping
2. **Quote Service** ✅ — Real-time quote streaming, WebSocket fan-out
3. **Trading Calendar** ✅ — Market holidays, session times, expected_candles_per_day
4. **Event Publishing Layer** ✅ — DomainEventPublisher integration

**Phase 0-4 Implementation (Q2 2026, Parallel to Frontend Phase 5)**:

**Phase 0: Foundation** ✅ READY
- **Bucket-Scoped Tick Buffer**: Per-(symbol, bucket_start) isolation; prevents cross-bucket contamination
- **Time-Driven Finalization Scheduler**: Runs every 60 seconds; closes buckets by wall-clock time (not tick arrival)
- **Memory Management Service**: TTL cleanup for finalized buckets (10-minute retention); prevents unbounded growth
- **Deterministic Idempotency**: SHA256 keys (symbol:timestamp:price:volume) stable across processes
- **PostgreSQL Schema**: Partitioned historical_candles table with idempotency_key UNIQUE constraint

**Phase 1: Historical Data Feed**
- Broker daily OHLC backfill (Fyers, Paper Broker)
- Gap detection & logging (missing candles, broker outages)
- Data quality metrics & validation (outliers, volume spikes)

**Phase 2: Multi-Interval Derivation**
- 5m/15m/1h/1d derivation from 1m candles
- Exact boundary conditions (watermark-based finalization)
- Idempotent derived candle inserts

**Phase 3: IV & Greeks Enhancement**
- Config-driven IV calculation (Black-Scholes, no hardcoded params)
- Real-time IV surface updates (on quote refresh)
- Multi-leg Greeks support (option spreads)

**Phase 4: Backtest Data Feed**
- Backtest API: `/api/v1/data/ohlc?symbol=SBIN-EQ&interval=5m&from=2025-01-01`
- Replay cursor abstraction (seek, peek, progress)
- Corporate action application (dividends, splits, bonus)

**Routes (Real-Time Implemented)**:
```
GET    /api/v1/instruments           — Lookup instruments
GET    /api/v1/instruments/{symbol}  — Get instrument detail
GET    /api/v1/data/quotes/{broker_id}  — Get quote snapshot

WS     /ws                           — Real-time market data (quotes, depth) ONLY
```

**Routes (Phase 0-4 Roadmap)**:
```
GET    /api/v1/data/ohlc             — Get historical OHLC candles (Phase 1)
GET    /api/v1/greeks                — Get Greeks for option (Phase 3)
GET    /api/v1/options/{symbol}      — Get option chain (Phase 3)
GET    /api/v1/iv_metrics/{symbol}   — Get IV surface (Phase 3)
WS     /ws/greeks                    — Real-time Greeks updates (Phase 3)
```

**Events Published**:
- `market.quote.v1` ✅ (real-time price)
- `market_data.candle.finalized.v1` 🔄 (1m/5m/15m/1h/1d candles, deterministic, with idempotency_key)
- `market_data.candle_gap.v1` 🔄 (missing candles, severity, context)
- `market_data.iv.calculated.v1` 🔄 (option IV, config-driven, with quality flags)

**Candle Event Schema (v2.1)**:
```python
class CandleFinalizedV1(BaseEvent):
    symbol: str
    interval: str  # "1m", "5m", "15m", "1h", "1d"
    timestamp: datetime  # UTC
    open, high, low, close: Decimal  # Precise, not float
    volume: int
    tick_count: int
    
    # v2.1 additions
    source: str  # "live", "derived", "broker_daily", "broker_1m_backfill"
    version: int  # Schema version (allows future upgrades)
    is_complete: bool  # False if gap detected
    confidence: str  # "high", "medium", "low"
    quality_flags: List[str]  # ["outlier_price"], ["volume_spike"], []
    
    # Audit trail
    created_at: datetime
    updated_at: datetime
    last_corrected_at: datetime | None
    
    # Idempotency (critical for distributed safety)
    idempotency_key: str  # SHA256(symbol:interval:timestamp), stable across processes
```

**Production Guarantees (v2.1)**:
- ✅ **Deterministic**: Same ticks in any order → identical candle (verified 10× in tests)
- ✅ **Idempotent**: All operations safe to retry; deduplication via DB UNIQUE constraints
- ✅ **Distributed-Safe**: Idempotency keys stable across processes; no process-local state
- ✅ **Memory-Safe**: No unbounded growth; TTL cleanup every 10 minutes
- ✅ **Resilient**: Circuit breaker on broker outages; rate limiting (10K ticks/sec); graceful degradation

**Consumption Model** (NOT Production Push):
- BAS consumes MDS events via Redis Streams (durable, ordered, idempotent)
- Strategy Service consumes via backtest API (replay cursor for deterministic backtesting)
- Frontend consumes WebSocket for real-time (best-effort, not durable)

#### MDS Service Boundaries (Strict Separation of Concerns)

**MDS OWNS** ✅:
- Real-time quote ingestion & WebSocket fan-out
- Instrument resolution & broker mapping
- Trading calendar (market hours, holidays, expected_candles)
- 1m candle aggregation from ticks (bucket-scoped buffering)
- Multi-interval derivation (5m/15m/1h/1d from 1m)
- IV calculation (Black-Scholes, config-driven)
- Backtest data feed (historical OHLC with gap detection)
- Candle versioning & source tracking
- Gap detection & severity classification

**MDS DOES NOT OWN** ❌:
- Trading events (orders, positions, trades) → **BAS responsibility**
- Signal generation (indicators, patterns) → **Strategy Service responsibility**
- Risk calculation (limits, loss, exposure) → **BAS Risk Engine responsibility**
- User authentication → **Auth Service responsibility**
- Execution state (which orders filled, what's pending) → **BAS Order State Machine**

**Critical: NO Mutation of Historical Data**
- Late ticks: Only `discard` or `log_only` policies allowed at runtime
- Corrections: Require explicit versioning mechanism + DB migration (not automatic)
- Determinism: Same candle input must always produce identical OHLC (no retroactive adjustments)

#### 3.3.1 Market Data Distribution Architecture (Redis Streams + KV Pattern v3.5+)

**Problem Solved**: BAS and PBS currently receive quotes via per-user WebSocket clients connected to MDS. This violates MDS ownership, lacks durability/replayability, and creates fragile distributed state. MDS must publish a durable, ordered, idempotent event stream that all backend services consume from.

**Solution: Dual-Channel Redis Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│ MDS (Market Data Service)                                   │
│ Ingests broker WebSocket, normalizes                        │
└────────────────┬────────────────────────────────────────────┘
                 │ On each price tick (normalized):
                 ├─→ (1) Write Redis KV: quote:{instrument_id}
                 │       TTL=60s, value={ ltp, bid, ask, ts, seq# }
                 │
                 └─→ (2) Publish Redis Stream: market.quote.v1
                         Fields={ instrument_id, ltp, bid, ask, ts, seq# }
                         
┌──────────────────────────────────────────────────────────────┐
│ Redis (Dual Store)                                           │
│                                                              │
│ KV (Latest Snapshot):                                        │
│   Key: quote:{instrument_id}                                │
│   Value: { instrument_id, ltp, bid, ask, timestamp,         │
│            sequence_number }                                 │
│   TTL: 60s                                                   │
│   Purpose: O(1) freshness check; quick snapshot reads        │
│                                                              │
│ Stream (Ordered Event Log):                                  │
│   Key: market.quote.v1                                 │
│   Fields: { instrument_id, ltp, bid, ask, timestamp,        │
│             sequence_number }                                │
│   Retention: 90 days (XTRIM)                                │
│   Purpose: Durable, ordered, replay-safe event log         │
└──────────────────┬───────────────┬──────────────┬───────────┘
                   │               │              │
        ┌──────────┘       ┌───────┘      ┌──────┘
        │                  │              │
        ▼                  ▼              ▼
    ┌─────────┐      ┌─────────┐   ┌─────────┐
    │ BAS     │      │ PBS     │   │ Strategy│
    │ Consumer│      │Consumer │   │(Backtest)
    │ Group   │      │ Group   │   │ API    │
    │bas-*   │      │pbs-*   │   │        │
    └─────────┘      └─────────┘   └─────────┘
    
Consumer Algorithm (BAS & PBS):
  while running:
    entries = XREADGROUP(
      group="bas-quote-consumer",
      consumer="bas-{hostname}",
      streams={"market.quote.v1": ">"},
      block=100ms
    )
    for msg_id, fields in entries:
      instrument_id = fields["instrument_id"]
      seq = int(fields["sequence_number"])
      
      # Idempotency: skip if already processed
      if seq <= last_seen[instrument_id]:
        XACK(stream, group, msg_id)
        continue
      
      # Freshness decision: stream event vs KV snapshot
      event_ltp = Decimal(fields["ltp"])
      event_ts = iso_to_datetime(fields["timestamp"])
      lag_ms = (now_utc - event_ts).total_seconds() * 1000
      
      if lag_ms > MAX_ALLOWED_LAG_MS:
        # Event is stale; use KV snapshot
        kv_ltp = KV_GET(f"quote:{instrument_id}")["ltp"]
        selected_price = kv_ltp or event_ltp
      elif abs(kv_ltp - event_ltp) / event_ltp > PRICE_THRESHOLD:
        # Divergence detected; prefer freshness (KV)
        selected_price = kv_ltp or event_ltp
      else:
        # Event is fresh and aligned; use it
        selected_price = event_ltp
      
      # Update local cache
      quote_store.update(instrument_id, selected_price, now_utc)
      
      # Mark processed
      last_seen[instrument_id] = seq
      XACK(stream, group, msg_id)
```

**Config Constants**:
| Constant | Value | Rationale |
|----------|-------|-----------|
| `MAX_ALLOWED_LAG_MS` | 100 | If event > 100ms old, prefer KV (more current) |
| `PRICE_THRESHOLD_PERCENT` | 0.1 | If divergence > 0.1%, prefer KV (drift protection) |
| `STREAM_RETENTION_DAYS` | 90 | Replay window for debugging; trading events kept |
| `KV_TTL_SECONDS` | 60 | Quote snapshot stale after 1 minute; forces stream read |
| `CONSUMER_BLOCK_MS` | 100 | Balances latency vs. CPU polling |

**Sequence Number Generation (per Instrument)**:
- Redis INCR on key `seq:{instrument_id}` at MDS publication time
- Strictly increasing, persists across MDS restarts
- Enables idempotency: skip duplicate fills with same seq#
- Example: INCR `seq:SBIN-EQ` → 42101, 42102, 42103 ...

**Guarantees**:
1. **Durable**: All quotes in Redis Stream (persists Redis restarts)
2. **Ordered**: Stream maintains FIFO per key; idempotency via seq# prevents duplicates
3. **Idempotent**: seq# + consumer group ACK pattern prevents double-processing
4. **Replay-Safe**: Read stream from beginning; same seq# will be skipped (idempotent)
5. **Freshness-Protected**: KV fallback ensures stale event prices are not used
6. **Single Source of Truth**: MDS is sole publisher; BAS/PBS are read-only consumers

**Anti-Patterns** (FORBIDDEN):
- ❌ BAS/PBS connecting directly to broker WebSocket for quotes
- ❌ Backend services opening WebSocket to MDS for quote data
- ❌ Bypassing Redis Stream; reading KV directly without stream ACK
- ❌ Publishing to stream before KV write completes (violates atomicity)
- ❌ Prices as float (use Decimal in KV; Decimal string in Stream JSON)
- ❌ Missing sequence_number in any quote event

---

### 3.4 Strategy Service (Port 8006 - FUTURE, Phase 4)

**Responsibility**: User-defined algo trading, backtesting, strategy state management.

**Components**:
1. **Signal Engine**: Pattern detection, indicator calculation
   - Indicators: RSI, MACD, Bollinger Bands, Moving Averages, VWAP
   - Options: IV rank, PCR ratio, OI trends
   - Events: Expiry approaching, opening range, news
2. **Strategy Engine**: DSL evaluation, trigger detection, state tracking
3. **Strategy Runtime Engine**: Execution actor pool, isolated strategy execution
4. **Backtest Engine**: Historical data replay, performance metrics, slippage simulation

**DSL Example**:
```
if (price > SMA(20)) and (RSI < 70) and (volume > 100K)
  then place_order(qty=10, type=market, limit=0.5%)
```

**Routes** (Phase 4):
```
POST   /api/v1/strategies            — Create strategy
PUT    /api/v1/strategies/{id}       — Update strategy
GET    /api/v1/strategies            — List strategies
DELETE /api/v1/strategies/{id}       — Delete strategy

POST   /api/v1/backtest              — Run backtest
GET    /api/v1/backtest/{id}         — Get backtest results

WS     /ws/strategy/{id}             — Real-time strategy state
```

---

### 3.5 AI Orchestrator Service (Port 8007 - NEW in v3.4, Phase 5)

**Responsibility**: Generate AI-powered trade insights, scoring, and advisory recommendations. **ADVISORY ONLY — does not execute trades**.

**Strict Constraint**: AI MUST NOT:
- ❌ Call broker APIs
- ❌ Manipulate execution layer
- ❌ Bypass risk controls
- ❌ Trigger trades directly
- ✅ CAN: Generate scores, insights, warnings for UI display

**Inputs** (consumes from event bus):
- `order.placed.v1`, `order.filled.v1`, `order.cancelled.v1` — Order execution
- `trade.filled.v1`, `trade.closed.v1` — Completed trades
- `position.changed.v1` — Position updates
- `strategy.signal.v1` — Strategy triggers
- `risk.breach.v1` — Risk violations
- `journal.entry.created.v1` — User trade annotations

**Outputs** (publishes to event bus):
- `ai.trade.score.v1` — Score (0-100) for historical trade
- `ai.nudge.v1` — Gentle suggestion (e.g., "Consider tightening stop-loss")
- `ai.recommendation.v1` — Strategy suggestion (e.g., "Similar pattern worked 75% of time")
- `ai.warning.v1` — Risk alert (e.g., "Similar setup led to 10% drawdowns")

**Components**:
1. **Feature Extractor**: Convert trades → ML features (volatility, momentum, correlation, etc.)
2. **Embeddings Store**: Store trade embeddings for similarity search
3. **Pattern Matcher**: Find historical similar trades
4. **LLM Interface**: Query Claude for insights
5. **Scoring Engine**: Score trades on profitability, risk-adjusted returns, execution quality

**API Routes** (Phase 5):
```
GET    /api/v1/ai/trade/{trade_id}/score       — Get trade score
GET    /api/v1/ai/insights/{symbol}            — Get insights for symbol
GET    /api/v1/ai/recommendations/{strategy}   — Get recommendations for strategy
GET    /api/v1/ai/history                      — Get historical patterns

WS     /ws/ai/insights                         — Real-time AI insights stream
```

**Example: Trade Scoring**:
```python
class AIOrchestrator:
    async def score_trade(self, trade: Trade) -> TradeScore:
        """Generate score for a closed trade"""
        # Extract features from trade
        features = await self.feature_extractor.extract(trade)

        # Find similar trades in history
        similar_trades = await self.embeddings_store.find_similar(
            features,
            limit=10
        )

        # Calculate metrics
        win_rate = sum(1 for t in similar_trades if t.pnl > 0) / len(similar_trades)
        avg_pnl = sum(t.pnl for t in similar_trades) / len(similar_trades)

        # LLM insight (optional)
        insight = await self.llm.query(f"""
            Trade setup: {trade.setup}
            Win rate: {win_rate}
            Avg P&L: {avg_pnl}
            Provide brief insight.
        """)

        return TradeScore(
            trade_id=trade.id,
            overall_score=calculate_score(win_rate, avg_pnl),
            insight=insight,
            similar_count=len(similar_trades),
            timestamp=datetime.utcnow()
        )
```

**Integration with Journal Service**:
- AI Orchestrator scores trades
- Scores and insights stored in Journal Service
- Users can query insights, similar patterns

**Non-Trading Guarantee**:
- ✅ AI observes execution flow
- ✅ AI suggests improvements
- ✅ Frontend displays AI recommendations
- ❌ AI never triggers execution
- ❌ If AI wants to "trade", it publishes event that frontend ignores

---

### 3.5a WebSocket Domain Separation Principle

**Architecture Decision** (April 2026): Frontend connects to TWO independent WebSocket streams, each owned by its service.

**Principle**: Each service owns its WebSocket delivery responsibility. No service relays another service's events.

#### 1. MDS WebSocket (`ws://mds:8004/ws`) — Market Data Only
- **Ownership**: Market Data Service
- **Content**: Real-time quotes, depth, market indices
- **Scope**: Public (all subscribed users)
- **Authentication**: JWT with market data scope
- **Data Source**: FyersDataSocket (direct connection, no relay)

#### 2. BAS WebSocket (`ws://bas:8005/api/v1/ws`) — Account Events Only
- **Ownership**: Broker Adapter Service
- **Content**: User-specific execution events (orders, trades, positions)
- **Scope**: User-isolated (only that user's events, enforced by JWT user_id claim)
- **Authentication**: JWT with user_id claim
- **Data Source**: EventBus → BAS WebSocket consumer (sequence-guaranteed delivery)
- **Ordering**: Strictly monotonic sequence numbers per user+account partition
- **Replay Protocol**: Client sends `last_sequence` on reconnect; server replays missed events

#### 3. Domain Ownership Principle
- **MDS owns market data delivery**: Quotes, depth, candles
- **BAS owns execution event delivery**: Orders, trades, positions
- **No relay**: MDS does NOT relay BAS events; BAS does NOT relay market data
- **Clear boundaries**: Each service responsible for its own WebSocket

#### 4. Benefits of Separation
| Aspect | Benefit |
|--------|---------|
| **Failure Isolation** | Market data resilient to trading outages; trading events resilient to market data outages |
| **Latency** | Trading events reach UI directly (1 hop) instead of via MDS relay (2 hops) |
| **Scalability** | Each stream scales independently; different SLAs per domain |
| **Ordering Guarantees** | BAS WebSocket provides sequence numbers for replay; MDS provides best-effort ordering |
| **Architecture** | Aligns with industry standard (Interactive Brokers, TD Ameritrade, Fyers all do this) |
| **Clarity** | Event ownership matches WebSocket ownership; no ambiguity about data source |

#### 5. Migration Path (Phased)
- **Phase 1 (Current)**: Introduce BAS WebSocket alongside existing patterns; both active
- **Phase 2 (Frontend migration)**: All account event consumers migrate to BAS WebSocket
- **Phase 3 (Deprecate)**: MDS no longer consumes order/trade/position events
- **Phase 4 (Cleanup)**: Remove relay code from MDS; MDS pure market data

---

### 3.6 Notification Service (Port 8008 - NEW in v3.4, Phase 5)

**Responsibility**: Deliver alerts and notifications via multiple channels (WebSocket, push, email) asynchronously.

**Constraint**: Must be non-blocking and never impact trading flow.

**Inputs** (consumes from event bus):
- `ai.nudge.v1`, `ai.warning.v1` — AI recommendations
- `risk.breach.v1` — Risk violations
- `order.*.v1` — Order state changes
- `position.changed.v1` — Position updates
- `system.error.v1`, `system.alert.v1` — System alerts

**Outputs**:
- `notification.sent.v1` — Successfully delivered
- `notification.failed.v1` — Failed delivery (retry queued)

**Channels**:
1. **WebSocket** (real-time, for connected clients)
   - Instant UI notifications
   - Bell icon badge, toast alerts

2. **Push Notifications** (mobile apps)
   - Risk breaches, critical alerts
   - Order fills, large position changes

3. **Email** (persistent, for historical reference)
   - Daily summary of trades
   - Important alerts
   - Weekly insights

4. **SMS** (critical only, opt-in)
   - Portfolio-level kill switch triggered
   - Extreme risk breach

**Configuration** (per user):
```yaml
notification_preferences:
  channels:
    websocket: true      # Always enabled for web app
    push: true
    email: true
    sms: false

  rules:
    ai_nudge: "websocket+email"
    risk_breach: "websocket+push+email"
    order_fill: "websocket"
    portfolio_alert: "websocket+push+sms"

  frequency:
    email_daily_summary: true
    email_weekly_insights: true
    max_notifications_per_hour: 100  # Prevent spam
```

**Implementation**:
```python
class NotificationService:
    """Async notification delivery"""

    async def on_risk_breach(self, event: RiskBreachEvent):
        """Handle risk breach notification"""
        # Queue to avoid blocking trading flow
        await self.notification_queue.put({
            "event_type": "risk_breach",
            "user_id": event.user_id,
            "severity": "critical",
            "message": f"Daily loss limit reached: {event.loss}%"
        })

    async def deliver_notifications(self):
        """Background worker: deliver queued notifications"""
        while True:
            notification = await self.notification_queue.get()
            try:
                # Get user preferences
                prefs = await self.user_prefs.get(notification["user_id"])

                # Deliver via configured channels
                if prefs.websocket:
                    await self.websocket_deliver(notification)
                if prefs.push:
                    await self.push_deliver(notification)
                if prefs.email:
                    await self.email_deliver(notification)

                # Publish success event
                await publish_event("notification.sent.v1", notification)
            except Exception as e:
                # Publish failure event
                await publish_event("notification.failed.v1", {
                    **notification,
                    "error": str(e)
                })
                # Retry with exponential backoff
                await self.retry_queue.put(notification)
```

**Monitoring**:
- Track notification delivery success rate per channel
- Alert if email delivery drops below 95%
- Monitor queue depth (should be <100)

---

### 3.7 Journal Service (Port 8009 - EXPANDED in v3.4)

**Responsibility**: Store complete trade history, user notes, and behavioral analytics for learning and compliance.

**Inputs**:
- `trade.filled.v1` — Completed trades
- `trade.closed.v1` — Closed positions
- `ai.trade.score.v1` — AI trade scores
- `ai.recommendation.v1` — AI recommendations

**Outputs**:
- `journal.entry.created.v1` — Trade recorded
- `journal.updated.v1` — User annotation added
- `journal.insight.generated.v1` — Pattern analysis complete

**Stored Data** (immutable core):
```python
class JournalEntry(BaseModel):
    # Immutable trade execution
    trade_id: str
    order_id: str
    symbol: str
    side: Literal["BUY", "SELL"]
    quantity: Decimal
    entry_price: Decimal
    exit_price: Decimal
    pnl: Decimal
    pnl_percent: Decimal
    duration: timedelta

    # Execution context
    executed_at: datetime
    closed_at: datetime
    strategy_id: Optional[str]

    # User annotations (mutable)
    user_notes: str = ""
    tags: List[str] = []

    # AI analysis (read-only for user)
    ai_score: Optional[float]
    ai_insights: Optional[str]

    # Analytics
    execution_quality: Optional[float]  # 0-100
    risk_adjusted_return: Optional[float]
    historical_similarity: Optional[str]
```

**APIs**:
```
GET    /api/v1/journal/trades                     — List trades
GET    /api/v1/journal/trades/{trade_id}          — Get trade detail
PUT    /api/v1/journal/trades/{trade_id}/notes    — Add user notes (only field that's mutable)
GET    /api/v1/journal/insights                   — Get analyzed patterns
GET    /api/v1/journal/stats                      — Win rate, avg P&L, etc.

POST   /api/v1/journal/trades/{trade_id}/tags     — Add tags ("good_setup", "rushed", etc.)
GET    /api/v1/journal/tags                       — List all tags
GET    /api/v1/journal/tags/{tag}                 — Find trades by tag

WS     /ws/journal/new-insights                   — Real-time pattern updates
```

**Immutability Guarantee**:
- ❌ User CANNOT edit trade execution data (price, quantity, date)
- ❌ User CANNOT delete trades
- ✅ User CAN add/edit personal notes
- ✅ User CAN add tags for categorization
- ✅ AI can generate insights

**AI Learning Integration**:
```python
class JournalService:
    async def generate_insights(self, user_id: str) -> Insights:
        """Analyze user's trading history for patterns"""
        trades = await self.get_closed_trades(user_id)

        # Find patterns
        profitable_setups = self.find_winning_patterns(trades)
        losing_setups = self.find_losing_patterns(trades)

        # Calculate statistics
        stats = {
            "win_rate": sum(1 for t in trades if t.pnl > 0) / len(trades),
            "avg_win": sum(t.pnl for t in trades if t.pnl > 0) / len([t for t in trades if t.pnl > 0]),
            "avg_loss": sum(t.pnl for t in trades if t.pnl < 0) / len([t for t in trades if t.pnl < 0]),
            "profit_factor": total_wins / abs(total_losses)
        }

        # Store insights
        await self.db.insights.create({
            "user_id": user_id,
            "profitable_setups": profitable_setups,
            "losing_setups": losing_setups,
            "stats": stats,
            "generated_at": datetime.utcnow()
        })

        # Publish event
        await publish_event("journal.insight.generated.v1", {
            "user_id": user_id,
            "insight_type": "pattern_analysis",
            "findings": len(profitable_setups) + len(losing_setups)
        })
```

**Compliance & Export**:
```
GET    /api/v1/journal/export?format=csv&start_date=...&end_date=...
→ Export all trades for compliance/taxes
```

---

### 3.8 Service Boundaries & Ecosystem Integration (NEW in v3.4)

**Principle**: Trading execution (core) remains isolated from advisory, notifications, and analytics (periphery).

#### Event Flow: From Execution to Insights

```
Core Trading Flow (synchronous)
┌────────────────────────────────┐
│ User places order              │
│ → Execution Orchestrator       │
│ → Risk Engine validation       │
│ → Broker submission            │
│ → order.placed event           │
└────────────┬───────────────────┘
             │
             ▼ (asynchronous event subscriptions)
┌────────────────────────────────────────────────────────────┐
│ Advisory & Support Layer (non-blocking)                    │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  order.placed.v1 ──┐                                       │
│                    ├──→ AI Orchestrator                     │
│                    │     ├─→ Score trade                    │
│  trade.filled.v1 ──┤     └─→ Find patterns                 │
│                    │        ai.trade.score.v1              │
│  strategy.signal ──┤        ai.recommendation.v1           │
│                    │                                        │
│                    ├──→ Notification Service               │
│  ai.nudge.v1 ──────┤     ├─→ WebSocket delivery           │
│  risk.breach.v1 ───┤     ├─→ Push notification            │
│                    │     └─→ Email summary                │
│                    │        notification.sent.v1          │
│                    │                                       │
│                    └──→ Journal Service                    │
│                        ├─→ Record trade                    │
│  ai.score.v1 ──────────┤    ├─→ Store execution data       │
│  ai.recommendation ────┤    ├─→ AI insights               │
│                        └─→ Provide learning dataset       │
│                           journal.insight.generated.v1    │
└────────────────────────────────────────────────────────────┘
```

**Critical Guarantee**: Advisory layer CANNOT affect core trading flow.

#### Service Isolation Rules

| Layer | Services | Can Do | Cannot Do |
|-------|----------|--------|-----------|
| **Core** | Execution, Risk, Broker | Execute trades, validate risk, block orders | Call AI, modify notifications |
| **Advisory** | AI Orchestrator | Score trades, find patterns, advise | Place orders, call broker, bypass risk |
| **Support** | Notification, Journal | Deliver alerts, store history, learn | Affect trading flow, modify execution |

#### Enforcement Mechanisms

**1. Import Restrictions** (code-level):
```python
# ❌ AI Orchestrator CANNOT import these
from broker_adapter_service.execution import ExecutionOrchestrator
from broker_adapter_service.broker_adapter import BrokerAdapter
from broker_adapter_service.risk_engine import RiskEngine

# ✅ AI Orchestrator CAN import these
from smarttrade_common.events import publish_event, subscribe_event
from smarttrade_common.database import AsyncSession
```

**2. Event Bus Permissions** (infrastructure-level):
```
Kafka topic ACLs:
├─ BAS can write to: order.*.v1, position.*.v1, trade.*.v1
├─ BAS WS consumer can read from: order.*.v1, trade.*.v1, position.*.v1
├─ AI can read from: order.*.v1, trade.*.v1, strategy.*.v1
├─ AI can write to: ai.*.v1
├─ Notification can read from: ai.*.v1, risk.*.v1, order.*.v1
├─ Notification can write to: notification.*.v1
├─ MDS CANNOT read from: order.*.v1, trade.*.v1, position.*.v1 (no relay responsibility)
└─ Journal can read from: trade.*.v1, ai.*.v1
    Journal can write to: journal.*.v1
```

**3. API Routing** (gateway-level):
```
/api/v1/orders/place
  → Only Execution Orchestrator can accept this
  → AI cannot send requests to this endpoint

/api/v1/ai/insights
  → Only frontend can consume this
  → This does NOT trigger trading
```

---

## 4. Execution Layer (Heart of the System)

### 4.1 Execution Orchestrator (Phase 2)

**Purpose**: Centralized, serialized order execution queue ensuring idempotency, determinism, and audit trails.

**Architecture**:
```
HTTP Request: POST /api/v1/orders/place {order_data}
       │
       ▼
Route Handler (thin)
  └─ Validate schema
  └─ Extract user_id, idempotency_key
  └─ Enqueue to Execution Orchestrator
       │
       ▼
Execution Orchestrator (asyncio queue per user)
  └─ Message 1: (priority=1, order_place)
  └─ Message 2: (priority=0, broker_fill)  [fills have priority]
  └─ Message 3: (priority=1, order_place)
       │
       ▼ Process Messages Sequentially
Risk Engine Validation
  └─ Check daily loss, position limits, per-trade risk
  └─ Reject if unsafe
       │
       ▼
Idempotency Check
  └─ Query DB: "Has idempotency_key been used?"
  └─ If yes: Return cached result
  └─ If no: Proceed to execution
       │
       ▼
Broker Adapter Call
  └─ Call FyersAdapter.place_order()
  └─ Handle timeout, network error with retry
       │
       ▼
State Persistence
  └─ Update order state in DB
  └─ Store result for idempotency
       │
       ▼
Event Publication
  └─ Publish order.placed event
  └─ Position engine, risk monitor subscribe
       │
       ▼
HTTP Response to Client
  └─ Return order ID, state, acceptance timestamp
```

**Guarantees**:
- ✅ No duplicate orders from network retries (idempotency)
- ✅ All orders flow through risk engine (no bypass)
- ✅ Serialized per-user (no race conditions)
- ✅ Immutable audit log of every operation
- ✅ Retry with exponential backoff on broker failures
- ✅ Durable queue: messages persisted before processing (FIX-03)
- ✅ Fault-tolerant: on restart, pending messages automatically replayed (FIX-03)
- ✅ Exactly-once semantics: idempotency + durability prevent duplicate execution (FIX-03)

**Implementation**:
```python
class ExecutionOrchestrator:
    """Per-user execution queue (asyncio-based)"""

    def __init__(self, user_id: str, db_session, broker_adapter):
        self.user_id = user_id
        self.queue = asyncio.PriorityQueue()  # (priority, message)
        self.db = db_session
        self.broker = broker_adapter

    async def enqueue(self, message: Message, priority: int = 1):
        await self.queue.put((priority, message))

    async def start(self):
        """Main event loop: process messages sequentially"""
        while True:
            priority, message = await self.queue.get()
            try:
                result = await self.dispatch(message)
                await self.publish_event(message.result_event(result))
            except Exception as e:
                await self.handle_error(e, message)

    async def dispatch(self, msg: Message):
        if msg.type == "place_order":
            # 1. Validate schema
            order = Order(**msg.order_data)

            # 2. Risk check
            await self.risk_engine.validate(order)

            # 3. Idempotency check
            cached = await self.idempotency.check(msg.idempotency_key)
            if cached:
                return cached

            # 4. Execute
            broker_result = await self.broker.place_order(order)

            # 5. Store idempotency result
            await self.idempotency.store(msg.idempotency_key, broker_result)

            # 6. Persist to DB
            order.broker_order_id = broker_result["order_id"]
            order.state = "accepted"
            await self.db.orders.update(order)

            return broker_result
```

#### Execution Durability (FIX-03: Fault-Tolerance)

**Problem**: If the service crashes mid-execution, pending messages are lost.

**Solution**: Durable message queue + replay on restart.

**Implementation**:
```python
class DurableExecutionQueue:
    """Persisted execution queue (Redis Streams or database-backed)"""

    def __init__(self, user_id: str, redis_client, db_session):
        self.user_id = user_id
        self.queue_key = f"execution_queue:{user_id}"
        self.redis = redis_client
        self.db = db_session

    async def enqueue(self, message: Message, priority: int = 1):
        """Persist message before adding to in-memory queue"""
        # 1. Store in Redis Streams (or DB)
        message_id = await self.redis.xadd(
            self.queue_key,
            {"data": message.model_dump_json(), "priority": priority},
            nomkstream=False
        )
        # 2. Now add to in-memory priority queue
        await self.queue.put((priority, message))
        return message_id

    async def start(self):
        """On startup, replay pending messages"""
        # Replay messages from Redis Streams
        pending_messages = await self.redis.xrange(self.queue_key)
        for msg_id, msg_data in pending_messages:
            message = Message.parse_obj(json.loads(msg_data["data"]))
            priority = int(msg_data["priority"])
            await self.queue.put((priority, message))

        # Main loop: process messages
        while True:
            priority, message = await self.queue.get()
            try:
                result = await self.dispatch(message)
                # Remove from Redis Streams after successful processing
                await self.redis.xdel(self.queue_key, message.id)
            except Exception as e:
                # On error, message stays in Redis (will retry on next startup)
                await self.handle_error(e, message)
```

**Guarantees**:
- ✅ Messages persisted to Redis Streams/DB before processing
- ✅ On service restart, pending messages automatically replayed
- ✅ Failed messages remain in queue for manual investigation
- ✅ Exactly-once behavior: idempotency + durability prevent duplication

---

### 4.2 Order State Machine (Phase 1)

**States** (formal, enforced):
```
pending → accepted → filled → settled
     ↗ rejected ↖
          ↓ cancelled
```

**Transitions**:
| Current | Event | Next | Action |
|---------|-------|------|--------|
| pending | submit | accepted | Broker accepted |
| pending | reject | rejected | Broker rejected |
| accepted | fill | filled | All qty filled |
| accepted | partial_fill | partially_filled | Some qty filled |
| partially_filled | fill | filled | Rest filled |
| filled | settle | settled | T+1 settlement done |
| pending | cancel | cancelled | User cancelled |
| any | error | rejected | Error occurred |

**Implementation**:
```python
class OrderStateMachine:
    VALID_TRANSITIONS = {
        "pending": ["accepted", "rejected", "cancelled"],
        "accepted": ["filled", "partially_filled", "cancelled"],
        "partially_filled": ["filled", "cancelled"],
        "filled": ["settled"],
        "rejected": [],  # Terminal
        "cancelled": [],  # Terminal
        "settled": [],  # Terminal
    }

    @classmethod
    def can_transition(cls, current: str, next: str) -> bool:
        return next in cls.VALID_TRANSITIONS.get(current, [])

    @classmethod
    def transition(cls, order: Order, event: str) -> str:
        """Apply event to order state; return new state or raise exception"""
        transitions = {
            ("pending", "submit"): "accepted",
            ("pending", "reject"): "rejected",
            ("accepted", "fill"): "filled",
            ("accepted", "partial_fill"): "partially_filled",
            # ... etc
        }
        key = (order.state, event)
        if key not in transitions:
            raise InvalidStateTransition(f"Cannot go from {order.state} via {event}")
        return transitions[key]
```

---

### 4.3 Idempotency Handler (Phase 1)

**Purpose**: Prevent duplicate operations when client retries.

**Mechanism**:
```
Client generates: idempotency_key = UUID()

Request 1: POST /api/v1/orders/place
  Header: Idempotency-Key: {UUID}
  → Executes order placement
  → Stores result in DB under UUID

Request 2: (network timeout, retry) POST /api/v1/orders/place
  Header: Idempotency-Key: {UUID}
  → Checks: Is UUID in DB?
  → YES: Return cached result (same as Request 1)
  → NO: Execute and cache

Result: Client always gets same response, even if network fails
```

**Implementation**:
```python
class IdempotencyHandler:
    async def check_or_execute(self, key: str, user_id: str, operation):
        # Query: SELECT result WHERE idempotency_key = key AND user_id = user_id
        cached = await self.db.idempotency.find(key, user_id)
        if cached:
            return cached.result

        # Execute operation
        result = await operation()

        # Store result
        await self.db.idempotency.create(
            key=key,
            user_id=user_id,
            result=result,
            created_at=datetime.utcnow()
        )
        return result
```

**Routes Using Idempotency**:
- `POST /api/v1/orders/place` — Create order
- `PUT /api/v1/orders/{id}` — Modify order
- `DELETE /api/v1/orders/{id}` — Cancel order

---

## 5. Concurrency Model: Actor Model Per User

### Design

**Principle**: Single-threaded execution per user via asyncio queue.

**Architecture**:
```
SmartTrade System
├─ User 1 Actor (user_123)
│  └─ Single queue: [msg1, msg2, msg3]
│     └─ One processor, serializes all ops
│
├─ User 2 Actor (user_456)
│  └─ Independent queue
│     └─ No interference with User 1
│
└─ User N Actor
   └─ Independent queue
```

**Message Priority**:
| Priority | Type | Urgency |
|----------|------|---------|
| 0 | FILL | Broker fill (highest) |
| 0 | RISK_BREACH | Risk limit breach |
| 1 | PLACE_ORDER | User action |
| 1 | SIGNAL_MATCHED | Algo signal |
| 2 | CANCEL_ORDER | Order cancel |
| 3 | QUERY | Informational |

**Guarantees**:
```
Per-user: No race conditions, deterministic ordering
Cross-user: Independent, parallel execution
Global: Portfolio aggregation (eventual consistency)
```

### Implementation Pattern

```python
class UserActor:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.queue = asyncio.PriorityQueue()
        self.risk_engine = RiskEngine()
        self.position_engine = PositionEngine()
        # ... other engines

    async def start(self):
        """Main event loop"""
        while True:
            priority, message = await self.queue.get()
            try:
                await self.dispatch(message)
            except Exception as e:
                logger.error(f"Error processing message", exc_info=True)

    async def dispatch(self, msg: Message):
        """Process ONE message atomically"""
        if msg.type == "place_order":
            await self.risk_engine.validate(msg.order)
            await self.execution_orchestrator.submit(msg.order)
            await self.position_engine.prepare(msg.order)
        elif msg.type == "process_fill":
            await self.position_engine.update(msg.fill)
            await self.portfolio_engine.update_mtm(msg.fill)
        # All state updates ATOMIC within one message
```

---

## 6. Broker Sync & External Trade Handling (Phase 3)

### Problem

**Risk**: Trades executed outside SmartTrade (mobile app, web terminal, other traders) cause state drift.

**Solution**: Broker Sync Engine detects, ingests, reconciles external trades.

### Design

```
┌─ SmartTrade Position DB
│  ├─ SBIN-EQ: 100 shares
│  └─ RELIANCE-EQ: 50 shares
│
└─ Broker Server (Fyers)
   ├─ SBIN-EQ: 150 shares [+50 external trade]
   ├─ RELIANCE-EQ: 50 shares [match]
   └─ HDFC-EQ: 25 shares [external, not in ST]

Broker Sync Engine:
1. Query broker: "Get all positions"
2. Compare with DB: SBIN-EQ drift (+50), HDFC-EQ missing
3. Classify drift: MEDIUM (>0.1%, symbol-specific)
4. Ingest: Create Trade record for external fill
5. Update positions: SBIN-EQ 100 → 150
6. Alert: "External trade detected on SBIN-EQ"
7. Selective pause: Pause auto-entry on SBIN-EQ (not global)
```

### Implementation

#### 1. External Trade Ingestion
```python
POST /api/v1/broker_sync/ingest
Body: {
    "symbol": "SBIN-EQ",
    "side": "BUY",
    "quantity": 50,
    "price": 625.50,
    "executed_at": "2026-03-26T10:30:00Z",
    "broker_trade_id": "ext_12345"
}

Response: {
    "trade_id": 999,
    "status": "ingested",
    "message": "External trade recorded"
}
```

#### 2. Drift Detection (Daily Reconciliation Job)
```python
async def reconcile_broker_positions():
    """Run daily: compare SmartTrade DB vs Broker"""
    # Query broker for all positions
    broker_positions = await broker.get_all_positions(user_id)

    # Query SmartTrade DB
    st_positions = await position_repo.find_by_user(user_id)

    # Compare symbol-by-symbol
    for symbol, broker_qty in broker_positions.items():
        st_qty = st_positions.get(symbol, 0)
        variance = abs(broker_qty - st_qty) / max(abs(st_qty), 1)

        if variance > 0.001:  # 0.1%
            drift = {
                "symbol": symbol,
                "st_qty": st_qty,
                "broker_qty": broker_qty,
                "variance": variance,
                "classification": classify_drift(variance)
            }
            await publish_event("broker_drift.detected", drift)
            # Selective pause: block auto-entry for this symbol
            await auto_entry_service.pause_symbol(symbol)
```

#### 3. Drift Classification
| Variance | Classification | Action |
|----------|---|---|
| < 0.1% | LOW | Log only; no action |
| 0.1% - 5% | MEDIUM | Alert; pause auto-entry for symbol |
| > 5% | HIGH | Critical alert; pause ALL trading for symbol |

#### 4. Reconciliation Strategy
```python
async def reconcile_symbol(symbol: str):
    """Reconcile single symbol"""
    broker_qty = await broker.get_position(symbol)
    st_qty = await position_repo.get_qty(symbol, user_id)

    if st_qty < broker_qty:
        # External buy: ingest external trade
        external_qty = broker_qty - st_qty
        await ingest_external_trade(
            symbol=symbol,
            side="BUY",
            quantity=external_qty,
            price=await get_last_execution_price(symbol)
        )
    elif st_qty > broker_qty:
        # Data error: reconcile to broker (broker is source of truth)
        discrepancy = st_qty - broker_qty
        logger.warning(f"Position discrepancy: {symbol} {discrepancy}")
        # Manual investigation required
```

---

## 7. Algo Trading Architecture (Phase 4-5)

### Full Pipeline

```
Market Data
  (quotes) [Greeks + options chains deferred to Phase 2]
        │
        ▼
Signal Engine (Phase 5)
  ├─ Indicators (RSI, MACD, MA, VWAP, Bollinger Bands)
  ├─ Options (IV rank, PCR, OI trends) [Phase 2 - requires Greeks/IV]
  └─ Events (expiry, opening range, news)
        │
        ▼ (signal.triggered events)
Strategy Engine (Phase 4)
  ├─ DSL evaluation
  ├─ State tracking
  ├─ Trigger detection
  └─ Rule matching
        │
        ▼ (strategy.signal events)
Strategy Runtime Engine (Phase 4)
  ├─ Actor pool (per strategy)
  ├─ Execution isolation
  ├─ State persistence
  └─ Conflict resolution
        │
        ▼ (place_order message)
Execution Orchestrator (Phase 2)
  ├─ Idempotency
  ├─ Risk re-validation
  ├─ Audit logging
  └─ Broker submission
        │
        ▼
Broker (Fyers, Mock, etc.)
```

### Signal Engine (Phase 5)

**Signals**: Indicators, options metrics, event-based.

**Indicators**:
```python
class SignalEngine:
    async def evaluate_indicators(self, symbol: str, candles: List[Candle]):
        """Calculate technical indicators"""
        rsi = RSI(candles[-14:])  # 14-period RSI
        macd = MACD(candles[-26:])  # MACD
        sma_20 = SMA(candles[-20:], period=20)
        sma_50 = SMA(candles[-50:], period=50)
        bb = BollingerBands(candles[-20:], period=20, std=2)

        # Publish events
        await publish_event("signal.rsi_oversold", {
            "symbol": symbol,
            "rsi": rsi,
            "timestamp": datetime.utcnow()
        })

        if sma_20 > sma_50:
            await publish_event("signal.golden_cross", {
                "symbol": symbol,
                "sma_20": sma_20,
                "sma_50": sma_50
            })
```

**Options Signals**:
```python
async def evaluate_options(self, symbol: str):
    """Options metrics"""
    chain = await mds.get_option_chain(symbol)
    iv_rank = calculate_iv_rank(chain)
    pcr = calculate_put_call_ratio(chain)
    oi_trend = calculate_oi_trend(chain)

    if iv_rank > 80:  # High IV
        await publish_event("signal.high_iv", {
            "symbol": symbol,
            "iv_rank": iv_rank,
            "recommendation": "sell_options"
        })

    if pcr > 1.5:  # High put/call ratio
        await publish_event("signal.high_pcr", {
            "symbol": symbol,
            "pcr": pcr,
            "sentiment": "bearish"
        })
```

**Event Signals**:
```python
async def monitor_events(self, symbol: str):
    """Event-based signals"""
    if is_expiry_day(symbol):
        await publish_event("signal.expiry_approaching", {
            "symbol": symbol,
            "days_to_expiry": 1
        })

    if is_opening_range(datetime.utcnow()):
        opening_range = await calculate_opening_range()
        await publish_event("signal.opening_range", {
            "symbol": symbol,
            "range": opening_range
        })
```

### Strategy Engine (Phase 4)

**DSL Example**:
```
Strategy: "BuyOnGoldenCross"

Conditions:
  - SMA(20) > SMA(50)                [Golden cross]
  - RSI < 70                         [Not overbought]
  - Volume > 100K                    [Liquid]

Actions:
  - Place order: BUY 10 @ MARKET (0.5% limit)
  - Set stop-loss: 2% below entry
  - Set take-profit: 5% above entry

Modes:
  - manual: Wait for user confirmation
  - semi-auto: Check signal, auto-execute if user is online
  - full-auto: Auto-execute without confirmation
```

**Implementation**:
```python
class StrategyEngine:
    async def evaluate(self, strategy: Strategy, market_data: MarketData):
        """Evaluate strategy conditions"""
        conditions_met = []

        # Evaluate each condition
        for condition in strategy.conditions:
            result = await self.evaluate_condition(condition, market_data)
            conditions_met.append(result)

        # If all conditions met, trigger action
        if all(conditions_met):
            await publish_event("strategy.signal", {
                "strategy_id": strategy.id,
                "signal": "BUY",
                "timestamp": datetime.utcnow()
            })

    async def evaluate_condition(self, condition: Condition, data: MarketData):
        """Evaluate single condition"""
        if condition.type == "sma_cross":
            sma_20 = SMA(data.candles[-20:])
            sma_50 = SMA(data.candles[-50:])
            return sma_20 > sma_50
        # ... etc
```

#### Strategy Execution Boundary (FIX-05: Enforce Isolation)

**Rule**: Strategy Engine cannot call broker or execution layer directly.

**WRONG** ❌:
```python
class StrategyEngine:
    async def evaluate(self, strategy: Strategy, market_data: MarketData):
        if sma_20 > sma_50:  # Signal detected
            # ❌ DO NOT DO THIS: Direct broker call
            await self.broker.place_order(order)
```

**CORRECT** ✅:
```python
class StrategyEngine:
    async def evaluate(self, strategy: Strategy, market_data: MarketData):
        if sma_20 > sma_50:  # Signal detected
            # ✅ DO THIS: Emit event only
            await publish_event("strategy.signal", {
                "strategy_id": strategy.id,
                "action": "place_order",
                "order_data": {...},
                "timestamp": datetime.utcnow()
            })
```

**Why**:
- Prevents bypassing risk engine (order.place → risk check → broker)
- Ensures idempotency layer is always engaged
- Maintains audit trail for all executions
- Allows conflict resolution if multiple strategies signal same symbol

**Enforcement**:
- ❌ Strategy code cannot import `BrokerAdapter` or `ExecutionOrchestrator`
- ✅ Strategy code can only call `publish_event()` or `logger.info()`
- ✅ All execution flows through Execution Orchestrator only

---

### Strategy Runtime Engine (Phase 4)

**Responsibility**: Execute strategies in isolated actor pool.

**Design**:
```
Strategy Runtime Engine
├─ Strategy 1 (BUY on Golden Cross)
│  └─ Actor: Evaluates continuously
│     └─ Publishes: strategy.signal events
│
├─ Strategy 2 (Sell on High IV)
│  └─ Actor: Independent
│     └─ No interference with Strategy 1
│
└─ Conflict Resolver
   ├─ If Strategy 1 = BUY SBIN-EQ
   ├─ And Strategy 2 = SELL SBIN-EQ
   └─ Resolution: Ask user or apply priority
```

**Algo Modes**:
| Mode | Behavior | User Action |
|------|----------|-------------|
| **manual** | Signal detected; wait for user | Click "Execute" button |
| **semi-auto** | Signal detected; auto-execute if user online | Optional confirmation |
| **full-auto** | Signal detected; immediate execution | None (fire-and-forget) |

---

## 8. Portfolio & Risk Layer

### Portfolio Engine (NEW in v3.2)

**Separated from Position Engine** to manage portfolio-level constraints.

**Responsibilities**:
```
Portfolio Engine
├─ Total MTM tracking
│  ├─ Sum all positions' P&L
│  ├─ Real-time NAV calculation
│  └─ Performance metrics
│
├─ Global stop-loss
│  ├─ Daily loss limit (portfolio-level)
│  ├─ Breach detection
│  └─ Auto-liquidation if needed
│
├─ Capital allocation
│  ├─ Allocation per strategy
│  ├─ Rebalancing
│  └─ Exposure limits
│
└─ Exposure management
   ├─ Sector exposure (max %)
   ├─ Instrument type exposure (options, futures, etc.)
   └─ Leverage limits
```

**API**:
```python
class PortfolioEngine:
    async def get_portfolio_summary(self, user_id: str) -> PortfolioSummary:
        """Get total MTM, NAV, performance"""
        positions = await position_repo.find_by_user(user_id)
        mtm_total = sum(p.mtm for p in positions)
        nav = initial_capital + mtm_total
        return PortfolioSummary(
            nav=nav,
            mtm=mtm_total,
            daily_loss=mtm_total - previous_nav,
            sectors={...},
            exposures={...}
        )

    async def check_portfolio_risk(self, order: Order) -> bool:
        """Check portfolio-level constraints"""
        # 1. Would this order breach daily loss limit?
        estimated_mtm = await self.estimate_mtm_after_order(order)
        if estimated_mtm < daily_loss_limit:
            raise SmartTradeError("RISK_001", "Daily loss limit breach")

        # 2. Would this order breach sector exposure?
        sector_exposure = await self.get_sector_exposure(order.symbol)
        if sector_exposure > max_sector_exposure:
            raise SmartTradeError("RISK_002", "Sector exposure limit")

        return True  # Order is safe
```

### Risk Engine (Current)

**Responsibilities**:
```
Risk Engine
├─ Position limits
│  └─ Max position per symbol, per strategy
│
├─ Daily loss limit
│  └─ Auto-liquidate if exceeded
│
├─ Per-trade risk
│  ├─ Max risk per order
│  ├─ Bet sizing
│  └─ Stop-loss validation
│
└─ Real-time breach detection
   └─ Publish alert on breach
```

**YAML Configuration**:
```yaml
risk_rules:
  daily_loss_limit:
    percentage: 2%          # 2% of capital
    absolute_amount: 5000   # OR ₹5000
    auto_liquidate: true

  position_limits:
    max_per_symbol: 100     # Max 100 shares per symbol
    max_per_strategy: 1000  # Max 1000 per strategy
    max_total: 5000         # Max 5000 total

  per_trade_risk:
    max_risk_per_trade: 100  # Max ₹100 risk
    max_leverage: 2.0        # 2x leverage max
    min_profit_factor: 1.5   # Profit/Risk ratio
```

### Integration: Portfolio + Risk

```
Order received
    ↓
Execution Orchestrator
    ↓
Risk Engine (First layer)
  └─ Position limits, daily loss check
    ↓ [if passes]
Portfolio Engine (Second layer)
  └─ Portfolio-level constraints
    ↓ [if passes]
Broker Adapter
  └─ Submit to broker
```

**Guarantee**: Portfolio controls always apply; cannot bypass via strategy.

### Multi-Level Kill Switches (FIX-08: Granular Risk Controls)

**Problem**: Current PIE engine has binary (on/off) kill switch. Need fine-grained control.

**Solution**: Kill switches at 4 levels with cascading effect.

**Hierarchy**:
```
┌─────────────────────────────────────┐
│ PORTFOLIO-LEVEL KILL SWITCH         │
│ Pauses ALL trading for all symbols  │
│ Triggered by: Daily loss > 2%       │
└─────────────────────────────────────┘
             ↑
┌─────────────────────────────────────┐
│ STRATEGY-LEVEL KILL SWITCH          │
│ Pauses trading for single strategy  │
│ Triggered by: Strategy max loss     │
└─────────────────────────────────────┘
             ↑
┌─────────────────────────────────────┐
│ SYMBOL-LEVEL KILL SWITCH            │
│ Pauses trading for single symbol    │
│ Triggered by: Symbol max loss OR    │
│              broker drift detected  │
└─────────────────────────────────────┘
             ↑
┌─────────────────────────────────────┐
│ USER-LEVEL KILL SWITCH              │
│ Manual user override                │
│ Triggered by: User clicks "Pause"   │
└─────────────────────────────────────┘
```

**Implementation**:
```python
class KillSwitchService:
    """Multi-level kill switch control"""

    async def check_can_execute(self, order: Order) -> bool:
        """Check all kill switch levels before execution"""
        user_id = order.user_id

        # Level 1: User override (highest priority)
        if await self.get_user_kill_switch(user_id):
            raise SmartTradeError("USER_PAUSED", "Trading paused by user")

        # Level 2: Symbol-level pause
        if await self.get_symbol_kill_switch(user_id, order.symbol):
            raise SmartTradeError("SYMBOL_PAUSED", f"Trading paused for {order.symbol}")

        # Level 3: Strategy-level pause
        if order.strategy_id:
            if await self.get_strategy_kill_switch(user_id, order.strategy_id):
                raise SmartTradeError("STRATEGY_PAUSED", f"Strategy {order.strategy_id} paused")

        # Level 4: Portfolio-level pause
        if await self.get_portfolio_kill_switch(user_id):
            raise SmartTradeError("PORTFOLIO_PAUSED", "Portfolio trading paused (daily loss limit)")

        return True

    async def trigger_symbol_pause(self, user_id: str, symbol: str, reason: str):
        """Pause trading for one symbol only"""
        await self.db.kill_switches.update(
            {"user_id": user_id, "symbol": symbol, "level": "symbol"},
            {"paused": True, "reason": reason, "paused_at": datetime.utcnow()}
        )
        await publish_event("kill_switch.triggered.v1", {
            "user_id": user_id,
            "level": "symbol",
            "symbol": symbol,
            "reason": reason
        })

    async def trigger_portfolio_pause(self, user_id: str):
        """Pause ALL trading for portfolio (daily loss breach)"""
        await self.db.kill_switches.update(
            {"user_id": user_id, "level": "portfolio"},
            {"paused": True, "reason": "Daily loss limit exceeded", "paused_at": datetime.utcnow()}
        )
        await publish_event("kill_switch.triggered.v1", {
            "user_id": user_id,
            "level": "portfolio",
            "reason": "Daily loss limit exceeded"
        })
```

**Configuration**:
```yaml
kill_switches:
  symbol_level:
    trigger: "symbol_daily_loss > 1%"  # 1% loss on symbol
    action: "pause_symbol"
    duration: "until_user_reset"

  strategy_level:
    trigger: "strategy_max_loss > 5%"  # 5% loss on strategy
    action: "pause_strategy"
    duration: "until_user_reset"

  portfolio_level:
    trigger: "daily_loss > 2%"  # 2% of capital
    action: "pause_portfolio"
    duration: "until_market_close"
```

**User API**:
```
GET  /api/v1/kill_switch/status
Response:
{
  "user_level": {"paused": false},
  "symbols": [
    {"symbol": "SBIN-EQ", "paused": true, "reason": "Drift detected"},
    {"symbol": "HDFC-EQ", "paused": false}
  ],
  "strategies": [
    {"strategy_id": "buy_on_cross", "paused": false}
  ],
  "portfolio": {"paused": false}
}

POST /api/v1/kill_switch/resume
Body: {"level": "symbol", "symbol": "SBIN-EQ"}
→ Resumes trading for SBIN-EQ only

POST /api/v1/kill_switch/pause
Body: {"level": "portfolio"}
→ Pauses ALL trading
```

---

## 9. Option Chain & Multi-Asset Support (PHASE 2 - DEFERRED)

**Note**: The Option Chain Service and Greeks Calculator are deferred to Phase 2 (post-launch). Current Phase 1 focuses on equity quotes only. This section documents the target design.

### Instrument Abstraction

**Types**:
```
Instrument
├─ Equity (SBIN-EQ, TCS-EQ)
├─ Option (SBIN23500CE, SBIN23500PE)
├─ Future (SBIN24MAR, NIFTY24MAR)
├─ Crypto (BTC-USDT, ETH-USDT)  [future]
└─ Forex (EURINR, GBPINR)  [future]
```

**Metadata** (per instrument):
```python
class Instrument:
    symbol: str              # "SBIN-EQ"
    instrument_type: str     # "equity", "option", "future"
    exchange: str            # "NSE", "NFO"
    broker_symbol: dict      # {"fyers": "SBIN-EQ", "zerodha": "..."}

    # For options:
    underlying: str          # "SBIN-EQ"
    strike: Decimal          # 500
    expiry: date            # 2026-03-28
    option_type: str        # "CE", "PE"

    # For futures:
    expiry: date            # 2026-03-28
    contract_multiplier: int # 1 share, or 100 shares
```

### Option Chain Service

**Responsibilities**:
```
Option Chain Service
├─ Retrieve option chains (per underlying)
├─ Cache chains (1-hour TTL)
├─ Calculate Greeks (Black-Scholes)
├─ Calculate IV metrics
│  ├─ IV rank (percentile over 252 days)
│  ├─ IV percentile
│  └─ Volatility surface
└─ Multi-leg Greeks (spreads, straddles)
```

**API**:
```
GET /api/v1/options/SBIN-EQ
Response: [
  {
    "symbol": "SBIN23500CE",
    "strike": 500,
    "expiry": "2026-03-28",
    "type": "CE",
    "bid": 10.50,
    "ask": 10.75,
    "ltp": 10.60,
    "volume": 50000,
    "open_interest": 200000,
    "greeks": {"delta": 0.45, "gamma": 0.02, "theta": -0.003, "vega": 0.12}
  },
  ...
]

GET /api/v1/iv_metrics/SBIN-EQ
Response: {
  "iv_rank": 75,           # 75th percentile (high IV)
  "iv_percentile": 60,
  "current_iv": 25.5,
  "52w_high_iv": 40.0,
  "52w_low_iv": 15.0,
  "volatility_surface": {...}
}
```

---

## 10. System Reliability & Resilience

### Rate Limiting & Throttling (FIX-04: Broker + System Limits)

**Problem**: Broker APIs have strict rate limits (e.g., 100 req/min). Exceeding them causes rejections, throttling, or bans.

**Solution**: Three-tier rate limiting before any broker call.

**Architecture**:
```
User Request
    ↓
UserRateLimiter (per-user)
  └─ Check: "Has user exceeded 500 orders/day?"
  ├─ YES: Reject with 429
  └─ NO: Pass to next layer
    ↓
BrokerRateLimiter (per-broker)
  └─ Check: "Has broker exceeded rate limit?"
  ├─ YES: Queue request (exponential backoff)
  └─ NO: Pass to next layer
    ↓
SystemRateLimiter (global)
  └─ Check: "Is system overloaded?"
  ├─ YES: Queue or reject gracefully
  └─ NO: Submit to broker
    ↓
Broker API Call
```

**Configuration**:
```python
RATE_LIMITS = {
    "user": {
        "orders_per_day": 500,
        "orders_per_hour": 100,
        "api_calls_per_minute": 60
    },
    "broker": {
        "fyers": {
            "requests_per_minute": 100,
            "requests_per_second": 10,
            "max_concurrent": 5
        }
    },
    "system": {
        "max_queued_messages": 10000,
        "max_processing_latency_ms": 5000
    }
}
```

**Implementation**:
```python
class BrokerRateLimiter:
    """Token bucket algorithm for broker rate limits"""

    def __init__(self, broker_name: str, requests_per_minute: int):
        self.broker = broker_name
        self.tokens = requests_per_minute
        self.capacity = requests_per_minute
        self.refill_rate = requests_per_minute / 60  # tokens per second
        self.last_refill = time.time()

    async def check_or_wait(self):
        """Check if request allowed; wait if needed"""
        self.refill_tokens()
        if self.tokens >= 1:
            self.tokens -= 1
            return True

        # Rate limit exceeded: calculate wait time
        wait_time = (1 - self.tokens) / self.refill_rate
        await asyncio.sleep(wait_time)
        self.tokens -= 1
        return True

    def refill_tokens(self):
        """Refill tokens based on elapsed time"""
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now
```

**Error Handling**:
```python
# When rate limit exceeded
try:
    await rate_limiter.check_or_wait()
    response = await broker.place_order(order)
except RateLimitExceeded as e:
    # Emit alert event
    await publish_event("rate_limit.exceeded", {
        "broker": "fyers",
        "user_id": user_id,
        "timestamp": datetime.utcnow(),
        "retry_after_seconds": e.retry_after
    })
    # Return graceful error to user
    raise SmartTradeError("BROKER_RATE_LIMIT", "Broker busy, please try again in 60s")
```

**Monitoring**:
- Track broker rate limit status per user (WebSocket updates)
- Alert if consistently hitting limits (suggests need for request batching)
- Expose `/api/v1/rate_limit/status` endpoint showing current state

---

### Read Models (CQRS-Lite) (FIX-06: Query Optimization)

**Problem**: Every API request requires expensive calculations (P&L rollup, sector exposure, risk metrics).

**Solution**: Maintain pre-calculated "read models" updated asynchronously via event subscriptions.

**Design**:
```
Write Path (Execution Orchestrator):
  Order → DB update → Publish event
       │
       └─→ Risk calculated once

Read Path (API requests):
  GET /api/v1/positions → Fetch pre-calculated view
       └─ Fast (O(1) or O(log n))
```

**Read Models to Maintain**:

| Model | Source | Refresh Trigger | TTL |
|-------|--------|-----------------|-----|
| `position_view` | position table | position.changed event | Real-time |
| `portfolio_view` | Rollup of all positions | portfolio.updated event | Real-time |
| `pnl_view` | P&L calculations | position.changed event | Real-time |
| `risk_status_view` | Risk limits vs current | risk.check event | 1 second |
| `sector_exposure_view` | Grouped by sector | position.changed event | Real-time |

**Implementation**:
```python
class ReadModelUpdateService:
    """Subscribe to events and update read models"""

    async def on_position_changed(self, event: PositionChangedEvent):
        """Recalculate views when position changes"""
        # Update position_view
        await self.db.position_view.update(
            event.user_id,
            event.symbol,
            position_data
        )

        # Update portfolio_view (rollup)
        total_mtm = await self.calculate_total_mtm(event.user_id)
        await self.db.portfolio_view.update(
            event.user_id,
            {"mtm": total_mtm, "updated_at": datetime.utcnow()}
        )

        # Publish portfolio.updated event
        await publish_event("portfolio.updated", {
            "user_id": event.user_id,
            "mtm": total_mtm,
            "timestamp": datetime.utcnow()
        })

async def get_portfolio_fast(user_id: str):
    """Fast API endpoint: returns pre-calculated view"""
    # Previously: sum all positions + calculate P&L (expensive)
    # Now: fetch pre-calculated view (cheap)
    return await db.portfolio_view.find(user_id)
```

**Rules**:
- ✅ Read models are projections (always derived from write models)
- ✅ Write models (orders, positions) remain source of truth
- ✅ If read model corrupts, can be rebuilt from events (event sourcing)
- ✅ Read models updated asynchronously (eventual consistency is OK for UI)
- ❌ Never write to read models directly (only via event handlers)

**Consistency Guarantee**:
- Strong consistency: Write path (Orchestrator → DB → Event)
- Eventual consistency: Read models (updated within 100ms of event)

---

### Broker Resilience

**Patterns**:
```
Broker API Call
    ↓
Retry with Exponential Backoff
  ├─ Attempt 1: Immediate
  ├─ Attempt 2: 100ms
  ├─ Attempt 3: 200ms
  ├─ Attempt 4: 400ms
  ├─ Attempt 5: 800ms
  └─ Failure: Return error (max 5 retries)
    ↓
Circuit Breaker
  ├─ If 5 consecutive failures
  └─ Open circuit: Reject calls for 60s
    ↓
Fallback
  ├─ Use cached data
  ├─ Use mock broker
  └─ Queue for retry
```

### Signal Deduplication

**Problem**: Same signal fires multiple times in short period.

**Solution**:
```python
class SignalDeduplicator:
    """Prevent duplicate signal fires"""

    def __init__(self):
        self.seen_signals = {}  # key: "RSI_OVERSOLD_SBIN", value: timestamp

    async def should_emit(self, signal_key: str, min_interval: int = 60):
        """Emit signal only once per min_interval seconds"""
        now = time.time()
        last_seen = self.seen_signals.get(signal_key)

        if last_seen is None or (now - last_seen) > min_interval:
            self.seen_signals[signal_key] = now
            return True
        return False
```

### Strategy State Persistence

**Persistence** (Redis + PostgreSQL):
```
Strategy State
├─ Redis (fast, short-term)
│  ├─ Current signal state
│  ├─ Execution counters
│  └─ TTL: 24 hours
│
└─ PostgreSQL (durable, long-term)
   ├─ Full state snapshots
   ├─ History
   └─ Recovery on crash
```

### Scheduler (APScheduler)

**Purpose**: Time-based triggers (market open, close, rebalance).

**Examples**:
```python
# Daily reconciliation at market close
scheduler.add_job(
    reconcile_broker_positions,
    trigger="cron",
    hour=15, minute=30,  # 3:30 PM
    timezone="Asia/Kolkata"
)

# Weekly rebalancing
scheduler.add_job(
    rebalance_portfolio,
    trigger="cron",
    day_of_week="fri",
    hour=15, minute=0
)
```

### Conflict Resolution (Multi-Strategy)

**Problem**: Two strategies want conflicting orders (BUY vs SELL same symbol).

**Resolution Strategies**:
1. **Priority-based**: Strategy with higher priority wins
2. **First-come-first-served**: First strategy to fire gets execution
3. **Manual**: Alert user to choose
4. **Aggregate**: Combine orders (net position)

```python
async def resolve_conflict(self, orders: List[Order]) -> Order:
    """Resolve conflicting orders for same symbol"""
    # Filter: only same symbol, same user
    if len(orders) == 1:
        return orders[0]

    # Multi-strategy conflict
    if strategy_mode == "full_auto":
        # Priority-based: highest priority wins
        return max(orders, key=lambda o: o.strategy.priority)
    elif strategy_mode == "semi_auto":
        # Alert user
        await publish_event("order_conflict", {
            "orders": orders,
            "action_required": "choose_order"
        })
```

---

## 11. Service Boundaries (Clarity)

| Service | Owns | Does NOT Own | Publishes | Consumes |
|---------|------|--------------|-----------|----------|
| **Auth** | Users, JWT, RBAC | — | user.* events | — |
| **BAS** | Orders, Positions, Risk, PIE, WebSocket for account events | Market data ✅, MDS's market WebSocket, Audit logs ⚠️, Metadata ⚠️ | order.*, position.*, trade.* | market_data.quote, user requests |
| **MDS** | Instruments, Quotes, Calendar, WebSocket for market data | Orders, Positions, Execution, Account events ✅ | market_data.quote | broker.session |
| **Journal Service** (future) | Trade history, Audit logs ⚠️ | Execution, Position mgmt | journal.entry.created.v1 | trade.*.v1, action.executed.v1 |
| **Strategy** (future) | Strategies, Backtest | Execution, Positions | strategy.signal | market_data.quote, signal.* |
| **smarttrade-common** | Auth, DB, Events, Errors | Business logic | — | (used by all) |

**Principle**: Minimize cross-service coupling; use events for async communication.

**Architectural Debt Notes**:
- ⚠️ **BAS Metadata**: Currently maintains local instrument_cache.py (TTL-based). Should subscribe to MDS events or use MDS API only.
- ⚠️ **BAS Quote Cache**: Currently maintains local quote_store.py (in-memory). Should consume from market_data.quote events only.
- ⚠️ **BAS Audit Logs**: Currently stores action_log directly in BAS DB (sync writes). Phase 2-3: Switch to event model → Journal Service (action.executed.v1 → journal.entry.created.v1)

---

## 11.5 Architectural Cleanup Roadmap

### Phase 1 (Go-Live Cleanup) — 18 hours

**Goal**: Remove cache duplication, stabilize boundaries before production.

#### Task 1.1: Remove Quote Cache from BAS (4 hours)
```
REMOVE: quote_store.py
  - Delete in-memory quote cache
  - Remove quote update from WebSocket feed

CHANGE: Risk calculations & position tracking
  - Instead of reading from quote_store.get()
  - Subscribe to market_data.quote events from MDS
  - Store latest quote in memory during event processing
  - Fallback: API call to MDS if quote not in recent event stream
```

#### Task 1.2: Remove Instrument Cache Refresh Logic (6 hours)
```
REMOVE: instrument_cache.py refresh loop
  - Delete periodic refresh from MDS
  - Keep local cache as LRU (1-hour TTL)

CHANGE: Instrument lookups
  - On miss: Call MDS /api/v1/instruments/{symbol} (cached 1 hour locally)
  - Alternative (Phase 2): Subscribe to instrument.updated.v1 events

BENEFIT: No dual sources of truth; MDS is authoritative
```

#### Task 1.3: Start Publishing PIE Action Events (8 hours)
```
ADD: action.executed.v1 event publishing
  - When: Kill-switch triggers, auto-entry executes, rule fires
  - Payload: user_id, action_type, rule_id, position_symbol, effect (qty_change, pnl_impact)
  - Publishes to: EventBus (Kafka/Redis topic: action.executed.v1)

KEEP FOR NOW: action_log table in BAS (for UI queries)
  - Also store locally for 24-hour UI history
  - Phase 2-3: Migrate to read from Journal Service events

NOTE: Don't delete ActionLogService yet; just add event publishing alongside
```

---

### Phase 2-3 (Portfolio & Action Log Refactor) — 40+ hours

**Goal**: Separate portfolio engine, event-driven action logging, clear PIE boundaries.

#### Task 2.1: Extract Portfolio Engine (20 hours)
```
REFACTOR: portfolio_service.py from BAS
  - Current: Lives in broker_adapter_service/services/
  - Target: Separate component/package

NEW BOUNDARIES:
  - PositionEngine: Per-instrument tracking (FIFO, MTM, P&L/symbol)
  - PortfolioEngine: Aggregate metrics (total MTM, exposure, allocation %)
  - Interface: Portfolio consumes position events (position.changed.v1)
```

#### Task 2.2: Migrate Action Logging to Event-Driven (15 hours)
```
REFACTOR: action_log_service.py
  - Current: Sync writes to BAS DB
  - Target: Event publishing → Journal Service consumption

FLOW:
  PIE triggers action
    → publish action.executed.v1 event
    → Journal Service consumes
    → stores in journal_db
    → BAS queries Journal Service for audit history (or reads events)

BENEFIT: Audit logs are immutable, centralized, event-sourced
```

#### Task 2.3: Define PIE Boundaries Clearly (5+ hours)
```
CLARIFY: What PIE does vs. what Strategy does

PIE (Defensive Policy Engine):
  - Kill-switch: Close on loss threshold
  - Take-profit: Auto-exit at price
  - Position limits: Block entry if at max positions
  - Daily loss cap: Pause trading if breached

DEFER to Strategy Engine (Phase 4):
  - Entry signals (RSI, MACD, patterns)
  - Exit optimization
  - User-defined DSL
  - Backtesting

DOCUMENT: Clear separation; no overlap
```

---

### Phase 4 (Pre-Strategy Launch) — TBD

**Goal**: PIE and Strategy coexist without conflicts.

#### Task 4.1: Prepare PIE for Strategy Coexistence
```
EXTRACT: PIE from BAS into separate logical module/service
  - Option A: pie-service (separate deployment)
  - Option B: PIE as pure policy engine in smarttrade-common

ENSURE: No duplicate execution logic between PIE and Strategy
  - PIE applies after Strategy (safety layer)
  - Clear precedence: Risk rules > Strategy rules

NOTE: Detailed design depends on Strategy requirements
```

---

## 12. Deployment Model

### Local-First (Docker Compose)

```yaml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]

  redis:
    image: redis:7
    ports: ["6379:6379"]

  auth-service:
    build: ./authentication-service
    ports: ["8001:8001"]
    depends_on: [postgres, redis]

  market-data-service:
    build: ./market-data-service
    ports: ["8004:8004"]
    depends_on: [postgres, redis]

  broker-adapter-service:
    build: ./broker-adapter-service
    ports: ["8005:8005"]
    depends_on: [postgres, redis]

  frontend:
    build: ./smarttrade-frontend
    ports: ["5173:5173"]
    depends_on: [broker-adapter-service, market-data-service]
```

**Single-User Optimization**: Services share one PostgreSQL (dev mode) but use separate DBs (production).

### Future Scaling

**Multi-User**: Implement user isolation via schema/database segregation.

**Distributed Actor Model**: Replace asyncio with Pydantic's distributed actors or Ray.

**Multi-Broker**: Plugin architecture already supports (Fyers + Mock + future Zerodha, IBKR).

---

## 13. Data Flow Diagrams (Textual)

### Order Placement Flow

```
Client (Frontend)
  │ POST /api/v1/orders/place {symbol, qty, type}
  ↓
Route Handler
  │ 1. Validate schema (Pydantic)
  │ 2. Extract user_id from JWT
  │ 3. Get idempotency_key from header
  ↓
Execution Orchestrator (per-user queue)
  │ 1. Priority-queue message
  │ 2. Wait for turn (serialized)
  ↓
Risk Engine (First Layer)
  │ 1. Check daily loss limit
  │ 2. Check position limits
  │ 3. Check per-trade risk
  │ 4. REJECT if unsafe
  ↓ [if passes]
Idempotency Handler
  │ 1. Check: Has idempotency_key been used?
  │ 2. Return cached result if yes
  │ 3. Proceed if no
  ↓
Portfolio Engine (Second Layer)
  │ 1. Check portfolio constraints
  │ 2. Check sector exposure
  │ 3. REJECT if unsafe
  ↓ [if passes]
Broker Adapter
  │ 1. Call FyersAdapter.place_order()
  │ 2. Retry with exponential backoff (5 tries)
  │ 3. Handle timeout, network error
  ↓ [if success]
Order State Machine
  │ 1. Update order state: pending → accepted
  │ 2. Store order in DB
  │ 3. Store idempotency result
  ↓
Event Publishing
  │ 1. Publish order.placed event
  │ 2. Position engine subscribes
  │ 3. Risk monitor subscribes
  ↓
HTTP Response
  │ {"order_id": 123, "state": "accepted", "timestamp": "..."}
  ↓
Client (Frontend)
  └─ Display order in Orders panel
```

### Fill Processing Flow

```
Broker WebSocket Event: Fill received
  │ (priority=0, highest)
  ↓
Execution Orchestrator (high priority)
  │ 1. Queue fill message (priority 0)
  │ 2. Process before any other messages
  ↓
Position Engine
  │ 1. Update position: qty += filled_qty
  │ 2. Calculate new avg cost
  │ 3. Publish position.changed event
  ↓
Portfolio Engine
  │ 1. Update MTM
  │ 2. Update NAV
  │ 3. Check portfolio risk
  ↓
Risk Monitor
  │ 1. Check daily loss breach
  │ 2. Publish alert if breach
  ↓
Strategy Runtime
  │ 1. Notify on_fill() callback
  │ 2. Strategy may trigger next action
  ↓
Audit Logger
  │ 1. Log fill immutably
  ↓
EventBus
  │ 1. Publish order.filled.v1, trade.executed.v1, position.updated.v1
  ↓
BAS WebSocket Consumer
  │ 1. Read events from EventBus partition (user_id)
  │ 2. Assign sequence numbers
  │ 3. Queue for delivery
  ↓
BAS WebSocket (`ws://bas:8005/api/v1/ws`)
  │ 1. User isolation (JWT user_id claim)
  │ 2. Sequence-based ordering
  ↓
Frontend (Real-time update)
  └─ Display positions, P&L, orders (orders, trades, positions)
```

### Broker Sync Flow

```
Scheduled Job (daily, 3:30 PM):
  │ reconcile_broker_positions()
  ↓
Broker Adapter
  │ 1. Query FyersAdapter.get_all_positions()
  │ 2. Return: {symbol: qty, ...}
  ↓
Broker Sync Engine
  │ 1. Query DB: SmartTrade positions
  │ 2. Compare symbol-by-symbol
  ↓
Drift Detection
  │ For each symbol:
  │   1. Calculate variance = |broker_qty - st_qty| / st_qty
  │   2. Classify: LOW (< 0.1%), MEDIUM (0.1%-5%), HIGH (> 5%)
  ↓
Low Drift: Log only
  │
Medium Drift: Alert + Pause Auto-Entry
  │ 1. Publish broker_drift.medium event
  │ 2. Pause auto-entry for symbol
  │ 3. Allow user to review
  ↓
High Drift: Critical Alert
  │ 1. Publish broker_drift.high event
  │ 2. Pause ALL trading for symbol
  │ 3. Require manual reconciliation
  ↓
Ingest External Trade (if external buy detected)
  │ 1. Create Trade record
  │ 2. Update Position
  │ 3. Publish trade.ingested event
  ↓
Resolution
  │ 1. Position db updated
  │ 2. Strategy runtime notified
  │ 3. User alerted in UI
```

---

## 13.5 Audit API (FIX-09: Immutable Execution Audit Trail)

**Purpose**: Expose immutable audit logs via API for compliance, debugging, and audit trails.

**Design**: All order execution steps are logged immutably; API allows querying by order, trade, or time range.

**Audit Log Structure**:
```python
class AuditLogEntry(BaseModel):
    id: str                  # UUID
    user_id: str
    order_id: str            # Reference
    timestamp: datetime      # When event happened
    action: str              # "ORDER_PLACED", "ORDER_ACCEPTED", "FILL_RECEIVED", etc.
    details: dict            # Event-specific details

    # Immutable metadata
    created_at: datetime     # When logged (always now())
    source: str              # "broker", "user", "system"
    status_before: str       # Order state before action
    status_after: str        # Order state after action
```

**API Endpoints**:
```
GET /api/v1/audit/orders/{order_id}
Response:
[
  {
    "timestamp": "2026-03-28T10:30:00Z",
    "action": "ORDER_PLACED",
    "details": {"symbol": "SBIN-EQ", "side": "BUY", "quantity": 100},
    "status_before": "pending",
    "status_after": "pending"
  },
  {
    "timestamp": "2026-03-28T10:30:01Z",
    "action": "ORDER_ACCEPTED",
    "details": {"broker_order_id": "FY123456"},
    "status_before": "pending",
    "status_after": "accepted"
  },
  {
    "timestamp": "2026-03-28T10:30:45Z",
    "action": "FILL_RECEIVED",
    "details": {"fill_qty": 50, "fill_price": 625.50},
    "status_before": "accepted",
    "status_after": "partially_filled"
  }
]

GET /api/v1/audit/trades/{trade_id}
Response: Audit trail for a specific fill/trade

GET /api/v1/audit/range
Query: ?start_time=2026-03-28T10:00:00Z&end_time=2026-03-28T11:00:00Z&user_id=123
Response: All audit entries in time range

GET /api/v1/audit/user/{user_id}
Response: All audit entries for user (with pagination)
```

**Storage**:
- All audit logs stored in immutable append-only table
- Indexed by (order_id, timestamp) for fast queries
- Retention: 7 years (financial compliance)

**Guarantees**:
- ✅ Every execution step is logged
- ✅ Logs are immutable (no deletes, only appends)
- ✅ Logs include both system actions and broker responses
- ✅ Timestamped to microsecond precision (UTC)
- ✅ Can reconstruct full order lifecycle from audit trail

**Example: Reconstruct Order State from Audit Trail**:
```python
async def reconstruct_order_state(order_id: str, timestamp: datetime):
    """Reconstruct order state at specific point in time"""
    audit_entries = await db.audit_logs.find(
        order_id=order_id,
        timestamp__lte=timestamp,
        order_by="timestamp desc"
    )
    # Replay state transitions from logs
    current_state = "pending"
    for entry in reversed(audit_entries):
        current_state = entry.status_after
    return current_state
```

**Compliance Notes**:
- Audit logs satisfy regulatory requirements (SEBI, NSE)
- Can be exported to CSV/JSON for compliance reports
- Supports regulatory inquiries (e.g., "Show all trading on 2026-03-28")

---

## 14. Execution Roadmap (5 Phases)

### Phase 1: Foundation (Weeks 1-2, 7 hours)
**Goal**: Fix critical execution gaps.

**Tasks**:
- 1.1: Implement Order State Machine (3 hrs)
- 1.2: Integrate idempotency into endpoints (2 hrs)
- 1.3: Add order-specific error codes (2 hrs)

**Deliverable**: Formal order lifecycle, network retry safety.

---

### Phase 2: Execution Layer (Weeks 3-5, 40 hours)
**Goal**: Build centralized, reliable execution.

**Tasks**:
- 2.1: Design Execution Orchestrator (3 hrs)
- 2.2: Implement asyncio-based queue (20 hrs)
- 2.3: Refactor OrderHandler to use orchestrator (10 hrs)
- 2.4: Add execution metrics (7 hrs)

**Deliverable**: Centralized order queue, P50/P99 latency < 500ms.

**Stabilization** (Phase 2.5, 60 hrs):
- Load testing execution layer
- Validate idempotency
- Simulate broker failures
- Ensure no duplicate orders

---

### Phase 3: Broker Sync (Weeks 6-9, 90 hours)
**Goal**: Ensure position parity with broker.

**Tasks**:
- 3.1: Design Broker Sync Engine (5 hrs)
- 3.2: Implement external trade ingestion (30 hrs)
- 3.3: Implement drift detection (30 hrs)
- 3.4: Selective strategy pause (15 hrs)
- 3.5: Reconciliation strategy (10 hrs)

**Deliverable**: External trades ingested within 1 hour, drift alerts working.

---

### Phase 4: Strategy Engine (Weeks 10-14, 160 hours)
**Goal**: Enable user-defined algo trading.

**Tasks**:
- 4.1: Design Strategy Language (10 hrs)
- 4.2: Implement StrategyEngine (50 hrs)
- 4.3: Implement StrategyRuntimeEngine (40 hrs)
- 4.4: Build backtest engine (60 hrs)

**Deliverable**: Users can define if-then rules; backtests produce consistent results.

---

### Phase 5: Signal Engine (Weeks 15-18, 100 hours)
**Goal**: Cross-market signal generation.

**Tasks**:
- 5.1: Design Signal Framework (5 hrs)
- 5.2: Implement indicators + options signals (70 hrs)
- 5.3: Integrate signals into strategies (15 hrs)
- 5.4: Polish, documentation (10 hrs)

**Deliverable**: Users can subscribe to cross-market signals; strategies trigger on signals.

---

## 15. Critical Guardrails (DO NOT DO)

### ❌ Bypass the Execution Orchestrator
Direct broker calls from multiple places → race conditions, duplicate fills.
**Always**: Flow through ExecutionOrchestrator.

### ❌ Use Float for Money
`float` → precision loss → compliance risk.
**Always**: Use `Decimal` for monetary values.

### ❌ Skip User Isolation
Missing `user_id` filter → cross-user data leakage.
**Always**: Every query filters by authenticated user.

### ❌ Fire-and-Forget on Async Operations
Event publish without confirmation → lost events.
**Always**: Store idempotency keys in DB before attempting operations.

### ❌ Hardcode Broker-Specific Logic
Logic must be in `BrokerAdapterPlugin`, not routes.
**Always**: Use adapter pattern; support multiple brokers from day one.

### ❌ Block on I/O in Routes
Sync DB calls, blocking requests → timeouts.
**Always**: Use async everywhere; all routes are `async def`.

### ❌ Mix Business Logic in Routes
Validation, calculations must be in services.
**Always**: Routes → schema validation → service call → response.

### ❌ Skip Portfolio-Level Risk Controls
Strategy must respect portfolio constraints.
**Always**: Portfolio Engine runs AFTER Risk Engine; cannot bypass.

### ❌ Ignore Broker Sync
Position drift undetected → external trades invisible.
**Always**: Run daily reconciliation; detect & ingest external trades.

### ❌ Deploy Without Stabilization Phase
Untested execution layer → production failures.
**Always**: Complete Phase 2.5 (load testing, failure scenarios) before Phase 3.

---

## 16. Development Guidelines

### Service Implementation Pattern

```
Models → Schemas → Repository → Services → Routes

1. Models (SQLAlchemy ORM)
   └─ Database schema

2. Schemas (Pydantic)
   └─ Request/response I/O

3. Repository
   └─ Data access layer

4. Services
   └─ Business logic (this is where the logic lives)

5. Routes
   └─ HTTP marshalling (thin)
```

### Testing Strategy

```
Unit Tests: Pure business logic (OrderHandler, RiskEngine)
  └─ Location: tests/unit/
  └─ Target: 80%+ coverage

Integration Tests: Service + mocked repos
  └─ Location: tests/integration/
  └─ Scope: Order lifecycle, settlement flow

E2E Tests: 2+ services, real databases
  └─ Location: smarttrade-tests/
  └─ Scope: Full workflows (login → order → settlement)
```

### Git Workflow

```
1. Create branch: git checkout -b feature/1.1-order-state-machine
2. Implement task (Phase 1.1, 1.2, etc.)
3. Write tests (80%+ coverage)
4. Commit: "Implement order state machine (task 1.1)"
5. Create PR with reference to phase + task
6. Review + merge
7. Close task in claude.json
```

---

## Appendix: Glossary

| Term | Definition |
|------|-----------|
| **BAS** | Broker Adapter Service (core execution) |
| **MDS** | Market Data Service (real-time quotes, instrument resolution) |
| **PIE** | Position Intelligence Engine (auto-entry, kill-switch) |
| **Execution Orchestrator** | Centralized order queue (asyncio) |
| **Strategy Runtime** | Execution of user-defined strategies |
| **Signal Engine** | Pattern detection, indicator calculation |
| **Broker Sync** | Reconciliation, external trade detection |
| **Actor Model** | Per-user single-threaded queue (guarantees serialization) |
| **Idempotency** | Request deduplication (same result on retry) |
| **Drift** | Position mismatch between SmartTrade and broker |
| **MTM** | Mark-to-market (current position value) |
| **Exposure** | Risk measure (sector %, leverage, etc.) |
| **DSL** | Domain-specific language (for strategy definition) |

---

## Document Control

| Field | Value |
|-------|-------|
| **Title** | SmartTrade Architecture v3.4 — Production-Ready + Advisory Ecosystem |
| **Version** | 3.4 (upgraded 2026-03-28) |
| **Date** | 2026-03-28 |
| **Author** | Claude Code (AI) |
| **Status** | APPROVED FOR IMPLEMENTATION |
| **Upgrades in v3.4** | AI Orchestrator (advisory-only), Notification Service (async alerts), Journal Service (expanded), service boundary enforcement |
| **Upgrades in v3.3** | Event bus durability, contract registry, execution fault-tolerance, rate limiting, multi-level kill switches, read models, versioning, audit API |
| **Upgrades in v3.2** | Execution Orchestrator, idempotency, broker sync engine, portfolio separation |
| **Next Review** | After Phase 4 completion (Strategy Engine + AI advisory) |
| **Related Docs** | CLAUDE.md (master playbook), claude.json (execution plan) |

---

**END OF DOCUMENT**
