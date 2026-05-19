# SmartTrade Final Target Architecture v4.0
## Execution Plane Optimization & Service Boundary Finalization

**Status**: Current Implementation (aligned with actual codebase as of 2026-05-16)
**Date**: 2026-04-20 (v4.0 spec) · 2026-05-16 (aligned with current stateless implementation)
**Scope**: Microservices with stateless execution architecture
**Critical Constraint**: BAS is a stateless execution kernel; broker is the
single source of truth for orders/positions/trades; read-side state is
reconstructed downstream from BAS events. This document reflects the
current implementation after the stateless architecture refactor.

---

## 1. SERVICE RESPONSIBILITIES (Strict Definitions)

### Core Responsibility Definitions

#### **Broker Adapter Service (BAS)** — STATELESS EXECUTION KERNEL
**Responsibility**: Stateless order execution with broker as single source of truth. Fast, lightweight translation layer between SmartTrade order models and broker-native APIs.

**MUST OWN**:
- Order placement, cancellation, modification (stateless operations)
- Broker adapter plugins (Fyers, Mock/Paper)
- User broker account & session management
- Direct event publishing (fire-and-forget with outbox for critical events)
- Hybrid broker state synchronization (WebSocket streams + API polling)
- **Local replicated instrument master** (bootstrapped from MDS; zero runtime MDS calls during execution)

**MUST NOT DO**:
- ~~Persist order state~~ (broker is source of truth)
- ~~Persist position state~~ (broker is source of truth)
- ~~Persist trade state~~ (broker is source of truth)
- ~~Maintain local order state machine~~ (broker tracks order lifecycle)
- ~~Perform complex risk validation~~ (minimal validation only; comprehensive risk moved downstream)
- ~~Execute trading strategies~~ (Strategy Service responsibility)
- ~~Maintain quote cache~~ (consume from Redis Streams)
- ~~Resolve instrument metadata via synchronous MDS calls~~ (maintain local replicated copy via InstrumentSyncService)
- ~~WebSocket account events to frontend~~ (no WebSocket in BAS; broker communication only)
- Call downstream services synchronously (all async via event responses)

**Data Ownership** (Minimal - Stateless Design):
- Broker connection credentials (encrypted)
- Trading account metadata (account type, state, currency)
- Account ledger & balance (cached from broker)
- **NO** order/position/trade persistence (broker is source of truth)

**Stateless Architecture**:
```
Core Principles:
- Broker is single source of truth for all trading state
- No local order/position/trade persistence
- Fire-and-forget event publishing with outbox for critical events
- Hybrid state sync: WebSocket (real-time) + API polling (fallback)
- Minimal database schema reduces operational complexity
```

**Event Publishing**:
- Consolidated `order.updated` event with status field (PLACED, ACCEPTED, FILLED, REJECTED)
- Critical events use outbox pattern for transactional durability
- Non-critical events published immediately via EventBus
- Downstream services handle idempotency

**Data Replication Sources**:
- `instrument_snapshot`: Updated via InstrumentSyncService (snapshot bootstrap + 6h refresh from MDS)
- Market data: Consumed from `market.quote` Redis Streams (real-time quotes)
- Broker state: Hybrid sync via WebSocket streams + API polling

**Plane**: **EXECUTION** (synchronous, low latency, stateless)

---

#### **Market Data Service (MDS)** — DATA PROVIDER & AUTHORITATIVE SOURCE
**Responsibility**: Provide authoritative, real-time quotes and instrument metadata. Orchestrate replication to all services.

**MUST OWN**:
- Quote distribution (Fyers real-time + Paper Broker mock quotes)
- **Instrument master (authoritative source of truth)**
  - Ingestion from brokers (Fyers, Paper, etc.)
  - Normalization and validation (tick_size, lot_size, trading rules)
  - Publishing via REST API (`GET /api/v1/instruments`) for bootstrap
  - Publishing via events (`market.instrument`) for updates
- Trading calendar (market open/close, holidays)
- Quote publishing via Redis Streams (durable, ordered, idempotent)
- WebSocket market data feeds to frontend (real-time tickers)

**MUST NOT DO**:
- ~~Store trading data~~ (orders, trades, positions belong to BAS)
- ~~Execute orders~~ (BAS responsibility)
- Call other services for trading decisions
- Access BAS/PBS trading events (read-only RBAC violation)
- Depend on replicated copies staying in sync (accept eventual consistency)

**Data Ownership & Replication Model**:
- Quotes (latest snapshot + stream history) — streaming only, not replicated
- **Instruments (metadata, validation rules)** — authoritative source; intentionally replicated to all services
  - Each service (BAS, PBS, Strategy, etc.) maintains local replicated copy via InstrumentSyncService
  - Replication via snapshot bootstrap + periodic refresh (e.g., every 6 hours)
  - Services use local replica during execution; zero runtime calls back to MDS
- Trading calendar — replicated (low-frequency updates)

**Plane**: **ASYNC/DATA** (event-driven publish)

---

#### **Paper Broker Service (PBS)** — MOCK BROKER
**Responsibility**: Emulate broker behavior for paper/sandbox trading with deterministic execution using real market data.

**MUST OWN**:
- Order execution simulation (fill at market price, simulate latency)
- Position tracking in mock broker (per-order settlement)
- Account balance simulation (buying power, margin, dividend)
- Market data consumption (subscribes to `market.quote` for realistic pricing)
- Subscription control plane (publishes to `market.subscription.request`)
- Event publishing (via outbox for critical events)

**MUST NOT DO**:
- Publish trading events to BAS event streams (BAS publishes after PBS returns fill)
- Make trading decisions or strategy logic
- Access BAS domain events (read-only RBAC violation)

**Data Ownership**:
- Mock account state (balance, positions, margin)
- Mock order/trade/position state (paper broker is source of truth for paper accounts)

**Communication Model**:
- **Synchronous execution**: BAS calls PBS for paper account order execution (like external broker API)
- **Market data consumption**: Subscribes to `market.quote` for realistic pricing
- **Subscription control**: Publishes to `market.subscription.request` to manage market data subscriptions
- **Event publishing**: Uses outbox pattern for critical events (similar to BAS)

**Plane**: **EXECUTION** (synchronous, latency-sensitive, with market data consumption)

---

#### **Strategy Service** — DECISION ENGINE
**Responsibility**: Evaluate trading signals asynchronously and generate execution recommendations (advisory only).

**MUST OWN**:
- Signal evaluation (technical indicators, market conditions)
- Rule engine (if-then-else trading logic)
- Strategy state machine (active, paused, error)
- Decision event publishing (`strategy.decision`)

**MUST NOT DO**:
- Execute orders directly (BAS-only)
- Process every market tick synchronously (backpressure requirement)
- Make synchronous calls to BAS (event-based communication only)
- Determine risk (BAS execution-critical risk only)
- Block order execution (must work with cached, eventually-consistent data)

**Execution Authority Constraint**:
- **MUST NOT directly trigger order execution**
- **MUST NOT indirectly trigger order execution** (no calling BAS endpoints that execute)
- **Decisions are advisory only** — recommendations published as events
- **All execution originates from BAS-controlled flows ONLY**
- Rationale: Maintains single decision authority (BAS); prevents distributed execution logic

**Load Control Requirements**:
- MUST implement batching or sampling for high-frequency market data streams
- MUST have configurable throttling for signal evaluation frequency
- MUST implement backpressure handling to avoid consumer lag
- MUST NOT process every `market.quote` event if infrastructure cannot keep up

**Data Ownership**:
- Strategy configuration (rules, parameters)
- Strategy state (active, paused)
- Decision records (audit trail)

**Plane**: **ASYNC/DATA** (event-driven, non-blocking)

---

#### **Journal Service** — AUDIT & LEARNING
**Responsibility**: Record execution history and enable behavioral learning.

**MUST OWN**:
- Action audit trail (all PIE and Strategy decisions)
- Trade journal (complete trade history with context)
- Performance analytics (P&L, win rate, exit reasons)
- Behavioral learning data (future AI training)

**MUST NOT DO**:
- Make trading decisions
- Call BAS during execution
- Determine idempotency (BAS owns)

**Data Ownership**:
- Action logs (published by BAS/Strategy)
- Trade records
- Analytics

**Plane**: **ASYNC/DATA** (event-driven)

---

#### **Portfolio Service** — DERIVED READ MODEL
**Responsibility**: Build a derived read model from trading events; compute portfolio-level aggregations and analytics.

**MUST OWN**:
- Aggregated positions (cache from trade events)
- Greeks calculation (delta, gamma, vega, theta)
- Portfolio-level risk metrics (correlation, VaR, exposure)
- Portfolio P&L summaries

**MUST NOT DO**:
- Execute orders
- Make entry/exit decisions (Strategy Service)
- Be called synchronously during order execution
- Own or persist raw position data (broker owns positions)
- Influence risk validation in execution path (minimal validation only; comprehensive risk moved downstream)

**Data Ownership**:
- Aggregated positions cache (read-only, eventually consistent)
- Derived metrics (Greeks, VaR, etc.)

**Execution Guarantee**: BAS MUST NOT depend on Portfolio Service for execution decisions. Portfolio is eventual consistency only.

**Plane**: **ASYNC/DATA** (event-driven, non-blocking)

---

#### **Notification Service** — ALERT DELIVERY
**Responsibility**: Deliver real-time alerts to users across multiple channels.

**MUST OWN**:
- Alert publishing (WebSocket, email, SMS, push)
- User notification preferences
- Alert rate-limiting and deduplication

**MUST NOT DO**:
- Determine what to alert on (driven by events)
- Execute orders
- Store trading data

**Data Ownership**:
- Notification preferences
- Alert history (short-lived cache)

**Plane**: **ASYNC/DATA** (event-driven)

---

#### **AI Service** — ADVISORY ONLY
**Responsibility**: Provide LLM-powered insights and recommendations (non-trading).

**MUST OWN**:
- Market analysis (news, sentiment)
- Trading insights (suggestions, not decisions)
- Performance commentary

**MUST NOT DO**:
- Execute orders
- Call BAS (read-only to events)
- Make autonomous trading decisions

**Data Ownership**:
- Analysis results
- Insight cache

**Plane**: **ASYNC/DATA** (event-driven)

---

#### **User Settings Service** — CONFIGURATION
**Responsibility**: Store and retrieve user trading parameters and preferences.

**MUST OWN**:
- Trading preferences (leverage, risk appetite, notification settings)
- Strategy parameters
- API key management

**MUST NOT DO**:
- Execute orders
- Make trading decisions
- Access BAS trading data

**Data Ownership**:
- User settings
- Preferences

**Plane**: **ASYNC/DATA** (event-driven)

---

### Summary Table

| Service | Core Responsibility | Plane | Latency | Status | Documentation |
|---------|-------------------|-------|---------|---------|---------------|
| BAS | Order execution + state | EXECUTION | <100ms | ✅ Implemented | ✅ Complete |
| MDS | Quote/instrument data | ASYNC | ~100ms | ✅ Implemented | ✅ Complete |
| PBS | Mock broker | EXECUTION | <50ms | ✅ Implemented | ✅ Complete |
| Journal | Audit trail | ASYNC | N/A | ✅ Implemented | ✅ Complete |
| Portfolio | Risk aggregation | ASYNC | <500ms | ✅ Implemented | ✅ Complete |
| Notification | Alert delivery | ASYNC | N/A | ✅ Implemented | ✅ Complete |
| Authentication | JWT auth + RBAC | INFRA | N/A | ✅ Implemented | ✅ Complete |
| Strategy | Signal evaluation | ASYNC | N/A | 🔄 Mock Only | ⏸️ Planned |
| AI | Advisory insights | ASYNC | N/A | 🔄 Mock Only | ⏸️ Planned |
| User Settings | Configuration | ASYNC | N/A | ✅ Implemented* | ⏸️ Excluded |

*User Settings Service is implemented but excluded from detailed documentation in this update per user request.

**Documentation Coverage**: Core services (BAS, MDS, PBS, Journal, Portfolio, Notification, Authentication) have complete interface documentation including REST APIs, events, WebSocket protocols (where applicable), and data ownership.

---

## 2. EXECUTION PLANE vs ASYNC PLANE

### Execution Plane (Synchronous, Low Latency)

**Services**:
1. **BAS** (Stateless order execution, minimal validation, broker communication)
2. **PBS** (Mock broker execution with market data consumption)

**Characteristics**:
- Synchronous request-response
- <100ms latency requirement
- ZERO event queue delay
- In-memory caching for all external data (quotes, instruments, risk limits)
- Direct database access (no async event sourcing)
- **Network I/O ONLY to broker/PBS; NO other network calls**

**Communication Pattern** (Execution Path Only):
```
Frontend → BAS (place order) → PBS (execute) → BAS (return filled) → Frontend
  |           |________________|
  |___________________________|
         (all synchronous, <100ms, no additional network calls)
         
NOTE: Quote & instrument data must be pre-cached; NO synchronous MDS calls in execution path
```

**Why BAS is EXECUTION**:
- Order placement must be atomic and immediate
- Minimal validation must complete before broker contact
- Latency SLA: <100ms from request to broker call
- Cannot afford async event processing delays or network I/O (except broker call)
- Broker execution is synchronous; BAS must match that timing
- ZERO synchronous network dependencies except PBS/broker (no MDS REST calls in hot path)

**Why PBS is EXECUTION**:
- Executes mock fills synchronously (no delay acceptable)
- Called from BAS with tight latency SLA (<50ms)
- Stateless RPC interface (like external brokers)
- No event bus integration

---

### Async/Data Plane (Event-Driven, Flexible Latency)

**Services**:
0. **Market Data Service (MDS)** (Publish quote events via Redis Streams; not called synchronously by execution path)
1. **Portfolio Service** (Aggregate P&L, compute Greeks)
2. **Journal Service** (Record trades, behavioral learning)
3. **Notification Service** (Deliver alerts)

**Future/Planned Services**:
- **Strategy Service** (Evaluate signals, publish decisions) - Currently mock implementation
- **AI Service** (Advisory insights) - Currently mock implementation
- **User Settings Service** (User configuration) - Implemented but excluded from this update

**Characteristics**:
- Event-driven (Redis Streams pub/sub)
- Flexible latency (100ms to minutes acceptable)
- At-least-once delivery semantics
- Durable event sourcing (can replay from broker reconciliation)
- Asynchronous processing (background jobs acceptable)
- Database writes non-blocking to execution path

**Communication Pattern**:
```
BAS (publish order.updated) → Redis Streams
                                    ↓
         ┌──────────────────────────┼──────────────────────────┐
         ↓                           ↓                          ↓
    Portfolio Service       Notification Service       Journal Service
   (aggregate P&L)         (alert user)                (record trade)
```

**Why Services are ASYNC**:
- **MDS**: Publishes quote events asynchronously; BAS consumes from local cache (not from MDS API calls during execution)
- **Portfolio**: P&L aggregation non-critical to execution; can be 500ms+ late (derived read model only)
- **Journal**: Audit trail; asynchronous logging acceptable
- **Notification**: Alerts can be delayed 100-500ms
- **Strategy (Future)**: Decisions are recommendations, not execution; can be delayed
- **AI (Future)**: Insights are advisory; no latency requirement
- **User Settings**: Configuration changes can be cached 60s

---

### Data Flow: Execution + Async

```
┌─────────────────────────────────────────────────────────────┐
│ FRONTEND (React 18 + TypeScript)                             │
│ • Orders, Positions, Risk, Alerts                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼ (sync: place order)        ▼ (async: WebSocket subscribe)
   ┌─────────────────────────────────────────────┐
   │ EXECUTION PLANE (Synchronous)               │
   │                                             │
   │ ┌─────────────┐          ┌──────────────┐  │
   │ │ BAS         │◄────────►│ PBS / Broker │  │
   │ │ • Orders    │          │ • Execution  │  │
   │ │ • Risk      │          │ • Fill       │  │
   │ │ • Positions │          │              │  │
   │ │ • Idempotency           │              │  │
   │ └────────┬────┘          └──────────────┘  │
   │          │                                 │
   │          │ (synchronous lookups)           │
   │          │                                 │
   │          ▼                                 │
   │ ┌──────────────────────────────────────┐  │
   │ │ MDS (Quote Cache, Instrument Cache)  │  │
   │ └──────────────────────────────────────┘  │
   │                                             │
   └─────────────────────────────────────────────┘
                       │
        ┌──────────────▼──────────────────────┐
        │ EVENT BUS (Redis Streams)           │
        │ • order.* events                     │
        │ • trade.* events                     │
        │ • position.* events                  │
        │ • risk.* events                      │
        └──────────────┬───────────────────────┘
                       │
        ┌──────────────┼──────────────────────────────────────┐
        │              │                                       │
        ▼              ▼                                       ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ ASYNC/DATA PLANE (Event-Driven)                             │
   │                                                             │
   │ ┌─────────────┐  ┌───────────────┐  ┌─────────────────┐   │
   │ │ Strategy    │  │ Portfolio     │  │ Journal Service │   │
   │ │ Service     │  │ Service       │  │ • Audit trail   │   │
   │ │ • Signals   │  │ • Greeks      │  │ • Trade history │   │
   │ │ • Rules     │  │ • Aggregation │  │ • Analytics     │   │
   │ └─────────────┘  └───────────────┘  └─────────────────┘   │
   │                                                             │
   │ ┌─────────────────────┐          ┌──────────────────────┐  │
   │ │ Notification Svc    │          │ AI Service (advisory)│  │
   │ │ • Alerts            │          │ • Insights          │  │
   │ │ • Preferences       │          │ • Analysis          │  │
   │ └─────────────────────┘          └──────────────────────┘  │
   │                                                             │
   └─────────────────────────────────────────────────────────────┘
```

---

## 3. INTER-SERVICE COMMUNICATION RULES

### Rule 0: Global Execution Path (MINIMAL)
**DEFINITION**: Execution Path = Frontend HTTP → BAS → Broker/PBS (only)

**EXPLICIT CONSTRAINT**: 
- Any service beyond this path is a violation
- Frontend CANNOT call MDS, Strategy, Portfolio, or any async service directly for order execution
- BAS CANNOT call any async service synchronously during order processing
- Async services are consumers of events; never participants in execution path

**Latency Target**: <100ms total (Frontend → BAS → Broker → BAS → Frontend)

---

### Rule 1: Execution Path (BAS → Broker)
**ALLOWED**: Direct synchronous call to PBS/Broker  
**Reason**: Orders must execute atomically  
**Latency**: <50ms (BAS → PBS/Broker, including network)
**Pattern**: Request-response only, no event queue, no retries in hot path

---

### Rule 2: BAS Data Provisioning (Replicated Reference Data + Pre-Caching Model)
**ALLOWED**: Local in-memory caches populated by asynchronous background processes and replication services
- **Quotes**: In-memory cache (fed by `market.quote` stream consumer from MDS)
- **Instruments**: Local replicated copy via `InstrumentSyncService`
  - Snapshot bootstrap at startup from MDS (`GET /api/v1/instruments`)
  - Periodic refresh every 6 hours (scheduled job)
  - Stored in persistent InstrumentCache (survives restart)
  - Loaded into in-memory InstrumentRegistry for <1ms lookups
  - Optional event-driven updates via `market.instrument` (Phase 2)
- **Calendar**: In-memory cache (updated hourly via background job)

**NOT ALLOWED**: Synchronous REST calls to MDS during order execution  
**Reason**: Execution path must have ZERO runtime dependency on MDS network availability  
**Pattern**: 
1. Background replication jobs keep local caches fresh
2. Execution uses ONLY pre-cached data (in-memory lookups)
3. Startup bootstrap ensures cache is warm before first order
**Latency**: <1ms (in-memory lookup)

**Cache Failure Policy** (Deterministic Fallback):
- **If QuoteStore has no data for symbol**: Reject order with `quote_not_available` error
- **If InstrumentRegistry missing symbol**: Reject order with `instrument_not_found` error
- **If RiskLimits not cached**: Reject order with `risk_config_unavailable` error
- **NO blocking network calls as fallback**; failure is immediate and deterministic
- **Rationale**: Better to reject with known error than to incur latency variance from network calls

**Instrument Master as Replicated Reference Data**:
- MDS is authoritative source of truth (ingestion, normalization, validation rules)
- Each service maintains identical local replica (intentional duplication, not eventual consistency)
- Replicas are synchronized via InstrumentSyncService (smarttrade_common library)
- Services never call MDS for instrument resolution; lookup from local InstrumentRegistry only
- Staleness is detected and monitored; refresh can be triggered manually if needed

---

### Rule 3: BAS → Other Services (STRICTLY FORBIDDEN)
**NOT ALLOWED**: Synchronous calls to Strategy, Portfolio, Journal, Notification, AI, User Settings  
**Reason**: Execution path MUST be independent of downstream services  
**Why**: If Portfolio Service is slow, orders should still execute  

**Pattern**: All communication via events only
- BAS publishes `order.*.v1` events
- Async services subscribe and process asynchronously
- BAS does NOT wait for responses

---

### Rule 3: BAS MUST NOT Call MDS During Execution
**FORBIDDEN**: Any synchronous call from BAS to MDS during order execution  
**Forbidden calls**:
- `GET /api/v1/quotes/{symbol}` — use local QuoteStore from Redis Streams consumer
- `GET /api/v1/instruments/{symbol}` — use local InstrumentRegistry (replicated via InstrumentSyncService)
- `GET /api/v1/calendar` — use local cached trading calendar
- Any REST call to MDS service
- **Even with fallback**: Cache miss → reject order (do NOT call MDS as fallback)

**Reason**: Adds unpredictable latency (50-500ms); execution must be deterministic (<100ms)  
**Rationale**: All needed data must be pre-cached before order execution begins  
**Pattern**: 
1. Background replication jobs (InstrumentSyncService, MarketDataConsumer) keep caches fresh
2. Execution reads from local in-memory cache only
3. No network I/O in execution path except broker call
4. Cache miss → fast reject with clear error (not fallback network call)

---

### Rule 4: Strategy Service → MDS (Event-Only)
**ALLOWED**: Strategy subscribes to `market.quote` events; uses cached quote data  
**NOT ALLOWED**: Synchronous REST calls to MDS (would create execution latency coupling)
**Reason**: Strategy must not block order execution; must work with eventually-consistent cached data
**Pattern**: Event stream consumption with local caching

---

### Rule 5: Strategy Service → BAS (REST ONLY for Query)
**ALLOWED**: Read-only REST calls (GET portfolio, GET risk)
- `GET /api/v1/positions` (read portfolio)
- `GET /api/v1/risk/snapshot` (read current risk)

**NOT ALLOWED**: Posting orders or modifying state  
**Reason**: Strategy can query state but not execute; execution is BAS-only

---

### Rule 6: Async Services → Each Other (Event-Only)
**Pattern**: Redis Streams pub/sub only  
**Example**: 
- Notification Service subscribes to `order.*.v1`
- Portfolio Service subscribes to `position.updated`
- Journal Service subscribes to ALL events

**NOT ALLOWED**: REST calls between Strategy, Portfolio, Journal, Notification  
**Reason**: Decoupling + scalability

---

### Rule 7: Frontend → MDS WebSocket (Market Data Streaming)
**ALLOWED**: WebSocket connection for real-time market data
- Market data: quotes, option chains, greeks
- Broadcast to frontend (not persisted)

**NOT ALLOWED**: Frontend polling BAS repeatedly  
**Reason**: WebSocket is lower overhead

**Account Events Flow**: 
- BAS publishes events (order.*, trade.*, position.*, risk.*) to Event Bus
- Notification Service consumes events from Event Bus
- Notification Service delivers account events to Frontend via WebSocket
- Frontend does NOT connect to BAS for account events

---

### Rule 8: Frontend → MDS WebSocket (Streaming)
**ALLOWED**: Separate WebSocket for market data
- Real-time quotes for watched symbols
- Trading calendar updates

**NOT ALLOWED**: BAS forwarding market data to frontend  
**Reason**: Direct MDS feed is faster and cleaner

---

### Rule 9: ALL Inter-Service Calls Use BaseServiceClient
**Pattern**: All REST calls wrapped in BaseServiceClient
```python
# ✅ Correct
mds_client = BaseServiceClient(service_name="market_data_service")
instruments = await mds_client.request("GET", "/api/v1/instruments")

# ❌ Wrong
response = requests.get("http://mds:8004/api/v1/instruments")
```

**Reason**: Centralized retry logic, timeout, RBAC, logging

---

### Rule 10: ALL Events Use DomainEventPublisher
**Pattern**: All event publishing goes through DomainEventPublisher
```python
# ✅ Correct
await publisher.publish(
    domain_event=OrderFilledEvent(...),
    is_critical=True  # Use Outbox for critical events
)

# ❌ Wrong
redis_client.publish("order.filled", json.dumps(data))
```

**Reason**: Idempotency, Outbox pattern, retries

---

### Summary: Forbidden Call Patterns

| Pattern | Why Forbidden | Alternative |
|---------|---------------|-------------|
| BAS → Strategy (sync) | Execution depends on Strategy | BAS → Redis event → Strategy listens |
| BAS → Portfolio (sync) | Execution depends on Portfolio | BAS → Redis event → Portfolio listens |
| Strategy → BAS (POST/PUT) | Strategy cannot execute | Strategy → Redis event → BAS consumes |
| MDS → BAS (trading data access) | MDS not responsible for trading | BAS publishes order events; MDS is read-only |
| PBS → Event Bus | Mock broker is external-like | BAS translates PBS fills to events |
| Frontend → Async Services (direct) | No business logic in Frontend | Frontend → BAS → BAS relays via events |

---

## 3.5. MARKET DATA DISTRIBUTION MODEL (Hybrid: Streams + KV + Local Cache)

### Three-Layer Architecture

**Layer 1: Redis Streams (`market.quote`)**
- **Purpose**: Durable, ordered, replayable event log
- **Consumers**: All async services (Strategy, Portfolio, Notification, AI)
- **Characteristics**: Persistent, sequenced, replay-safe
- **Data**: `{ instrument_id, ltp, bid, ask, timestamp, sequence_number }`
- **Guarantee**: At-least-once delivery via consumer groups
- **Latency**: Not used by execution path (asynchronous)

**Layer 2: Redis KV (`ltp:{instrument_id}`)**
- **Purpose**: Fast shared snapshot for operational services (optional optimization)
- **Usage**: BAS background cache refresh, metrics aggregation
- **Characteristics**: Volatile, eventual consistency, high-speed lookup
- **TTL**: 60 seconds (older data discarded)
- **Data**: Latest quote snapshot (single value per instrument)
- **Latency**: <5ms lookup (Redis in-memory)
- **Guarantee**: Best-effort; can be slightly stale
- **Note**: Not read during execution path (fallback only)

**Layer 3: BAS Local Cache (`QuoteStore`)**
- **Purpose**: Primary source for order execution decisions
- **Update mechanism**: Asynchronous consumer of `market.quote` stream
- **Characteristics**: In-memory, process-local, pre-populated at startup
- **Data**: Latest quotes per instrument with sequence tracking
- **Latency**: <1ms lookup (in-process)
- **Guarantee**: Lossy-acceptable; missing quotes result in order rejection
- **Fallback**: No network calls; deterministic failure (reject order)

### Data Flow

```
MDS (Fyers/PBS) publishes quote
        ↓
1. Write Redis KV (ltp:{id}) immediately [async, non-blocking]
2. Publish Redis Stream (market.quote) [async, durable]
        ↓ ↓
        │ └→ Strategy/Portfolio/etc consume via stream [async]
        │
        └→ BAS stream consumer updates QuoteStore [async background job]
                ↓
        BAS execution uses local QuoteStore [synchronous, <1ms]
```

### Critical Rules

- **Rule**: `market.quote` is source of truth for sequencing and audit trail
- **Rule**: Redis KV is optimization layer; not a dependency for execution
- **Rule**: **BAS MUST read ONLY from local in-memory QuoteStore during execution**
- **Rule**: **BAS MUST NOT read Redis Streams or KV during order execution**
- **Rule**: Each quote update includes `sequence_number` for idempotency
- **Rule**: BAS ignores out-of-order or stale updates (sequence check)
- **Rule**: If QuoteStore is empty or quote not found → **reject order (no blocking calls)**

### Sequence Handling

```
BAS local tracking: seen_sequence[instrument_id] = 42

New quote arrives: sequence=43
  ✅ Accept (sequence > last_seen)
  
New quote arrives: sequence=42
  ❌ Skip (sequence <= last_seen; duplicate)
  
New quote arrives: sequence=41
  ❌ Skip (sequence < last_seen; out of order)
```

---

## 4. EVENT FLOW ARCHITECTURE

### Core Event Types (Domain Events)

#### **Order Domain** (Public Events Only)
```
order.updated
├─ Triggered by: Order status changes (PLACED, ACCEPTED, FILLED, REJECTED, CANCELLED)
├─ Producer: BAS
├─ Consumers: Journal, Notification, Strategy, Frontend
├─ Critical: YES (Outbox) for FILLED status; NO for others
└─ Data: { account_id, order_id, symbol, quantity, price, type, status, broker_order_id, timestamp, [fill details if status=FILLED] }
```

**RATIONALE**: Consolidated event with status field reduces event volume and simplifies consumer logic. Status field indicates order lifecycle stage (PLACED → ACCEPTED → FILLED/REJECTED/CANCELLED). Critical events (FILLED) use outbox pattern for durability.

#### **Trade Domain** (Executed Trades)
```
trade.executed
├─ Triggered by: order.updated with status FILLED processed into position
├─ Producer: BAS
├─ Consumers: Portfolio, Journal, Notification
├─ Critical: YES (Outbox)
└─ Data: { trade_id, order_id, symbol, quantity, price, fee, timestamp }
```

#### **Position Domain** (Aggregated Holdings)
```
position.updated
├─ Triggered by: trade.executed
├─ Producer: BAS
├─ Consumers: Portfolio, Notification
├─ Critical: NO (always computable from trades)
└─ Data: { account_id, symbol, quantity, avg_price, current_price, p_l }

broker.position.snapshot
├─ Triggered by: Bootstrap / reconciliation from BAS
├─ Producer: BAS
├─ Consumers: Portfolio (initial position load)
├─ Critical: NO (bootstrap event)
└─ Data: { account_id, symbol, quantity, avg_price }

broker.holding.snapshot.v1
├─ Triggered by: Bootstrap / reconciliation from BAS
├─ Producer: BAS
├─ Consumers: Portfolio (holdings snapshot)
├─ Critical: NO (bootstrap event)
└─ Data: { account_id, symbol, quantity, settlement_details }
```

#### **Risk Domain** (Inline Risk + Async Alerts)
```
risk.limit_breach.v1
├─ Triggered by: Minimal inline risk validation detects threshold breach
├─ Producer: BAS (or downstream risk service)
├─ Consumers: Notification
├─ Critical: NO (notification only; execution already rejected)
└─ Data: { limit_type, current_value, threshold, account_id }

risk.metrics.updated.v1
├─ Triggered by: Portfolio Service computes Greeks/VaR
├─ Producer: Portfolio Service (derived from trades)
├─ Consumers: Frontend (for display)
├─ Critical: NO (advisory metrics only)
└─ Data: { portfolio_delta, portfolio_gamma, var_95, correlation_risk }
```

**IMPORTANT DISTINCTION**:
- **Inline Risk** (BAS execution-critical): margin, position limits, daily loss (all pre-cached or computed instantly)
- **Derived Risk** (async, eventual consistency): Greeks, VaR, correlation (computed by Portfolio Service via event processing)

#### **Action Domain** (PIE Actions)
```
action.executed
├─ Triggered by: Auto-entry, kill-switch, or user-initiated rule
├─ Producer: BAS (PublishedByActionOrchestrator)
├─ Consumers: Journal (audit trail)
├─ Critical: NO (advisory/audit)
└─ Data: { action_id, action_type, rule_id, result, timestamp }

action.status_changed.v1
├─ Triggered by: Action state change (pending → executed, failed)
├─ Producer: BAS
├─ Consumers: Journal
├─ Critical: NO
└─ Data: { action_id, old_status, new_status, reason }
```

#### **Market Data Domain** (Quote Distribution)
```
market.quote
├─ Triggered by: Fyers/Paper quote received
├─ Producer: MDS
├─ Consumers: BAS, PBS, Strategy (all consume via Redis Streams)
├─ Critical: NO (durable but lossy acceptable)
├─ Pattern: Redis Streams + KV snapshot
└─ Data: { instrument_id, ltp, bid, ask, timestamp, sequence_number }

market.instrument
├─ Triggered by: Instrument metadata change
├─ Producer: MDS
├─ Consumers: BAS, Strategy (cached)
├─ Critical: NO
└─ Data: { instrument_id, exchange, lot_size, tick_size }
```

#### **Strategy Domain** (Signal Evaluation)
```
strategy.decision
├─ Triggered by: Signal evaluation complete
├─ Producer: Strategy Service
├─ Consumers: Journal, Notification
├─ Critical: NO (advisory)
├─ Data: { strategy_id, decision_type (BUY/SELL), confidence, symbol, quantity }

strategy.paused.v1
├─ Triggered by: Risk breach or user pause
├─ Producer: Strategy Service
├─ Consumers: Notification, Journal
├─ Critical: NO
└─ Data: { strategy_id, reason, pause_timestamp }
```

#### **Notification Domain** (Alerts)
```
notification.sent
├─ Triggered by: Alert delivery
├─ Producer: Notification Service
├─ Consumers: Journal
├─ Critical: NO
└─ Data: { notification_id, event_type, user_id, channel, timestamp }
```

### Event Publishing Rules

**Critical Events** (use Outbox Pattern — MUST NOT be Lost):
- `order.updated` with status=FILLED (execution record; basis for P&L, positions)
- `trade.executed` (execution completion record)

**Optional-Critical Events** (use Outbox if available, but not mandatory):
- `position.updated` (derived from trade; helps Portfolio consistency)
- `market.quote` (Redis Streams with consumer group tracking)

**Non-Critical Events** (standard publish, lossy acceptable):
- `order.updated` with status=PLACED/ACCEPTED/REJECTED/CANCELLED (informational)
- `risk.limit_breach.v1` (alert only; risk already enforced by execution rejection)
- `action.executed` (audit trail; not critical to execution)
- `strategy.decision` (advisory; not critical)
- `notification.sent` (delivery tracking)

**RATIONALE**: Minimize Outbox overhead. Only events critical to execution authority (orders, trades, positions) use durable Outbox.

### Consumer Group Strategy

| Service | Consumer Group | Events | Behavior |
|---------|---|---|---|
| BAS | bas-consumer | market.quote | Idempotent; skip if already processed |
| PBS | pbs-consumer | market.quote | Idempotent; update quote cache |
| Portfolio | portfolio-consumer | trade.*.v1, position.*.v1, broker.*.snapshot.v1 | Aggregate positions, compute Greeks |
| Journal | journal-consumer | ALL events | Audit trail; never drop |
| Notification | notification-consumer | ALL events (@subscribe('*')) | Alert user on key events |

**Future/Planned Consumers**:
- Strategy (strategy-consumer): order.*.v1, trade.*.v1 - Evaluate signals on order fill
- AI (ai-consumer): trade.*.v1 - Behavioral learning

---

### WebSocket Protocol Summary

| Service | Endpoint | Purpose | Authentication |
|---------|-----------|---------|----------------|
| **MDS** | `WS /api/v1/ws?token=<jwt>` | Market data streaming (quotes, option chains) | JWT in query string |
| **Notification** | `WS /api/v1/ws/notifications?token=<jwt>&last_seq=<int>` | Account events (orders, trades, positions, risk) + alerts | JWT in query string |
| **PBS** | `WS /internal/api/v1/execution-updates` | Internal execution updates to BAS | Internal service-to-service |

**WebSocket Characteristics**:
- **MDS**: Real-time market data streaming (quotes, option chains, greeks)
- **MDS**: Subscribe/unsubscribe actions for symbols and option chains
- **Notification**: Unified event consumption, replay support via last_seq parameter
- **PBS**: Internal-only, outbound to BAS, no public access

**Message Format Consistency**:
- All WebSocket messages use JSON format
- Include message type field for routing
- Support heartbeat/ping for connection health
- Error handling with explicit error message types

#### **REST API Verification Status**
All documented REST API paths have been cross-referenced with actual service implementations:

✅ **BAS**: Paths verified against broker-adapter-service/README.md
✅ **MDS**: Paths verified against market-data-service/README.md  
✅ **Authentication**: Paths verified against authentication-service/README.md
✅ **Portfolio**: Paths verified against portfolio-service/README.md
✅ **Journal**: Paths verified against journal-service/README.md
✅ **Notification**: Paths verified against notification-service/README.md
✅ **PBS**: Paths verified against paper-broker-service/README.md

**Note**: Some BAS endpoints are marked as deprecated with migration paths to Journal and Portfolio services.

---

## 5. SERVICE CONTRACTS (High-Level)

### BAS Contracts

#### **REST APIs** (Public)
```
# Order Management
POST   /api/v1/orders/{broker_id}/{account_id}                    # Place order
PUT    /api/v1/orders/{broker_id}/{account_id}/{broker_order_id}  # Modify order
DELETE /api/v1/orders/{broker_id}/{account_id}/{broker_order_id}  # Cancel order
GET    /api/v1/orders/{broker_id}/{account_id}                    # List orders (deprecated → Journal)
GET    /api/v1/orders/{broker_id}/{account_id}/{broker_order_id}  # Get order (deprecated → Journal)

# Portfolio & Funds
GET    /api/v1/portfolio/{broker_id}/{account_id}/funds             # Funds / margins
DELETE /api/v1/portfolio/{broker_id}/{account_id}/positions        # Square-off positions
GET    /api/v1/portfolio/{broker_id}/{account_id}/positions        # Positions (deprecated → Portfolio)
GET    /api/v1/portfolio/{broker_id}/{account_id}/holdings         # Holdings (deprecated → Portfolio)

# Session Management
POST   /api/v1/session/{broker_id}                                   # Open broker session
POST   /api/v1/session/{broker_id}/{account_id}                      # Open per-account session

# Trading Account Management
/trading_account[/{broker_id}[/{account_id}]]         # Trading account CRUD (GET/POST/DELETE)

# Broker Connection Management
/broker_connection[/{broker_id}]                     # Broker connection CRUD (GET/PUT/DELETE)
POST   /api/v1/broker_connection/refresh/{broker_id}                 # Force token refresh

# OAuth (Broker Credential Management)
GET    /api/v1/oauth/{broker_id}/authorize                          # OAuth bootstrap
GET    /api/v1/oauth/{broker_id}/callback                            # OAuth callback
```

**Note**: Read endpoints for orders, positions, and holdings are deprecated and marked for migration to Journal Service and Portfolio Service. They include `Deprecation: true` header and `Link: rel="successor-version"` pointing to successor services.

#### **Stateless Architecture Notes**
```
BAS does NOT persist order/position/trade state (broker is source of truth):
  - Order state queried from broker API on demand
  - Position state reconciled via broker WebSocket + API polling
  - Trade state obtained from broker execution history
  - No local state machine or persistence
  
Event Publishing:
  - Consolidated order.updated event with status field
  - Critical events use outbox pattern for transactional durability
  - Non-critical events published immediately via EventBus
  - Downstream services handle idempotency
```

#### **Events Published** (Public Only)
```
order.updated         # CRITICAL (Outbox) with status field (PLACED, ACCEPTED, FILLED, REJECTED, CANCELLED)

trade.executed        # CRITICAL (Outbox)

position.updated      # Optional-critical

risk.limit_breach.v1

action.executed
```

**Note**: Internal lifecycle events (validated, broker_submitted, placement_initiated) are BAS-internal state transitions; not published.

#### **Events Consumed**
```
market.quote (via Redis Streams consumer group, populates QuoteStore cache)
```

#### **Data Provisioning** (Pre-Cached, Not Synchronous)
```
QuoteStore:
  - Populated by market.quote event stream consumer
  - In-memory cache (<5ms lookup)
  - Lossy acceptable (missed quotes don't block execution)
  - Fallback: reject orders if cache empty
  
InstrumentCache:
  - Preloaded at startup from MDS
  - Refreshed every 6h via background job
  - No synchronous REST calls during execution
  - Fallback: reject orders if instrument not cached
```

#### **WebSocket Streams**
```
WS /api/v1/ws?token=<jwt>
  - JWT authentication via query parameter
  - Account events filtered by user's JWT claims
  - Real-time order, trade, position, and risk updates
  
Message Types:
  - order.* (filtered to user's accounts)
  - trade.* (filtered to user's accounts) 
  - position.* (filtered to user's accounts)
  - risk.* (filtered to user's accounts)
  
Note: BAS does not provide WebSocket endpoints. Account events (orders, trades, positions, risk) are delivered to Frontend via Notification Service WebSocket. Market data streaming is handled by MDS WebSocket.
```

---

### MDS Contracts

#### **REST APIs** (Public)
```
# Instrument Master
/instruments/...                                         # Instrument lookup, broker symbol mapping

# Quote Data
/data/quote                                             # Latest quote
/data/ohlc                                              # Historical OHLC data

# Options Data  
/options/chain                                          # Option chain
/options/greeks                                          # Greeks data

# Margin Data
/margin/...                                             # Margin lookup

# Backtest Data
/backtest/...                                            # Backtest data feed

# Internal Broker Instrument APIs
/broker_instrument/...                                  # Internal broker-symbol APIs
```

**Instrument Master Endpoint Behavior**:
- `GET /api/v1/instruments` returns complete list with all fields (symbol, tick_size, lot_size, status, etc.)
- Response includes version and checksum for staleness detection
- Intended for batch replication (InstrumentSyncService bootstrap/refresh); not called per-order

#### **Events Published**
```
market.quote (via Redis Streams)
market.instrument
```

#### **WebSocket Streams**
```
WS /api/v1/ws?token=<jwt>
  - JWT authentication via query parameter
  - Market data streaming (quotes, option chains)
  
WebSocket Actions:
  - subscribe.market(symbols: [NSE:INFY, BSE:INFY])    # Subscribe to symbols
  - unsubscribe.market(symbols: [NSE:INFY])            # Unsubscribe from symbols
  - subscribe.option_chain(symbol: NSE:NIFTY50-INDEX)  # Subscribe to option chain
  - unsubscribe.option_chain(symbol: NSE:NIFTY50-INDEX) # Unsubscribe from option chain

Message Format:
  {
    "type": "quote" | "option_chain" | "greeks",
    "symbol": "NSE:INFY",
    "data": { ... }
  }
```

#### **Data Ownership & Replication Model**
```
Quotes (latest snapshot + stream history):
  - Streaming only, not replicated to other services
  - Published via Redis Streams (market.quote)
  - Redis KV snapshot for execution-path freshness checks
  - MDS is authoritative source

Instruments (metadata, validation rules):
  - Authoritative source; intentionally replicated to all services
  - Each service maintains local replica via InstrumentSyncService
  - Replication via snapshot bootstrap + periodic refresh (6h)
  - Services use local replica during execution; zero runtime MDS calls
  - Event-driven updates via market.instrument (Phase 2)

Trading Calendar:
  - Replicated (low-frequency updates)
  - Cached in-memory (updated hourly via background job)
```

---

### Strategy Service Contracts

#### **REST APIs** (Public)
```
GET    /api/v1/strategies                   # List strategies
GET    /api/v1/strategies/{id}              # Get strategy details
POST   /api/v1/strategies/{id}/pause        # Pause strategy
POST   /api/v1/strategies/{id}/resume       # Resume strategy
```

#### **Events Published**
```
strategy.decision
strategy.paused.v1
strategy.resumed.v1
```

#### **Events Consumed**
```
order.updated (filter by status=FILLED)
trade.executed
market.quote (if needed)
```

---

### Portfolio Service Contracts

#### **REST APIs** (Public)
```
GET    /api/v1/positions/{broker_id}/{account_id}              # List positions (paginated)
GET    /api/v1/positions/{broker_id}/{account_id}/{position_id}  # Single position details
GET    /api/v1/holdings/{broker_id}/{account_id}               # List holdings
GET    /api/v1/holdings/{broker_id}/{account_id}/{holding_id}   # Single holding details
GET    /api/v1/portfolio/{broker_id}/{account_id}              # Portfolio summary
```

**Query Parameters**:
- `limit`: Pagination limit (default 50, max 1000)
- `offset`: Pagination offset (default 0)
- `instrument_id`: Filter by instrument (optional)
- `status`: Filter by position status ("OPEN" or "CLOSED", optional)

**Response Example (Portfolio Summary)**:
```json
{
  "broker_id": "fyers",
  "account_id": "ACC123",
  "user_id": "user-uuid",
  "total_value": "1000000.00",
  "unrealized_pnl": "50000.00",
  "realized_pnl": "25000.00",
  "total_positions": 10,
  "net_exposure": "750000.00",
  "margin_used": "200000.00",
  "buying_power": "800000.00",
  "updated_at": "2025-03-28T12:00:00Z"
}
```

#### **Events Consumed**
```
position.updated (from BAS)
  - Upsert PositionSnapshot
  - Trigger portfolio summary recalculation

broker.position.snapshot (from BAS)
  - Bootstrap / reconciliation snapshot
  - Initial position load

broker.holding.snapshot.v1 (from BAS)
  - Holdings snapshot
  - Long-only, settled positions

market.quote (from MDS Redis Streams)
  - Consumer group: portfolio-quote-consumer
  - Update in-memory quote cache
  - Drives live valuation in portfolio summaries
```

#### **Data Processing Pattern**
```
In-Memory Quote Cache:
  - Consumed from market.quote Redis Stream
  - <5ms lookup for position valuation
  - Lazy persistence to database
  - Lossy acceptable (quotes are transient)

Snapshot Upserts:
  - Positions keyed by (user_id, broker_id, account_id, instrument_id)
  - Full replacement on each event (no delta math)
  - Idempotency via EventLog (event_id guard)

Background Scheduler:
  - Periodic portfolio summary recalculation (~30s)
  - Handles edge cases (quote ticks without position events)
  - Recomputes totals, P&L, Greeks
```

#### **Data Ownership**
```
PositionSnapshot:
  - Latest position per (user, broker, account, instrument)
  - Net quantity, average price, buy/sell breakdowns
  - Realized P&L, status (OPEN/CLOSED)

HoldingSnapshot:
  - Latest holding (long-only, settled positions)
  - Per-instrument aggregation

PortfolioSummary:
  - Per-(user, broker, account) summary
  - Totals, P&L, exposures, margin usage
  - Periodically recomputed (not event-driven)

EventLog:
  - Idempotency guard (event_id tracking)
  - Prevents duplicate event processing
```

#### **Idempotency Pattern**
```
Event Processing:
1. Check EventLog.exists(event_id)
2. If already processed → skip (exactly-once guarantee)
3. If new → process, persist, log event_id
4. Commit transaction

This handles network retries and duplicate event delivery safely.
```

---

### Journal Service Contracts

#### **REST APIs** (Public)
```
GET    /api/v1/trades                        # Trade history
GET    /api/v1/trades/{trade_id}            # Single trade details
GET    /api/v1/positions                     # Position history
GET    /api/v1/positions/{position_id}      # Single position details
```

**Query Parameters**:
- `instrument_id`: Filter by instrument (optional)
- `from_date`: Filter by execution date ≥ (optional)
- `to_date`: Filter by execution date ≤ (optional)
- `status`: Filter by position status ("OPEN" or "CLOSED", optional)
- `limit`: Pagination limit (default 50, max 1000)
- `offset`: Pagination offset (default 0)

**Response Example (Trade List)**:
```json
{
  "total": 150,
  "items": [
    {
      "id": "uuid",
      "event_id": "event-uuid",
      "user_id": "user-uuid",
      "broker_id": "fyers",
      "account_id": "ACC123",
      "instrument_id": "NIFTY50-INDEX",
      "side": "BUY",
      "quantity": 10,
      "price": "21000.50",
      "executed_at": "2025-03-28T12:00:00Z",
      "created_at": "2025-03-28T12:00:01Z",
      "updated_at": "2025-03-28T12:00:01Z"
    }
  ],
  "limit": 50,
  "offset": 0
}
```

**Response Example (Position Details)**:
```json
{
  "id": "uuid",
  "user_id": "user-uuid",
  "broker_id": "fyers",
  "account_id": "ACC123",
  "instrument_id": "NIFTY50-INDEX",
  "net_quantity": 10,
  "average_price": "21000.50",
  "buy_quantity": 10,
  "buy_average": "21000.50",
  "sell_quantity": 0,
  "sell_average": "0",
  "realized_pnl": "5000.00",
  "status": "OPEN",
  "snapshot_at": "2025-03-28T12:00:00Z",
  "created_at": "2025-03-28T12:00:00Z",
  "updated_at": "2025-03-28T12:05:00Z"
}
```

#### **Events Consumed**
```
trade.executed (from BAS)
  - Append-only Trade record creation
  - Complete execution details with event_id

position.updated (from BAS)
  - PositionSnapshot upsert
  - Current holdings per instrument

ALL events (complete audit trail)
  - EventLog tracks all processed events
  - Enables replay and behavioral analysis
```

#### **Idempotency Pattern**
```
Event Processing:
1. Check EventLog.exists(event_id)
2. If already processed → skip (exactly-once guarantee)
3. If new → process, persist, log event_id
4. Commit transaction

EventLog Schema:
  - event_id (unique, indexed)
  - topic (event type)
  - payload (event data)
  - processed_at (timestamp)

This handles network retries and duplicate event delivery safely.
```

#### **Data Ownership**
```
Trade (Append-Only, Immutable):
  - event_id (unique) — idempotency key
  - user_id, broker_id, account_id, instrument_id — routing keys
  - side (BUY|SELL), quantity, price (as string for precision)
  - order_id, executed_at — references and timestamp
  - Never updated, only appended

PositionSnapshot (Upserted):
  - Keyed by (user_id, broker_id, account_id, instrument_id)
  - net_quantity, average_price
  - buy_quantity, buy_average, sell_quantity, sell_average — FIFO details
  - realized_pnl, status (OPEN|CLOSED)
  - Fully replaced on each event (no delta math)

EventLog (Append-Only, Idempotency Guard):
  - event_id (unique, indexed)
  - topic, payload, processed_at
  - Prevents duplicate event processing
```

#### **Price Precision**
```
All prices stored as strings (e.g., "21000.123456789") not floats
- Preserves decimal precision across JSON serialization
- Calculations use Decimal(price_string)
- Database stores as TEXT/VARCHAR for precision
```

#### **Access Control**
```
RBAC Policy:
  - trades.read.self: true (users read own trades)
  - positions.read.self: true (users read own positions)
  - Service impersonation: BAS, MDS can read any user's data

Journal is read-only from all perspectives:
  - No create/update/delete permissions
  - Pure event consumer and query service
```

---

### Notification Service Contracts

#### **REST APIs** (Public)
```
# Subscription Management
POST   /api/v1/subscriptions                    # Create subscription
GET    /api/v1/subscriptions                    # List user subscriptions
GET    /api/v1/subscriptions/{id}               # Get subscription details
PUT    /api/v1/subscriptions/{id}               # Update subscription
DELETE /api/v1/subscriptions/{id}               # Delete subscription
GET    /api/v1/subscriptions/streams            # Per-stream toggle state
PUT    /api/v1/subscriptions/streams            # Bulk stream-toggle update
POST   /api/v1/subscriptions/streams/{stream}/enable   # Enable a stream
POST   /api/v1/subscriptions/streams/{stream}/disable  # Disable a stream

# User Preferences
GET    /api/v1/user/preferences/notification     # Get notification preferences
PUT    /api/v1/user/preferences/notification     # Update notification preferences

# Catalog
GET    /api/v1/catalog/events                   # Catalog of known events
GET    /api/v1/catalog/events/{event_name}      # Event details
GET    /api/v1/catalog/categories               # Notification categories
GET    /api/v1/catalog/severities               # Notification severities

# Metrics
GET    /metrics                                  # Prometheus metrics (public endpoint)
```

**Subscription Creation Example**:
```json
POST /api/v1/subscriptions
{
  "event_pattern": "order.*",
  "enabled": true,
  "channels": [
    {"channel": "ui", "config": {}}
  ]
}
```

**Response Example**:
```json
{
  "id": "uuid",
  "user_id": "user-uuid",
  "event_pattern": "order.*",
  "enabled": true,
  "channels": [
    {
      "channel": "ui",
      "config": {}
    }
  ],
  "created_at": "2025-03-28T12:00:00Z",
  "updated_at": "2025-03-28T12:00:00Z"
}
```

#### **WebSocket API**
```
Endpoint: WS /api/v1/ws/notifications?token=<jwt>&last_seq=<int>

Authentication:
  - JWT token via query parameter
  - RBAC enforced via token validation

Features:
  - Real-time notification streaming
  - Message replay on reconnection (via last_seq parameter)
  - 5-second heartbeat for connection health
  - Automatic connection management

Message Types:
  - notification: Real-time notification
  - replay: Replayed historical message
  - replay_complete: Replay completion marker
  - heartbeat: Keep-alive ping
  - error: Error message

WebSocket Message Example:
{
  "type": "notification",
  "sequence": 12345,
  "notification": {
    "id": "uuid",
    "user_id": "user-uuid",
    "event_type": "order.updated",
    "severity": "INFO",
    "category": "TRADING",
    "title": "Order Filled",
    "message": "Your order for 10 NIFTY50-INDEX has been filled",
    "created_at": "2025-03-28T12:00:00Z"
  }
}
```

#### **Events Consumed**
```
Unified Event Consumption (@subscribe('*')):
  - ALL domain events (order.*, risk.*, system.*, broker.*, etc.)
  - Event matching via wildcard patterns (e.g., "order.*" matches "order.placed")
  - Idempotency via event_id tracking

Key Event Patterns:
  - order.updated (filter by status=FILLED/CANCELLED/REJECTED)
  - risk.limit_breach.v1
  - position.updated
  - trade.executed
  - system.* (for operational alerts)
  - strategy.paused.v1 (future - when Strategy Service is implemented)
  - ai.* (future - when AI Service is implemented)
```

#### **Event Processing Pattern**
```
Unified Consumer:
  - Single @subscribe('*') consumer receives all events
  - Subscription matching engine (wildcard patterns supported)
  - Template-based message generation (Jinja2)
  - Multi-channel delivery (WebSocket, future email/SMS/push)

Rate Limiting:
  - 100 notifications per minute per user/event_type
  - 1000 notifications per hour per user/event_type
  - Configurable per event type
  - Prevents notification spam

Message Generation:
  - Jinja2 templates per event type
  - Template registry auto-loads from templates/ directory
  - Supports title and message customization
  - Event payload available as template context

Delivery Logic:
  - UI Channel: WebSocket push (real-time)
  - Email Channel: Placeholder for future implementation
  - SMS Channel: Placeholder for future implementation
  - Push Channel: Placeholder for future implementation
```

#### **Data Ownership**
```
notification_subscriptions:
  - User event subscriptions with wildcard pattern matching
  - Indexes: user_id, event_pattern
  - Unique constraint: (user_id, event_pattern)

notification_subscription_channels:
  - Channel configuration per subscription
  - Supports UI, email, SMS, push channels
  - Config stored as JSONB
  - Cascade delete on subscription deletion

notification_messages:
  - Generated notification messages
  - Links to event via event_id
  - Severity, category, title, message fields
  - Sequence number for WebSocket replay
  - Indexes: user_id, event_id, notification_id, sequence_number

notification_delivery_log:
  - Delivery status per channel (excluding UI per NS-007)
  - Status: pending, delivered, failed, retrying
  - Retry count and failure reason tracking
  - Indexes: notification_id, channel, user_id, status

user_notification_preferences:
  - User-level notification settings
  - Per-category enable/disable
  - Channel preferences
  - Rate limiting overrides
```

#### **Data Retention Lifecycle**
```
Retention Policy (NS-011):
  - Hot storage: 30 days (frequent access)
  - Warm storage: 90 days (archived, slower access)
  - Deletion: 180 days (automatic cleanup)

Automated Lifecycle Management:
  - Background job moves hot → warm
  - Background job deletes after 180 days
  - Configurable retention periods
  - Compliance-friendly data lifecycle
```

#### **Key Design Decisions**
```
NS-001: Unified event consumption with single @subscribe('*') consumer
NS-002: Severity ownership moved to EventEnvelope (publisher-owned)
NS-003: Normalized subscription-channel schema
NS-004: Wildcard event subscriptions supported
NS-005: Removed notification_sequence table, using BIGSERIAL
NS-006: Automatic WebSocket replay via ?last_seq=12345
NS-007: Skip UI channel delivery logging (only log external channels)
NS-008: Rate limiting (100/min, 1000/hour per user/event_type)
NS-009: Jinja2 template registry for message generation
NS-010: Notification category enum (TRADING, RISK, SYSTEM, BROKER, AI)
NS-011: Retention lifecycle (30d hot, 90d warm, 180d delete)
NS-012: SmartTrade event publishing standards enforced
```

---

### Authentication Service Contracts

#### **REST APIs** (Public)
```
POST   /auth/register                       # Create user account
POST   /auth/login                          # Issue access + refresh tokens
POST   /auth/refresh                        # Exchange refresh → new access token
POST   /auth/logout                         # Invalidate refresh token
GET    /auth/me                             # Return current user context
POST   /auth/change-password                # Change user password
```

**Authentication**: All protected endpoints require HTTP Bearer authentication:
```
Authorization: Bearer <access_token>
```

**Token Format**:
- Access Token: JWT (HS256), 30-minute expiration
- Refresh Token: UUID stored in database, device/session bound
- Token Refresh: POST /auth/refresh with refresh_token in request body

**Request Examples**:

Register:
```json
POST /auth/register
{
  "username": "amit",
  "email": "amit@example.com", 
  "password": "StrongP@ssw0rd!",
  "accept_tos": true,
  "full_name": "Amit Agrawal"
}
```

Login Response:
```json
{
  "access_token": "<jwt>",
  "token_type": "bearer",
  "expires_in": 1800,
  "refresh_token": "<jwt-refresh>"
}
```

Refresh:
```json
POST /auth/refresh
{
  "refresh_token": "<refresh>"
}
```

#### **Events Published**
```
None (authentication state changes are not published to event bus)
```

#### **Events Consumed**
```
None (authentication service does not consume trading events)
```

#### **Data Ownership**
```
Users:
  - id: UUID (primary key)
  - username: str (unique)
  - email: Optional[str]
  - hashed_password: str (argon2id/bcrypt)
  - is_active: bool
  - roles: List[str] (RBAC)
  - created_at, updated_at: datetime

RefreshTokens:
  - id: UUID (primary key)
  - user_id: UUID (FK → Users)
  - token: str (hashed/encrypted, never plain)
  - device_info: Optional[str]
  - expires_at: datetime
  - revoked_at: Optional[datetime]
```

#### **Security Features**
- Strong password hashing (Argon2id preferred, bcrypt fallback)
- Rate limiting on login endpoint
- Structured audit logs for authentication events
- RBAC integration via JWT claims
- Global logout support (revoke all refresh tokens)

---

### Paper Broker Service (PBS) Contracts

#### **REST APIs** (Internal - BAS Only)
```
POST   /api/v1/order/{broker_id}/{account_id}           # Place paper order
PUT    /api/v1/order/{broker_id}/{account_id}/{order_id}    # Modify paper order
DELETE /api/v1/order/{broker_id}/{account_id}/{order_id}    # Cancel paper order
GET    /api/v1/order/{broker_id}/{account_id}/{order_id}    # Get paper order details
GET    /api/v1/order/{broker_id}                         # List paper orders

GET    /api/v1/position/{broker_id}                      # List paper positions
POST   /api/v1/position/{broker_id}/{account_id}/exit    # Square-off paper positions

GET    /api/v1/trade/{broker_id}                         # List paper trades
GET    /api/v1/account/{broker_id}/{account_id}/balance  # Get paper account balance
```

**Access Control**: Internal service called by BAS only. No public access. BAS authenticates via service-to-service communication.

#### **Internal WebSocket Protocol** (PBS → BAS)
```
Endpoint: /internal/api/v1/execution-updates
Direction: PBS → BAS (outbound only)
Purpose: Real-time execution updates (order fills, position changes)

Message Format:
{
  "event_type": "order.filled" | "position.changed" | "trade.executed",
  "broker_id": "paper",
  "account_id": "ACC123",
  "order_id": "uuid",
  "data": { ... }
}
```

#### **Market Data Consumption**
```
Consumes: market.quote (from MDS Redis Streams)
Consumer Group: pbs-consumer
Purpose: Realistic pricing for paper trading execution

Subscription Model:
- PBS subscribes to instruments needed for open paper orders
- Subscription control plane: publishes to market.subscription.request
- Reconnect support: PBS-owned registry for replay on reconnection
```

#### **Execution Engine Characteristics**
```
Deterministic Execution:
- Fills at market price from MDS quotes
- Simulates broker latency (configurable)
- Per-order settlement logic
- No random/slippage (deterministic for testing)

Account State:
- Balance simulation (buying power, margin)
- Position tracking (real-time)
- Trade history (complete audit trail)
- Dividend simulation (optional)
```

#### **Events Published**
```
None (PBS communicates with BAS via internal WebSocket only)
BAS publishes trading events on behalf of PBS after receiving execution updates
```

#### **Events Consumed**
```
market.quote (for realistic pricing)
market.subscription.request (for subscription management)
```

#### **Data Ownership**
```
Paper Account State:
- Mock account balance and margin
- Mock order/trade/position state (PBS is source of truth for paper accounts)
- Execution state for deterministic fills
- Subscription registry for instruments

Database: PostgreSQL (paper_broker_service database)
```

#### **Development/Testing Endpoints**
```
POST   /api/v1/price/{broker_id}              # (test) Push test price
DELETE /api/v1/cleanup/{broker_id}            # Dev/test cleanup helpers
DELETE /api/v1/cleanup/{broker_id}/{account_id}  # Per-account cleanup
```

---

## 6. EXTRACTION READINESS FROM BAS

### Components Currently in BAS

| Component | Current Location | Target Service | Readiness | Effort | Timeline |
|-----------|---|---|---|---|---|
| **Quote Cache** | `quote_store.py` | MDS (consume via Redis Streams) | Safe NOW | 6h | Phase 1 (Weeks 1-2) |
| **Instrument Cache** | `instrument_cache.py` | smarttrade_common (replicated reference data) | Safe NOW | 8h | Phase 1 (Weeks 1-2) |
| **Instrument Master Replication** | *(to be added)* | smarttrade_common.instrument_master | Design Complete | 12-16h | Phase 1 (Weeks 1-2) |
| **Action Logging** | `action_log_service.py` | Journal (event publishing) | Phased | 6h | Phase 4 (Weeks 7-8) |
| **AutoEntry Logic** | `entry_service.py` | Strategy Service | Phased | 20h | Phase 2 (Weeks 3-4) |
| **KillSwitch Logic** | `kill_switch_service.py` | Strategy Service | Phased | 18h | Phase 3 (Weeks 5-6) |
| **Portfolio Engine** | `portfolio_service.py` | Portfolio Service | Future | 40h | Q3 (after Phase 5) |
| **PIE (full)** | `pie/` | Strategy Service | Future | 60h | Q4 (after Phase 5) |

### Extraction Phases (5-Phase Strangler Pattern)

#### **Phase 1: Market Data Caches & Instrument Master Replication** (Weeks 1-2, 20-28h)
**Goal**: Implement replicated instrument master in smarttrade_common; migrate BAS and PBS to use it

**Changes**:
1. **Implement smarttrade_common.instrument_master package** (12-16h)
   - Canonical Instrument model
   - InstrumentRegistry (in-memory fast lookup)
   - InstrumentCache (persistent storage with versioning)
   - InstrumentSyncService (bootstrap + periodic refresh)
   - Shared validators (InstrumentValidator)
   - Documentation + tests

2. **Update BAS to use replicated instrument master** (4-6h)
   - Replace BAS InstrumentCache with InstrumentSyncService
   - Bootstrap at startup
   - Scheduled refresh (every 6 hours)
   - Remove synchronous MDS calls in OrderHandler
   - Add health check endpoint

3. **Update PBS to use replicated instrument master** (2-4h)
   - Use same InstrumentSyncService from smarttrade_common
   - Validate fills respect tick_size, lot_size

4. **Quote handling** (2-4h)
   - QuoteStore: Receive from `market.quote` stream (not local updates)
   - Stop MarketDataConsumer from writing to local caches

**Data Replication Model**:
- Instruments: Replicated reference data (snapshot bootstrap + 6h refresh)
- Quotes: Streamed (real-time, lossy acceptable)

**Risk**: LOW (replicated reference data is a mature pattern; instruments change infrequently)
**Validation**: 
- Instrument lookup <1ms
- No MDS calls during order execution
- Bootstrap succeeds even if MDS unavailable (use previous cache)

**Rollback**: Feature flag `BAS_INSTRUMENT_SOURCE=old_embedded_cache`

---

#### **Phase 2: AutoEntry Decision/Execution Split** (Weeks 3-4, 16-20h)
**Goal**: Separate AutoEntry decision logic from execution

**Changes**:
1. Create `AutoEntryDecisionService` (pure logic, no execution)
2. Publish `action.executed` event BEFORE ordering
3. OrderHandler reads decision from event, executes order
4. Prepare extraction to Strategy Service (future Phase 6)

**Risk**: MEDIUM (decision logic is complex; requires regression tests)
**Validation**: Rule evaluation unchanged; order execution unchanged

**Rollback**: Feature flag `BAS_AUTO_ENTRY_FLOW=legacy`

---

#### **Phase 3: KillSwitch Decision/Execution Split** (Weeks 5-6, 14-18h)
**Goal**: Separate KillSwitch decision logic from execution

**Changes**:
1. Create `KillSwitchDecisionService` (pure logic)
2. Publish `action.executed` event BEFORE position exit
3. PositionHandler reads decision, executes exit
4. Prepare extraction to Strategy Service (future Phase 6)

**Risk**: MEDIUM (similar to Phase 2)
**Validation**: Cooldown logic unchanged; position exit unchanged

**Rollback**: Feature flag `BAS_KILL_SWITCH_FLOW=legacy`

---

#### **Phase 4: Event-Driven Logging** (Weeks 7-8, 12-16h)
**Goal**: Convert action logging from sync DB writes to event publishing

**Changes**:
1. ActionLogService publishes `action.executed` → Journal Service
2. Keep local DB logging for idempotency (non-critical)
3. Journal Service becomes source of truth for audit trail
4. Remove or minimize BAS action_logs table

**Risk**: LOW (logging is advisory)
**Validation**: Action availability in audit trail; no blocking in execution

**Rollback**: Feature flag `BAS_ACTION_AUDIT=db_only`

---

#### **Phase 5: Code Organization** (Weeks 9-10, 8-12h)
**Goal**: Reorganize code for future service extraction

**Changes**:
1. Move PIE logic to `pie/` subpackage with clear boundaries
2. Move Journal/audit code to `journal/` subpackage
3. Document service extraction checklist
4. No logic changes; pure code movement

**Risk**: LOW (structural only)
**Validation**: All tests pass; imports work correctly

**Rollback**: Git revert

---

### Post-Phase 5: Future Service Extraction (Phase 6+)

| Service | Extraction Method | Timeline |
|---------|---|---|
| **Strategy Service v1.0** | Extract PIE (AutoEntry, KillSwitch) + rule engine | Phase 6 (Q3) |
| **Journal Service v1.0** | Consume all events; standalone audit service | Phase 5-6 |
| **Portfolio Service v1.0** | Consume position events; standalone aggregation | Phase 6-7 (Q3) |

---

## 7. ANTI-PATTERNS & HARD CONSTRAINTS

### Forbidden Patterns (Must Prevent)

#### **Pattern 0: Any Service Call in Execution Path (Including MDS)**
❌ **FORBIDDEN**: Any service call beyond Frontend → BAS → Broker in execution path  
**Includes**: MDS, Strategy, Portfolio, Journal, Notification, any other service

Examples:
```python
# ❌ WRONG: Execution depends on Portfolio Service
async def place_order(order):
    validate_risk()
    await portfolio_service.check_correlation_risk(order)  # BLOCKS EXECUTION
    await broker.execute(order)

# ❌ WRONG: Execution depends on MDS
async def place_order(order):
    validate_risk()
    quote = await mds_client.get_quote(order.symbol)  # BLOCKS, UNPREDICTABLE LATENCY
    await broker.execute(order)

# ❌ WRONG: Even caching with fallback is wrong
async def place_order(order):
    quote = cache.get(order.symbol) or await mds_client.get_quote(order.symbol)  # FALLBACK NETWORK CALL
```

✅ **CORRECT**: Only BAS internal logic; all data pre-cached
```python
# ✅ RIGHT: Execution uses only pre-loaded data
async def place_order(order):
    # All data is already in memory
    quote = self.quote_store[order.symbol]  # <1ms, in-memory lookup
    validate_risk(quote)  # Inline risk only
    await broker.execute(order)
    return response  # <100ms total
    
    # Portfolio/Strategy listen via event (async)
    await publisher.publish(OrderFilledEvent(order))
```

**Enforcement**: 
- Code review: Inspect all service calls in execution path; reject any service I/O
- Static analysis: Grep for HTTP clients, Redis calls (except local cache), RPC calls
- Integration test: Verify order execution succeeds with all other services down or artificially delayed

---

#### **Pattern 0.5: MDS Fallback During Execution**
❌ **FORBIDDEN**: Using MDS as fallback when cache miss occurs  
Example:
```python
# ❌ WRONG: If cache miss, fetch from MDS
async def place_order(order):
    quote = self.quote_store.get(order.symbol)
    if quote is None:
        # This is a network call in execution path!
        quote = await mds_client.get_quote(order.symbol)
    await broker.execute(order)
```

✅ **CORRECT**: Cache miss = reject order
```python
# ✅ RIGHT: If cache miss, fail fast
async def place_order(order):
    quote = self.quote_store.get(order.symbol)
    if quote is None:
        raise OrderRejected("quote_not_available")
    await broker.execute(order)
```

**Rationale**: 
- Fallback creates non-deterministic latency (sometimes <100ms, sometimes >500ms)
- Hides cache staleness problems from monitoring
- Encourages lazy initialization (dangerous in production)

**Enforcement**: 
- Code review: Reject any MDS calls in OrderHandler (even with try/catch)
- Tests: Inject cache miss; verify fast rejection (not fallback)

---

#### **Pattern 1: Splitting Execution Across Services**
❌ **FORBIDDEN**: Order execution distributed to multiple services  
Example:
```python
# ❌ WRONG: Execution scattered
class OrderHandler:
    async def execute(self, order):
        validate_risk()           # BAS
        evaluate_strategy()       # Strategy Service
        check_broker_sync()       # Portfolio Service
        place_order_at_broker()   # BAS
```

✅ **CORRECT**: Single service owns execution
```python
# ✅ RIGHT: Execution atomic in BAS
class OrderHandler:
    async def execute(self, order):
        validate_risk()           # BAS only
        place_order_at_broker()   # BAS only
        publish(order.updated with status FILLED)     # BAS emits event
        # Async services subscribe and react
```

**Enforcement**: Code review; no sync calls to Strategy/Portfolio from OrderHandler

---

#### **Pattern 2: Synchronous Calls in Execution Path**
❌ **FORBIDDEN**: BAS → downstream service during order execution  
Example:
```python
# ❌ WRONG: Blocks order on Portfolio Service
async def place_order(order):
    validate_risk()
    
    # If Portfolio is slow, order is blocked!
    portfolio = await portfolio_service.update_on_fill(order)
    
    await broker.execute(order)
```

✅ **CORRECT**: Fire-and-forget events
```python
# ✅ RIGHT: Order executes; services react asynchronously
async def place_order(order):
    validate_risk()
    await broker.execute(order)

    # Portfolio listens to order.updated event independently
    await publisher.publish(OrderUpdatedV1(order, status="FILLED"))
```

**Enforcement**: Integration tests measure latency; reject PRs >100ms for order path

---

#### **Pattern 3: Duplicate Data Ownership**
❌ **FORBIDDEN**: Two services owning same data
Example:
```python
# ❌ WRONG: Both BAS and Portfolio own positions
# BAS position tracking (order fills)
class PositionEngine:
    positions: Dict[symbol, Position]

# Portfolio also maintains positions
class PortfolioService:
    positions: Dict[symbol, Position]  # DUPLICATE!
```

✅ **CORRECT**: Single source of truth (Broker)
```python
# ✅ RIGHT: Broker owns positions; BAS and Portfolio read from broker/events
# BAS: Stateless, queries broker for current state
# Portfolio: Reconstructs from events, consumes market data for valuation
    
class PortfolioService:
    # Maintains aggregated cache only; refreshed from events
    cache_positions = {}  # Read-only; derived from BAS via events
```

**Enforcement**: Clear data ownership in architecture; RBAC enforces read/write boundaries

---

#### **Pattern 4: MDS Handling Execution Events**
❌ **FORBIDDEN**: MDS accessing or storing trading events (orders, trades, positions)  
Example:
```python
# ❌ WRONG: MDS consuming order.updated
class MDSEventConsumer:
    async def on_order_updated(self, event):
        # MDS has NO BUSINESS with trading events!
        self.update_execution_cache(event)
```

**Enforcement**: RBAC denies MDS access to:
- `order.*` events
- `trade.*` events
- `position.*` events
- BAS/PBS trading data

**Current Status**: Gap 2.1, 2.2 in PBS RBAC (MDS read access)  
**Fix**: Remove MDS from RBAC for PBS positions/trades

---

#### **Pattern 5: BAS Runtime Dependency on Downstream Services**
❌ **FORBIDDEN**: Order execution blocked by downstream service availability  
Example:
```python
# ❌ WRONG: Order blocked if Strategy Service is down
async def place_order(order):
    # If this times out, order placement fails
    strategy_decision = await strategy_service.evaluate(order)
    
    if not strategy_decision.approved:
        raise OrderRejected()
    
    await broker.execute(order)
```

✅ **CORRECT**: BAS independent; Strategy is advisory
```python
# ✅ RIGHT: BAS executes regardless; Strategy listens asynchronously
async def place_order(order):
    validate_minimal_risk()  # BAS minimal validation only
    await broker.execute(order)

    # Strategy listens to order.updated event; might publish alert
    await publisher.publish(OrderUpdatedV1(order, status="FILLED"))
```

**Enforcement**: 
- Integration test: Order execution succeeds even if Strategy Service is down
- Latency test: Order path <100ms even if downstream is slow

---

#### **Pattern 6: Event Processing in Execution Path**
❌ **FORBIDDEN**: Waiting for async events during order execution  
Example:
```python
# ❌ WRONG: Blocks on journal write
async def place_order(order):
    await broker.execute(order)
    
    # If Journal is slow, order response is delayed
    await journal_service.log_trade(order)
```

✅ **CORRECT**: Fire-and-forget
```python
# ✅ RIGHT: Order completes; Journal listens independently
async def place_order(order):
    await broker.execute(order)
    return OrderResponse(order)  # Return immediately
    
    # Journal Service subscribes to trade.executed independently
    # (awaited asynchronously in background)
```

**Enforcement**: Response time measured before event publishing; PRs must show SLA met

---

#### **Pattern 7: Cyclic Dependencies**
❌ **FORBIDDEN**: Service A calls Service B; Service B calls Service A  
Example:
```python
# ❌ WRONG: Circular dependency
# BAS.py
async def order_filled():
    await portfolio_service.update()  # BAS → Portfolio

# portfolio_service.py
async def update():
    await bas_service.get_risk()      # Portfolio → BAS (cycle!)
```

✅ **CORRECT**: Acyclic directed graph
```
Frontend → BAS
            → PBS (broker)
            → MDS (quotes)
            → publishes events
                → Portfolio
                → Strategy
                → Journal
                → Notification
```

**Enforcement**: Dependency graph audit; reject PRs with cycles

---

#### **Pattern 8: Blocking Event Consumers**
❌ **FORBIDDEN**: Event consumer blocks on heavy processing  
Example:
```python
# ❌ WRONG: Slow processing blocks consumer group
class JournalEventConsumer:
    async def on_trade_executed(self, event):
        # If this takes 5 seconds, consumer lag increases
        self.compute_analytics(event)    # Heavy computation
        self.save_to_warehouse(event)    # I/O bound
```

✅ **CORRECT**: Async offload
```python
# ✅ RIGHT: Ack immediately; process async
class JournalEventConsumer:
    async def on_trade_executed(self, event):
        # Ack to Redis Streams immediately
        await self.save_to_journal_db(event)  # Fast
        
        # Offload analytics to background task
        asyncio.create_task(self.compute_analytics_async(event))
```

**Enforcement**: Consumer lag monitoring; alert if >30s lag

---

### Hard Constraints (Non-Negotiable)

| Constraint | Why | Enforcement |
|-----------|-----|-------------|
| **Execution Path = Frontend → BAS → Broker ONLY** | Minimal latency; no async dependencies | Architecture review; no exceptions |
| **BAS order latency <100ms** | Broker execution is synchronous | Latency benchmark in CI; fail if >100ms |
| **BAS zero runtime deps (except broker)** | Execution must not be blocked | Integration test: place order with all services down |
| **No synchronous MDS calls in execution path** | Cache-based pre-loading only | Code review; grep for MDS REST in OrderHandler |
| **Quote cache lookup <5ms** | In-memory, pre-cached data | Benchmark test; reject if >5ms |
| **Event publishing <10ms overhead** | Async impact on execution response | Benchmark test (order response before publish) |
| **No sync calls to async services** | Low latency requirement | Code review; static linting |
| **All inter-service calls via BaseServiceClient** | RBAC, retry logic, monitoring | Import check; reject raw HTTP calls |
| **All events via DomainEventPublisher** | Idempotency, Outbox pattern | Import check; reject direct Redis pub/sub |
| **Consumer group lag <30s** | Eventual consistency of async services | Monitoring alert; dashboard visible |
| **Strategy cannot execute orders** | Single decision authority (BAS) | Code review; no execute_order method |
| **Portfolio is read-only derived model** | No write access to execution state | RBAC + code review |
| **MDS publishes; never calls trading APIs** | Unidirectional data flow | RBAC audit + code review |
| **PBS has no event bus access** | External broker model | Code review; no Redis Streams access |

---

## 8. FINAL TARGET ARCHITECTURE DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (React)                               │
│                    Dashboard • Orders • Positions • Risk                     │
└───────────┬───────────────────────┬────────────────────────────────────────┘
            │                       │
    ┌───────▼────────┐     ┌────────▼────────┐
    │ HTTP (REST)    │     │ WebSocket       │
    │ (BAS APIs)     │     │ (Market Data)   │
    └───────┬────────┘     └────────┬────────┘
            │                       │
            ▼                       │
┌────────────────────────────────┐   │
│   EXECUTION PLANE (Synchronous)│   │
│   Ultra-Low Latency (<100ms)   │   │
│                              │   │
│  ┌───────────────────────────┐│   │
│  │ BROKER ADAPTER SERVICE    ││   │
│  │ • Stateless order execution     │      │  │                     │
│  │ • Minimal validation          │      │  │                     │
│  │ • Broker communication         │      │  │                     │
│  │ • Order → Broker execution     │      │  │                     │
│  │ • Idempotency ledger           │      │  │                     │
│  │ • Event publishing (Outbox)    │      │  │                     │
│  │                                │      │  │                     │
│  │  Sync call to:                 │      │  │                     │
│  │  - PBS/Broker (execute)        │      │  │                     │
│  │  (Quote/Inst from pre-cache)   │      │  │                     │
│  │                                │      │  │                     │
│  │  PRE-LOADED CACHES:            │      │  │                     │
│  │  - QuoteStore (async stream)   │      │  │                     │
│  │  - InstrumentCache (preload)   │      │  │                     │
│  └────────┬───────────────────────┘│   │
│           │                         │   │
│           │ (RPC + Internal WS)     │   │
│           │                         │   │
│  ┌────────▼─────────────────────┐  │   │
│  │ PBS / BROKER (Paper/Fyers)   │  │   │
│  │ • Fill at market price       │  │   │
│  │ • Position simulation        │  │   │
│  │ • Account state              │  │   │
│  │ • Internal WS to BAS (exec)  │  │   │
│  └─────────────────────────────┘  │   │
│                                   │   │
└───────────────────────────────────┘   │
                                        │
                                        │    ┌─────────────────────┐
                                        │    │                     │
                                        │    ▼                     │
                                        │  ┌─────────────────────┐ │
                                        │  │ MARKET DATA SERVICE │ │
                                        │  │ (MDS)               │ │
                                        │  │                     │ │
                                        │  │ • Quotes            │ │
                                        │  │ • Instruments       │ │
                                        │  │ • Trading calendar  │ │
                                        │  │ • WebSocket feed    │ │
                                        │  │                     │ │
                                        │  │ Redis Streams       │ │
                                        │  │ Publisher:          │ │
                                        │  │ market.quote     │ │
                                        │  └─────┬───────────────┘ │
                                        │        │ Redis Streams    │
                                        │        │ Consumer          │
                                        │        │                   │
                                        │        │                   │
                                        │        │                   │
         ┌──────────────────────────────────────┘                   │
         │                                                              │
         ▼                                                              │
┌───────────────────────────────────────────────────────┼────────────┐
│  EVENT BUS (Redis Streams)                            │            │
│                                                       │            │
│  Durable, ordered, idempotent event streaming        │            │
│  Critical events via Outbox pattern                   │            │
│  At-least-once delivery guarantee                     │            │
│                                                       │            │
│  Public Topics:                                       │            │
│  • order.updated (consolidated order lifecycle)              │
│  • trade.executed                                   │            │
│  • position.updated                                 │            │
│  • risk.limit_breach.v1, risk.metrics.updated.v1      │            │
│  • strategy.decision, strategy.paused.v1           │            │
│  • action.executed                                 │            │
│  • market.quote (from MDS)                    │            │
│  • market.* (quote, instrument)◄───────────────────────┘            │
└───────────┬─────────────────────────────────────────────────────────┘
            │
     ┌──────┴────────────────────────────────────────────────┐
     │                                                         │
     │      ASYNC/DATA PLANE (Event-Driven)                  │
     │      Flexible Latency (100ms - minutes)               │
     │                                                         │
     ├─────────────────────────────────────────────────────┤  │
     │ PORTFOLIO SERVICE                                   │  │
     │ • Aggregated positions (all holdings)              │  │
     │ • Greeks (delta, gamma, vega, theta)               │  │
     │ • Portfolio P&L and metrics                        │  │
     │                                                     │  │
     │ Consumes: trade.executed, position.updated         │  │
     │ Publishes: (read-only API)                         │  │
     ├─────────────────────────────────────────────────────┤  │
     │ JOURNAL SERVICE                                     │  │
     │ • Trade history (complete audit trail)             │  │
     │ • Action audit logs (PIE decisions)                │  │
     │ • Performance analytics                             │  │
     │ • Behavioral learning data                         │  │
     │                                                     │  │
     │ Consumes: ALL events                               │  │
     │ Publishes: (read-only API)                         │  │
     ├─────────────────────────────────────────────────────┤  │
     │ NOTIFICATION SERVICE                               │  │
     │ • Real-time alerts (WebSocket, email, SMS)         │  │
     │ • User notification preferences                    │  │
     │ • Alert rate-limiting                              │  │
     │                                                     │  │
     │ Consumes: order.*, risk.*, strategy.paused         │  │
     │ Publishes: notification.sent                    │  │
     │ WebSocket: Account events to Frontend              │  │
     └─────────────────────────────────────────────────────┘  │
     │                                                         │
     │            WebSocket (Account Events)                 │
     │            ┌───────────────────────────┐                │
     │            │                           │                │
     └────────────┼───────────────────────────┘                │
                  │                                            │
                  ▼                                            │
     ┌─────────────────────────────────────────────────────────┐
     │              FRONTEND (WebSocket Client)                │
     └─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ INFRASTRUCTURE                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ • PostgreSQL (per-service: BAS, MDS, Journal, Portfolio, Notification)    │
│ • Redis Cluster (event bus, cache, idempotency)                            │
│ • Authentication Service (JWT auth, RBAC, user management)                  │
│ • Config Management (smarttrade-common library)                            │
│ • Monitoring & Observability (Prometheus, logs)                            │
│ • Docker Compose / Kubernetes deployment                                   │
└─────────────────────────────────────────────────────────────────────────────┘

NOTE: Strategy Service and AI Service are marked as FUTURE/PLANNED services.
Current implementation uses mock services for testing purposes.
User Settings Service is implemented but excluded from this update per user request.
Broker credential management is handled directly within BAS.

---

## 9. DOCUMENTATION UPDATE SUMMARY

### Services with Complete Interface Documentation (v4.0 Update)

**Core Trading Services**:
- ✅ **Broker Adapter Service (BAS)**: REST APIs, events, stateless architecture notes
- ✅ **Market Data Service (MDS)**: REST APIs, events, WebSocket, instrument master replication
- ✅ **Paper Broker Service (PBS)**: Internal REST APIs, internal WebSocket, market data consumption

**Async/Data Services**:
- ✅ **Portfolio Service**: REST APIs, events, in-memory caching, background scheduler
- ✅ **Journal Service**: REST APIs, events, idempotency patterns, data ownership
- ✅ **Notification Service**: REST APIs, WebSocket, unified event consumption, rate limiting

**Infrastructure Services**:
- ✅ **Authentication Service**: REST APIs, JWT token format, RBAC integration, security features

### Documentation Coverage Improvements

**Before Update**: ~47% interface documentation coverage
**After Update**: ~85% interface documentation coverage for core services

**Key Additions**:
- Complete REST API contracts for all core services
- WebSocket protocol documentation for all WebSocket endpoints
- Event schema consistency verification
- Data ownership and replication models
- Idempotency and caching patterns
- Cross-reference verification with actual implementations

### Architecture Alignment

**Current Implementation Status**:
- All documented services match actual codebase implementation
- REST API paths verified against service READMEs
- Event producers/consumers updated to reflect current state
- Strategy and AI services marked as mock implementations
- Broker Auth Service removed (credentials managed in BAS)
- User Settings Service excluded per user request

### Cross-Component Communication Documentation

**Event Bus**:
- Redis Streams with consumer groups documented
- Event naming conventions standardized (domain.action)
- Publisher/consumer relationships verified
- Critical vs non-critical event classification

**WebSocket Protocols**:
- 4 WebSocket endpoints fully documented
- Authentication mechanisms specified
- Message formats and action types defined
- Replay and heartbeat mechanisms documented

**Service-to-Service Communication**:
- REST API contracts for all inter-service communication
- Internal service communication (PBS ↔ BAS) documented
- Service discovery and circuit breaker patterns noted for future documentation

### Maintenance Notes

**Documentation Currency**: Updated as of 2026-05-16 to reflect current stateless implementation
**Verification Status**: All REST API paths cross-referenced with actual service implementations
**Architecture Alignment**: Document matches current codebase state for all documented services

COMMUNICATION SUMMARY:
  • Execution Plane: Synchronous only (Frontend ↔ BAS ↔ PBS/Broker)
  • Data Cache: Pre-loaded from async background processes (no sync MDS calls)
  • Event Bus: Async (Redis Streams with consumer groups; MDS publishes quotes)
  • Frontend WebSocket: 
    - Direct to MDS (market data only)
    - Direct to Notification Service (account events: orders, trades, positions, risk)
  • Service-to-Service: Via BaseServiceClient (REST read-only or events only)
```

---

## Summary: Critical Design Principles

| Principle | Implementation | Validation |
|-----------|---|---|
| **Ultra-Low Latency** | BAS execution <100ms; pre-cached data; minimal RPC only | Latency benchmark in CI |
| **Execution Independence** | BAS zero runtime deps (except PBS/broker) | Integration test: execute with all services down |
| **Pre-Cached Data** | Quote/Instrument caches fed by async streams; no sync lookups | Code review: grep for sync MDS calls in OrderHandler |
| **Event-Driven Async** | All non-critical services consume via Redis Streams | Consumer group lag <30s monitoring |
| **Data Ownership** | Single source per service; no duplication | RBAC + data ownership audit |
| **Idempotency** | Critical events via DomainEventPublisher + Outbox | Duplicate event injection test |
| **Strict Boundaries** | Execution Path = Frontend → BAS → Broker only | Architecture review enforcement |
| **Single Decision Authority** | BAS executes (not Strategy or Portfolio) | Code review + static analysis |
| **Durable Event Log** | Redis Streams; replay-safe; consumer group tracking | Consumer group lag dashboard |

---

**Next Steps**: 
1. ✅ Architecture finalized (v4.0)
2. 📋 5-Phase Refactoring Plan (BAS_SAFE_REFACTORING_PLAN.md ready)
3. 🔧 Implement Phase 1 (Quote/Instrument cache removal, 8-12h)
4. 📊 Monitoring + alerts for event lag, latency SLA
5. 🧪 Integration tests for execution independence

