# SmartTrade Final Target Architecture v4.0
## Execution Plane Optimization & Service Boundary Finalization

**Status**: Architecture Specification (Ready for Implementation)  
**Date**: 2026-04-20  
**Scope**: Microservices with ultra-low latency execution path  
**Critical Constraint**: BAS is execution kernel; ZERO runtime dependencies except broker  

---

## 1. SERVICE RESPONSIBILITIES (Strict Definitions)

### Core Responsibility Definitions

#### **Broker Adapter Service (BAS)** — EXECUTION KERNEL
**Responsibility**: Execute orders atomically with 100% correctness and <100ms latency.

**MUST OWN**:
- Order placement (immediate execution or queued)
- Order state machine (pending → filled → settled)
- Risk pre-execution validation (daily loss, position limits, margin)
- Broker communication (single source of truth)
- Execution idempotency (deduplication of duplicate requests)
- Position tracking (fill aggregation, P&L at order level)
- Execution event publishing (`order.filled.v1`, `trade.executed.v1`)
- WebSocket account events to frontend (BAS-originated only)
- ExecutionContext encapsulation (internal model for deterministic execution)

**MUST NOT DO**:
- ~~Maintain quote cache~~ (consume from Redis Streams)
- ~~Resolve instrument metadata~~ (query MDS on-demand or cache with MDS as source)
- ~~Store action logs~~ (publish action.executed.v1 events)
- ~~Calculate portfolio-level Greeks~~ (Portfolio Service)
- Call downstream services synchronously (all async via event responses)
- Determine trading signals/strategy logic (Strategy Service)
- Make trading decisions based on market conditions beyond risk validation
- Handle external trades or broker reconciliation (Phase 3 future)

**Data Ownership**:
- Orders (all states)
- Trades (execution record)
- Positions (per-order fills and settlement)
- Risk snapshots (per-moment validation state)
- Idempotency ledger (request deduplication)

**ExecutionContext Model** (Internal):
```
ExecutionContext {
  user_id: UUID
  account_id: UUID
  order_id: UUID
  positions_snapshot: Dict[symbol, Position]  # from BAS DB, pre-fetched
  risk_snapshot: RiskState                    # margin, daily loss, position limits
  quote_snapshot: Dict[symbol, Quote]        # from local QuoteStore
  instrument_snapshot: Dict[symbol, Instrument]  # from local InstrumentCache
  idempotency_key: str                        # for deduplication
  timestamp: datetime
}
```
**Purpose**: Encapsulates all data required for deterministic order execution WITHOUT external calls. Reduces non-determinism and latency variance.

**Plane**: **EXECUTION** (synchronous, low latency)

---

#### **Market Data Service (MDS)** — DATA PROVIDER
**Responsibility**: Provide authoritative, real-time quotes and instrument metadata.

**MUST OWN**:
- Quote distribution (Fyers real-time + Paper Broker mock quotes)
- Instrument metadata (symbols, exchange, contract details, validation rules)
- Trading calendar (market open/close, holidays)
- Quote publishing via Redis Streams (durable, ordered, idempotent)
- WebSocket market data feeds to frontend (real-time tickers)

**MUST NOT DO**:
- ~~Store trading data~~ (orders, trades, positions belong to BAS)
- ~~Execute orders~~ (BAS responsibility)
- Call other services for trading decisions
- Access BAS/PBS trading events (read-only RBAC violation)

**Data Ownership**:
- Quotes (latest snapshot + stream history)
- Instruments (metadata, validation rules)
- Trading calendar

**Plane**: **ASYNC/DATA** (event-driven publish)

---

#### **Paper Broker Service (PBS)** — MOCK BROKER
**Responsibility**: Emulate broker behavior for paper/sandbox trading (external dependency model).

**MUST OWN**:
- Order execution simulation (fill at market price, simulate latency)
- Position tracking in mock broker (per-order settlement)
- Account balance simulation (buying power, margin, dividend)

**MUST NOT DO**:
- Publish trading events (BAS publishes after PBS returns fill)
- Access event bus (NO event bus access; like external brokers Fyers/etc.)
- Subscribe to market data feeds
- Make trading decisions

**Data Ownership**:
- Mock account state (balance, positions, margin during execution only)

**Communication Model**: **Stateless RPC** (like calling external broker API)
- PBS is synchronous execution dependency only
- No subscription-based communication
- No event publishing/consuming

**Plane**: **EXECUTION** (synchronous, latency-sensitive)

---

#### **Strategy Service** — DECISION ENGINE
**Responsibility**: Evaluate trading signals asynchronously and generate execution recommendations (advisory only).

**MUST OWN**:
- Signal evaluation (technical indicators, market conditions)
- Rule engine (if-then-else trading logic)
- Strategy state machine (active, paused, error)
- Decision event publishing (`strategy.decision.v1`)

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
- MUST NOT process every `market_data.quote.v1` event if infrastructure cannot keep up

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
- Own or persist raw position data (BAS owns positions)
- Influence risk validation in execution path (inline risk only)

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

#### **Broker Auth Service** — CREDENTIAL MANAGEMENT
**Responsibility**: Manage broker credentials and authentication state.

**MUST OWN**:
- Credential encryption/decryption
- Broker session management
- Authentication token refresh

**MUST NOT DO**:
- Execute orders
- Make trading decisions

**Data Ownership**:
- Encrypted credentials
- Session tokens

**Plane**: **ASYNC/DATA** (stateless)

---

### Summary Table

| Service | Core Responsibility | Plane | Latency |
|---------|-------------------|-------|---------|
| BAS | Order execution + state | EXECUTION | <100ms |
| MDS | Quote/instrument data | ASYNC | ~100ms |
| PBS | Mock broker | EXECUTION | <50ms |
| Strategy | Signal evaluation | ASYNC | N/A |
| Journal | Audit trail | ASYNC | N/A |
| Portfolio | Risk aggregation | ASYNC | <500ms |
| Notification | Alert delivery | ASYNC | N/A |
| AI | Advisory insights | ASYNC | N/A |
| User Settings | Configuration | ASYNC | N/A |
| Broker Auth | Credentials | ASYNC | N/A |

---

## 2. EXECUTION PLANE vs ASYNC PLANE

### Execution Plane (Synchronous, Low Latency)

**Services**:
1. **BAS** (Order execution, risk validation, state machine)
2. **PBS** (Mock broker execution)

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
- Risk validation must complete before broker contact
- Latency SLA: <100ms from request to broker call
- Cannot afford async event processing delays or network I/O
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
1. **Strategy Service** (Evaluate signals, publish decisions)
2. **Portfolio Service** (Aggregate P&L, compute Greeks)
3. **Journal Service** (Record trades, behavioral learning)
4. **Notification Service** (Deliver alerts)
5. **AI Service** (Advisory insights)
6. **User Settings Service** (User configuration)
7. **Broker Auth Service** (Credential management)

**Characteristics**:
- Event-driven (Redis Streams pub/sub)
- Flexible latency (100ms to minutes acceptable)
- At-least-once delivery semantics
- Durable event sourcing (can replay from broker reconciliation)
- Asynchronous processing (background jobs acceptable)
- Database writes non-blocking to execution path

**Communication Pattern**:
```
BAS (publish order.filled.v1) → Redis Streams
                                    ↓
         ┌──────────────────────────┼──────────────────────────┐
         ↓                           ↓                          ↓
    Portfolio Service       Notification Service       Journal Service
   (aggregate P&L)         (alert user)                (record trade)
```

**Why Services are ASYNC**:
- **MDS**: Publishes quote events asynchronously; BAS consumes from local cache (not from MDS API calls during execution)
- **Strategy**: Decisions are recommendations, not execution; can be delayed
- **Portfolio**: P&L aggregation non-critical to execution; can be 500ms+ late (derived read model only)
- **Journal**: Audit trail; asynchronous logging acceptable
- **Notification**: Alerts can be delayed 100-500ms
- **AI**: Insights are advisory; no latency requirement
- **User Settings**: Configuration changes can be cached 60s
- **Broker Auth**: Token refresh can be asynchronous

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

### Rule 2: BAS Data Provisioning (Pre-Caching + Fallback Rejection Model)
**ALLOWED**: Local in-memory caches populated by asynchronous background processes
- Quotes: In-memory cache (fed by `market_data.quote.v1` stream consumer)
- Instruments: In-memory cache (preloaded at startup, refreshed every 6h)
- Calendar: In-memory cache (updated hourly via background job)

**NOT ALLOWED**: Synchronous REST calls to MDS during order execution  
**Reason**: Execution path must have ZERO runtime dependency on MDS network availability  
**Pattern**: Background refresh jobs ensure cache freshness; execution uses pre-cached data only  
**Latency**: <1ms (in-memory lookup)

**Cache Failure Policy** (Deterministic Fallback):
- **If QuoteStore has no data for symbol**: Reject order with `quote_not_available` error
- **If InstrumentCache missing symbol**: Reject order with `instrument_not_found` error
- **If RiskLimits not cached**: Reject order with `risk_config_unavailable` error
- **NO blocking network calls as fallback**; failure is immediate and deterministic
- **Rationale**: Better to reject with known error than to incur latency variance from network calls

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
- `GET /api/v1/quotes/{symbol}`
- `GET /api/v1/instruments/{symbol}`
- `GET /api/v1/calendar`
- Any REST call to MDS service

**Reason**: Adds unpredictable latency; execution must be deterministic  
**Rationale**: All needed data must be pre-cached before order execution begins  
**Pattern**: Background jobs refresh caches; execution reads from local cache only

---

### Rule 4: Strategy Service → MDS (Event-Only)
**ALLOWED**: Strategy subscribes to `market_data.quote.v1` events; uses cached quote data  
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
- Portfolio Service subscribes to `position.updated.v1`
- Journal Service subscribes to ALL events

**NOT ALLOWED**: REST calls between Strategy, Portfolio, Journal, Notification  
**Reason**: Decoupling + scalability

---

### Rule 7: Frontend → BAS WebSocket (Streaming)
**ALLOWED**: WebSocket connection for real-time events
- Account events: `order.*`, `trade.*`, `position.*`, `risk.*`
- Broadcast to frontend (not persisted)

**NOT ALLOWED**: Frontend polling BAS repeatedly  
**Reason**: WebSocket is lower overhead

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

**Layer 1: Redis Streams (`market_data.quote.v1`)**
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
- **Update mechanism**: Asynchronous consumer of `market_data.quote.v1` stream
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
2. Publish Redis Stream (market_data.quote.v1) [async, durable]
        ↓ ↓
        │ └→ Strategy/Portfolio/etc consume via stream [async]
        │
        └→ BAS stream consumer updates QuoteStore [async background job]
                ↓
        BAS execution uses local QuoteStore [synchronous, <1ms]
```

### Critical Rules

- **Rule**: `market_data.quote.v1` is source of truth for sequencing and audit trail
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
order.accepted.v1
├─ Triggered by: Order passes risk validation and is submitted to broker
├─ Producer: BAS
├─ Consumers: Journal, Notification, Strategy, Frontend
├─ Critical: NO (informational; execution already happened)
└─ Data: { account_id, order_id, symbol, quantity, price, type, broker_order_id, timestamp }

order.filled.v1
├─ Triggered by: PBS/Broker returns fill; recorded in BAS position
├─ Producer: BAS (after fill is persisted to DB)
├─ Consumers: Portfolio, Journal, Notification, Strategy, Frontend
├─ Critical: YES (Outbox) — MUST NOT be lost
└─ Data: { order_id, fill_price, fill_quantity, fill_timestamp, commission, avg_fill_price }

order.rejected.v1
├─ Triggered by: Risk validation fails or broker rejects
├─ Producer: BAS
├─ Consumers: Journal, Notification, Strategy, Frontend
├─ Critical: NO (informational)
└─ Data: { order_id, reason_code, reason_message, rejected_timestamp }
```

**RATIONALE**: Only 3 public events. Internal lifecycle events (validation, broker_submitted) are BAS-internal state transitions; not published. This simplifies event taxonomy and reduces event volume.

#### **Trade Domain** (Executed Trades)
```
trade.executed.v1
├─ Triggered by: order.filled.v1 processed into position
├─ Producer: BAS
├─ Consumers: Portfolio, Journal, Notification
├─ Critical: YES (Outbox)
└─ Data: { trade_id, order_id, symbol, quantity, price, fee, timestamp }

trade.settled.v1
├─ Triggered by: T+2 settlement (future)
├─ Producer: BAS
├─ Consumers: Portfolio, Journal
├─ Critical: YES (Outbox)
└─ Data: { trade_id, settlement_date, amount }
```

#### **Position Domain** (Aggregated Holdings)
```
position.updated.v1
├─ Triggered by: trade.executed.v1 or trade.settled.v1
├─ Producer: BAS
├─ Consumers: Portfolio, Notification
├─ Critical: NO (always computable from trades)
└─ Data: { account_id, symbol, quantity, avg_price, current_price, p_l }

position.closed.v1
├─ Triggered by: Position quantity = 0
├─ Producer: BAS
├─ Consumers: Portfolio, Journal, Notification
├─ Critical: NO
└─ Data: { account_id, symbol, realized_p_l, holding_period }
```

#### **Risk Domain** (Inline Risk + Async Alerts)
```
risk.limit_breach.v1
├─ Triggered by: Inline risk validation detects threshold breach
├─ Producer: BAS
├─ Consumers: Notification
├─ Critical: NO (notification only; execution already rejected)
└─ Data: { limit_type, current_value, threshold, account_id }

risk.metrics.updated.v1
├─ Triggered by: Portfolio Service computes Greeks/VaR
├─ Producer: Portfolio Service (derived from trades)
├─ Consumers: Frontend, AI Service (for insights)
├─ Critical: NO (advisory metrics only)
└─ Data: { portfolio_delta, portfolio_gamma, var_95, correlation_risk }
```

**IMPORTANT DISTINCTION**:
- **Inline Risk** (BAS execution-critical): margin, position limits, daily loss (all pre-cached or computed instantly)
- **Derived Risk** (async, eventual consistency): Greeks, VaR, correlation (computed by Portfolio Service via event processing)

#### **Action Domain** (PIE Actions)
```
action.executed.v1
├─ Triggered by: Auto-entry, kill-switch, or user-initiated rule
├─ Producer: BAS (PublishedByActionOrchestrator)
├─ Consumers: Journal, Strategy
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
market_data.quote.v1
├─ Triggered by: Fyers/Paper quote received
├─ Producer: MDS
├─ Consumers: BAS, PBS, Strategy (all consume via Redis Streams)
├─ Critical: NO (durable but lossy acceptable)
├─ Pattern: Redis Streams + KV snapshot
└─ Data: { instrument_id, ltp, bid, ask, timestamp, sequence_number }

market_data.instrument_updated.v1
├─ Triggered by: Instrument metadata change
├─ Producer: MDS
├─ Consumers: BAS, Strategy (cached)
├─ Critical: NO
└─ Data: { instrument_id, exchange, lot_size, tick_size }
```

#### **Strategy Domain** (Signal Evaluation)
```
strategy.decision.v1
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
notification.sent.v1
├─ Triggered by: Alert delivery
├─ Producer: Notification Service
├─ Consumers: Journal
├─ Critical: NO
└─ Data: { notification_id, event_type, user_id, channel, timestamp }
```

### Event Publishing Rules

**Critical Events** (use Outbox Pattern — MUST NOT be Lost):
- `order.filled.v1` (execution record; basis for P&L, positions)
- `trade.executed.v1` (execution completion record)

**Optional-Critical Events** (use Outbox if available, but not mandatory):
- `position.updated.v1` (derived from trade; helps Portfolio consistency)
- `market_data.quote.v1` (Redis Streams with consumer group tracking)

**Non-Critical Events** (standard publish, lossy acceptable):
- `order.rejected.v1` (informational)
- `risk.limit_breach.v1` (alert only; risk already enforced by execution rejection)
- `action.executed.v1` (audit trail; not critical to execution)
- `strategy.decision.v1` (advisory; not critical)
- `notification.sent.v1` (delivery tracking)

**RATIONALE**: Minimize Outbox overhead. Only events critical to execution authority (orders, trades, positions) use durable Outbox.

### Consumer Group Strategy

| Service | Consumer Group | Events | Behavior |
|---------|---|---|---|
| BAS | bas-consumer | market_data.quote.v1 | Idempotent; skip if already processed |
| PBS | pbs-consumer | market_data.quote.v1 | Idempotent; update quote cache |
| Strategy | strategy-consumer | order.*.v1, trade.*.v1 | Evaluate signals on order fill |
| Portfolio | portfolio-consumer | trade.*.v1, position.*.v1 | Aggregate positions, compute Greeks |
| Journal | journal-consumer | ALL events | Audit trail; never drop |
| Notification | notification-consumer | order.*.v1, risk.*.v1 | Alert user on key events |
| AI | ai-consumer | trade.*.v1 | Behavioral learning |

---

## 5. SERVICE CONTRACTS (High-Level)

### BAS Contracts

#### **REST APIs** (Public)
```
POST   /api/v1/orders                        # Place order
GET    /api/v1/orders/{id}                  # Get order details
GET    /api/v1/orders?status=pending        # List orders
DELETE /api/v1/orders/{id}                  # Cancel order

GET    /api/v1/positions                    # Get all positions
GET    /api/v1/positions/{symbol}           # Get position details

GET    /api/v1/risk/snapshot                # Get current risk state
GET    /api/v1/risk/daily-pnl               # Get daily P&L

GET    /api/v1/actions                      # Get action audit trail
GET    /api/v1/actions?status=executed      # List actions by status
```

#### **Events Published** (Public Only)
```
order.accepted.v1
order.filled.v1          # CRITICAL (Outbox)
order.rejected.v1

trade.executed.v1        # CRITICAL (Outbox)
trade.settled.v1

position.updated.v1      # Optional-critical
position.closed.v1

risk.limit_breach.v1

action.executed.v1
```

**Note**: Internal lifecycle events (validated, broker_submitted, placement_initiated) are BAS-internal state transitions; not published.

#### **Events Consumed**
```
market_data.quote.v1 (via Redis Streams consumer group, populates QuoteStore cache)
```

#### **Data Provisioning** (Pre-Cached, Not Synchronous)
```
QuoteStore:
  - Populated by market_data.quote.v1 event stream consumer
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
/ws/account/{account_id}
  ├─ order.* (filtered to account)
  ├─ trade.*
  ├─ position.*
  └─ risk.*
```

---

### MDS Contracts

#### **REST APIs** (Public)
```
GET    /api/v1/quotes/{symbol}              # Latest quote
GET    /api/v1/quotes?symbols=NSE:INFY      # Multiple quotes

GET    /api/v1/instruments                  # All instruments
GET    /api/v1/instruments/{symbol}         # Instrument details

GET    /api/v1/calendar                     # Trading calendar (cached)
GET    /api/v1/calendar?exchange=NSE        # Calendar by exchange
```

#### **Events Published**
```
market_data.quote.v1 (via Redis Streams)
market_data.instrument_updated.v1
```

#### **WebSocket Streams**
```
/ws/quotes
  ├─ subscribe(symbols: [NSE:INFY, BSE:INFY])
  └─ Receive quote updates (streaming)
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
strategy.decision.v1
strategy.paused.v1
strategy.resumed.v1
```

#### **Events Consumed**
```
order.filled.v1
trade.executed.v1
market_data.quote.v1 (if needed)
```

---

### Portfolio Service Contracts

#### **REST APIs** (Public)
```
GET    /api/v1/portfolio/summary            # Aggregate portfolio stats
GET    /api/v1/portfolio/positions          # All positions (aggregated)
GET    /api/v1/portfolio/greeks             # Greeks (delta, gamma, vega, theta)
GET    /api/v1/portfolio/pnl                # Real-time P&L
```

#### **Events Consumed**
```
trade.executed.v1
position.updated.v1
trade.settled.v1
```

---

### Journal Service Contracts

#### **REST APIs** (Public)
```
GET    /api/v1/journal/trades               # Trade history
GET    /api/v1/journal/actions              # Action audit trail
GET    /api/v1/journal/analytics            # Performance analytics
GET    /api/v1/journal/actions?status=executed  # Filter actions
```

#### **Events Consumed**
```
ALL events (complete audit trail)
```

---

### Notification Service Contracts

#### **REST APIs** (Public)
```
GET    /api/v1/notifications/preferences    # User preferences
POST   /api/v1/notifications/preferences    # Update preferences
GET    /api/v1/notifications/history        # Recent notifications
```

#### **Events Consumed**
```
order.filled.v1
order.cancelled.v1
order.error.v1
risk.limit_breach.v1
strategy.paused.v1
```

---

## 6. EXTRACTION READINESS FROM BAS

### Components Currently in BAS

| Component | Current Location | Target Service | Readiness | Effort | Timeline |
|-----------|---|---|---|---|---|
| **Quote Cache** | `quote_store.py` | MDS (consume via Redis Streams) | Safe NOW | 6h | Phase 1 (Weeks 1-2) |
| **Instrument Cache** | `instrument_cache.py` | MDS (REST + 24h TTL cache) | Safe NOW | 8h | Phase 1 (Weeks 1-2) |
| **Action Logging** | `action_log_service.py` | Journal (event publishing) | Phased | 6h | Phase 4 (Weeks 7-8) |
| **AutoEntry Logic** | `entry_service.py` | Strategy Service | Phased | 20h | Phase 2 (Weeks 3-4) |
| **KillSwitch Logic** | `kill_switch_service.py` | Strategy Service | Phased | 18h | Phase 3 (Weeks 5-6) |
| **Portfolio Engine** | `portfolio_service.py` | Portfolio Service | Future | 40h | Q3 (after Phase 5) |
| **PIE (full)** | `pie/` | Strategy Service | Future | 60h | Q4 (after Phase 5) |

### Extraction Phases (5-Phase Strangler Pattern)

#### **Phase 1: Market Data Caches** (Weeks 1-2, 8-12h)
**Goal**: Remove local quote/instrument caches; consume from MDS

**Changes**:
1. QuoteStore: Receive from `market_data.quote.v1` stream (not local updates)
2. InstrumentCache: Replace with MDS REST calls + Redis cache (24h TTL)
3. Stop MarketDataConsumer from writing to local caches

**Risk**: LOW (quote cache is lossy; missing quotes acceptable for risk validation)
**Validation**: Quote availability in risk checks; instrument resolution latency

**Rollback**: Feature flag `BAS_QUOTE_SOURCE=local_cache`

---

#### **Phase 2: AutoEntry Decision/Execution Split** (Weeks 3-4, 16-20h)
**Goal**: Separate AutoEntry decision logic from execution

**Changes**:
1. Create `AutoEntryDecisionService` (pure logic, no execution)
2. Publish `action.executed.v1` event BEFORE ordering
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
2. Publish `action.executed.v1` event BEFORE position exit
3. PositionHandler reads decision, executes exit
4. Prepare extraction to Strategy Service (future Phase 6)

**Risk**: MEDIUM (similar to Phase 2)
**Validation**: Cooldown logic unchanged; position exit unchanged

**Rollback**: Feature flag `BAS_KILL_SWITCH_FLOW=legacy`

---

#### **Phase 4: Event-Driven Logging** (Weeks 7-8, 12-16h)
**Goal**: Convert action logging from sync DB writes to event publishing

**Changes**:
1. ActionLogService publishes `action.executed.v1` → Journal Service
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
        publish(order.filled)     # BAS emits event
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
    
    # Portfolio listens to order.filled event independently
    await publisher.publish(OrderFilledEvent(order))
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

✅ **CORRECT**: Single source of truth
```python
# ✅ RIGHT: BAS owns positions; Portfolio reads from events
class PositionEngine:
    positions: Dict[symbol, Position]  # Single source
    
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
# ❌ WRONG: MDS consuming order.filled
class MDSEventConsumer:
    async def on_order_filled(self, event):
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
    validate_risk()  # BAS validation only
    await broker.execute(order)
    
    # Strategy listens to order.filled event; might publish alert
    await publisher.publish(OrderFilledEvent(order))
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
    
    # Journal Service subscribes to trade.executed.v1 independently
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
└──────────────────────────┬──────────────────────────┬────────────────────────┘
                           │                          │
                 ┌─────────▼──────────┐       ┌──────▼──────────┐
                 │ HTTP / WebSocket   │       │ WebSocket       │
                 │ (Account Events)   │       │ (Market Data)   │
                 └────────────┬───────┘       └────────┬────────┘
                              │                        │
        ┌─────────────────────┴──────────────────┐    │
        │                                         │    │
        ▼                                         │    ▼
┌────────────────────────────────────────┐      │  ┌─────────────────────┐
│   EXECUTION PLANE (Synchronous)        │      │  │ MARKET DATA SERVICE │
│   Ultra-Low Latency (<100ms)           │      │  │ (MDS)               │
│                                        │      │  │                     │
│  ┌─────────────────────────────────┐  │      │  │ • Quotes            │
│  │  BROKER ADAPTER SERVICE (BAS)   │  │      │  │ • Instruments       │
│  │  • Order state machine          │  │      │  │ • Trading calendar  │
│  │  • Risk validation              │  │      │  │ • WebSocket feed    │
│  │  • Position tracking            │  │      │  │                     │
│  │  • Order → Broker execution     │  │      │  │ Redis Streams       │
│  │  • Idempotency ledger           │  │      │  │ Publisher:          │
│  │  • Event publishing (Outbox)    │  │      │  │ market_data.quote.v1│
│  │                                 │  │      │  └─────┬──────────────┘
│  │  Sync call to:                  │  │      │        │
│  │  - PBS/Broker (execute)         │  │      │        │ Redis Streams
│  │  (Quote/Inst from pre-cache)   │  │      │        │ Consumer
│  │                                 │  │      │        │
│  │  PRE-LOADED CACHES:             │  │      │        │
│  │  - QuoteStore (async stream)   │  │      │        │
│  │  - InstrumentCache (preload)   │  │      │        │
│  └────────┬────────────────────────┘  │      │        │
│           │                            │      │        │
│           │ (only RPC to broker, <50ms) │   │        │
│           │                            │      │        │
│  ┌────────▼────────────────────────┐  │      │        │
│  │  PBS / BROKER (Paper/Fyers)     │  │      │        │
│  │  • Fill at market price         │  │      │        │
│  │  • Position simulation          │  │      │        │
│  │  • Account state                │  │      │        │
│  └────────────────────────────────┘  │      │        │
│                                        │      │        │
└────────────────────────────────────────┘      │        │
                                                │        │
         ┌──────────────────────────────────────┘        │
         │                                               │
         ▼                                               │
┌───────────────────────────────────────────────────────┼────────────┐
│  EVENT BUS (Redis Streams)                            │            │
│                                                       │            │
│  Durable, ordered, idempotent event streaming        │            │
│  Critical events via Outbox pattern                   │            │
│  At-least-once delivery guarantee                     │            │
│                                                       │            │
│  Public Topics:                                       │            │
│  • order.accepted.v1, order.filled.v1, order.rejected.v1           │
│  • trade.executed.v1, trade.settled.v1                │            │
│  • position.updated.v1, position.closed.v1            │            │
│  • risk.limit_breach.v1, risk.metrics.updated.v1      │            │
│  • strategy.decision.v1, strategy.paused.v1           │            │
│  • action.executed.v1                                 │            │
│  • market_data.quote.v1 (from MDS)                    │            │
│  • market_data.* (quote, instrument_updated)◄─────────┘            │
└───────────┬─────────────────────────────────────────────────────────┘
            │
     ┌──────┴────────────────────────────────────────────────┐
     │                                                         │
     │      ASYNC/DATA PLANE (Event-Driven)                  │
     │      Flexible Latency (100ms - minutes)               │
     │                                                         │
     ├─────────────────────────────────────────────────────┐  │
     │ STRATEGY SERVICE                                    │  │
     │ • Signal evaluation (rules, indicators)             │  │
     │ • Decision publishing (strategy.decision.v1)        │  │
     │ • Strategy state (active, paused)                   │  │
     │                                                     │  │
     │ Consumes: order.filled, trade.executed             │  │
     │ Publishes: strategy.decision.v1                     │  │
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
     │ Publishes: notification.sent.v1                    │  │
     ├─────────────────────────────────────────────────────┤  │
     │ AI SERVICE (Advisory Only)                         │  │
     │ • Market analysis (LLM-powered)                    │  │
     │ • Trading insights (non-autonomous)                │  │
     │ • Performance commentary                           │  │
     │                                                     │  │
     │ Consumes: trade.executed (learning)                │  │
     │ Publishes: (read-only API)                         │  │
     ├─────────────────────────────────────────────────────┤  │
     │ USER SETTINGS SERVICE                              │  │
     │ • Trading preferences (leverage, risk)             │  │
     │ • Strategy parameters                              │  │
     │ • Notification settings                            │  │
     │ • API key management                               │  │
     │                                                     │  │
     │ Consumes: (none)                                   │  │
     │ Publishes: (none)                                  │  │
     ├─────────────────────────────────────────────────────┤  │
     │ BROKER AUTH SERVICE                                │  │
     │ • Credential encryption/decryption                 │  │
     │ • Broker session management                        │  │
     │ • Token refresh                                    │  │
     │                                                     │  │
     │ Consumes: (none)                                   │  │
     │ Publishes: (none)                                  │  │
     └─────────────────────────────────────────────────────┘  │
     │                                                         │
     └─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ INFRASTRUCTURE                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ • PostgreSQL (per-service: BAS, MDS, Journal, Portfolio)                   │
│ • Redis Cluster (event bus, cache, idempotency)                            │
│ • Authentication Service (JWT, RBAC)                                       │
│ • Config Management (smarttrade-common library)                            │
│ • Monitoring & Observability (Prometheus, logs)                            │
│ • Docker Compose / Kubernetes deployment                                   │
└─────────────────────────────────────────────────────────────────────────────┘

COMMUNICATION SUMMARY:
  • Execution Plane: Synchronous only (Frontend ↔ BAS ↔ PBS/Broker)
  • Data Cache: Pre-loaded from async background processes (no sync MDS calls)
  • Event Bus: Async (Redis Streams with consumer groups; MDS publishes quotes)
  • Frontend WebSocket: Direct to BAS (account events) + MDS (market data)
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

