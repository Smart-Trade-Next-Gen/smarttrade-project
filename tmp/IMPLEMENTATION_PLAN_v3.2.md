# SmartTrade Implementation Plan v3.2 FINAL
## High-Level Roadmap & Skills Matrix

**Date**: 2026-03-26
**Version**: 3.2 FINAL (Production-Ready)
**Current Status**: Partial implementation (Auth ✅, MDS ✅, BAS 70%, Frontend 30%)
**Target**: Production-grade AI-driven algo trading platform
**Execution First**: Build core execution layer, then enhance strategy/signal generation

---

## PATCH NOTES v3.2 FINAL

### Critical Additions for Production Correctness

**Phase 1: Execution Foundation**
✅ Minimal broker sync (pre-execution checks)
✅ Enhanced order states (CREATED → VALIDATED → SENT → ACK → PENDING → PARTIAL → FILLED → CANCELLED → REJECTED)
✅ Explicit idempotency scope (user_id + client_order_id, 24h window)
✅ Rate limiting requirement (per-user + global broker limits)

**Phase 2: Orchestrator**
✅ Explicit orchestrator responsibilities (validate → idempotency → risk re-check → execute → audit)
✅ Rate limit awareness in resilience layer
✅ Latency budget (50ms p99 for order placement)

**Phase 2.5: Stabilization**
✅ Warm restart optimization (snapshots reduce startup latency)
✅ Enhanced metrics (per-user queue depth, broker latencies)

**Phase 3: Broker Sync**
✅ Hybrid model (WebSocket primary + polling fallback)
✅ Drift classification (LOW/MEDIUM/HIGH with different actions)
✅ External trade safety rules (pause only affected strategies, recalculate portfolio)
✅ Explicit: Broker is source of truth

**Phase 4: Strategy Runtime**
✅ Service boundary notes (initially in BAS, future separate service)
✅ Sandbox security (timeout, memory limit, no file access)
✅ Explicit deduplication (cooldown window, suppress duplicates)
✅ Risk layer enforcement (all execution through orchestrator)

**Global**
✅ Concurrency model definition (actor per user, no shared state)
✅ Feature flags (safe rollout of new components)
✅ Environment modes (mock, paper, live)
✅ Kill switch hierarchy (strategy → user → global)

---

## PART 1: HIGH-LEVEL IMPLEMENTATION PLAN

### Overall Vision
SmartTrade moves from **static portfolio tracking** to **dynamic execution-driven trading**:
- Phase 1-2: Secure execution foundation + concurrency safety
- Phase 2.5: Stabilization & edge case handling
- Phase 3: Broker sync + external trade handling
- Phase 4: Algo trading + signal generation
- Phase 5: Multi-broker + options + advanced features

### Implementation Principles
1. **Execution-First**: Correct order execution > feature richness
2. **Safety-Critical**: No data loss, no race conditions, no crossed orders
3. **Incremental**: Small, validated steps; small capital for real testing
4. **Event-Driven**: All communication via typed events (Redis/Kafka)
5. **User-Isolated**: Every query filters by user_id; no cross-user data leakage
6. **Financial Correctness**: Decimal only, ACID transactions, immutable audit logs

---

## GLOBAL EXECUTION PATTERNS (Apply Across All Phases)

### Concurrency Model (MANDATORY)
```
One UserActor per user_id:
- Single-threaded message queue
- All operations for that user serialized
- No shared mutable state across users

Example:
  User 123 (actor_123) → queue: [fill, order, cancel, query]
  User 456 (actor_456) → queue: [order, fill] (independent, parallel)

Guarantee: No race conditions within a user's operations.
Trade-off: Some operations queued (acceptable for safety).
```

### Feature Flags (For Safe Rollout)
```python
# In settings.json or env vars
ENABLE_EXECUTION_ORCHESTRATOR = true       # Route orders through orchestrator
ENABLE_BROKER_SYNC = false                 # Enable broker reconciliation
ENABLE_STRATEGY_RUNTIME = false            # Enable user-defined strategies
ENABLE_SIGNAL_ENGINE = false               # Enable automated signals

# Gradual rollout:
# Week 1: all false (use legacy path)
# Week 2: ENABLE_EXECUTION_ORCHESTRATOR = true, rest false
# Week 3: + ENABLE_BROKER_SYNC = true
# Week 4: + ENABLE_STRATEGY_RUNTIME = true (staged users only)
```

### Environment Modes
```
mock:    Paper trading only (no real capital)
         - Orders go to mock broker
         - Fills simulated (realistic with slippage)
         - Real capital: $0
         - Use for: Development, testing, learning

paper:   Real account, small capital
         - Orders go to real broker
         - Real fills, real slippage
         - Real capital: $1-10k (test amount)
         - Use for: Phase validation, edge case discovery

live:    Real account, full capital
         - Orders go to real broker
         - Real fills, real capital
         - Real capital: Full deployment
         - Use for: Production trading
```

### Kill Switch Hierarchy (Nested Safety)
```
Level 1: Strategy-level kill switch
         - Pauses specific strategy
         - Other strategies continue
         - User can resume

Level 2: User-level kill switch
         - Closes all user's positions
         - Cancels all pending orders
         - Pauses all strategies
         - User cannot resume (requires admin)

Level 3: Global system kill switch
         - Closes all positions (all users)
         - Cancels all pending orders (all users)
         - Pauses entire platform
         - Emergency only (requires admin + senior eng)

Trigger: Manual button OR automated (daily loss limit, API error rate >10%)
```

---

## PART 2: PHASE-WISE BREAKDOWN

### PHASE 1: Execution Foundation (Weeks 1-3)
**Goal**: Build safe, deterministic order execution with actor model concurrency

#### Components
1. **UserActor System** (NEW)
   - Per-user message queue (priority-based)
   - Atomic message processing
   - Per-user state management
   - Backpressure & queue overflow handling
   - Actor per user_id (no shared mutable state across users)

2. **Execution Orchestrator** (NEW in BAS)
   - Idempotency key tracking (within 24h window)
   - Audit event generation (immutable)
   - Request validation & risk pre-check
   - Order state transitions
   - Rate limit enforcement (per-user throttle + global broker limits)

3. **Order Lifecycle State Machine** (ENHANCE)
   - Formal states: CREATED → VALIDATED → SENT → ACK → PENDING → PARTIAL → FILLED → CANCELLED → REJECTED
   - State transition validation (prevent invalid transitions)
   - Immutable state history (audit trail)

4. **Idempotency Framework** (NEW in smarttrade-common)
   - Idempotency key: `user_id + client_order_id`
   - Validity window: 24 hours (same key returns same result, no duplicate execution)
   - Replay detection (check if already processed)
   - Duplicate request handling (return cached result)

5. **Minimal Broker Sync** (NEW)
   - On startup: fetch open orders + positions from broker
   - Pre-execution check: fetch latest broker state before placing order
   - Alignment verification: internal state ≈ broker state (within tolerance)
   - Purpose: Catch broker side effects, prevent conflict with external trades
   - Frequency: Once on startup, once per order (optional, trade latency vs accuracy)

#### Dependencies
- Auth Service ✅ (already complete)
- Database transactions (existing)
- Event bus (existing Redis)

#### Success Criteria
- [ ] UserActor system passes race condition tests (concurrent fills + orders)
- [ ] Execution Orchestrator passes idempotency tests (same key = same result)
- [ ] Idempotency window: 24 hours (keys expire after 24h)
- [ ] Order state machine: all 9 states working, invalid transitions rejected
- [ ] Minimal broker sync: pre-execution check passes 100% of tests
- [ ] Rate limiting: per-user throttle + global broker limit respected
- [ ] All state changes are immutable + audited
- [ ] <100 orders/sec throughput sustained
- [ ] Zero race conditions (verified with ThreadSanitizer / async tests)

#### Services Affected
- **smarttrade-common**: Add IdempotencyService, UserActor base
- **broker-adapter-service**: Add ExecutionOrchestrator, OrderStateManager

#### Effort
- Implementation: 10-15 developer-days
- Testing: 7-10 days
- Risk mitigation: 3-5 days

---

### PHASE 2: Execution Orchestrator + Concurrency (Weeks 4-5)
**Goal**: Multi-order concurrency, conflict resolution, broker resilience

#### Execution Orchestrator Responsibilities (EXPLICIT)
```
Per-order execution:
1. Validate request format + user authorization
2. Check idempotency (same key = return cached result)
3. Re-validate risk (ensure order respects daily loss, position limits)
4. Emit audit event (pre-execution)
5. Call Broker API with retry logic
6. Track order lifecycle (CREATED → ACK → PENDING → FILLED)
7. Emit confirmation event (post-execution)
8. Update actor state (immutable)
```

#### Components
1. **Execution Queue per User** (ENHANCE UserActor)
   - Priority queue (fills > orders > cancellations)
   - Message batching (group multiple orders, respect rate limits)
   - Timeout handling (max wait per message)
   - Backpressure (stop accepting if queue size > threshold)

2. **Conflict Resolution Engine** (NEW)
   - Detect simultaneous orders on same symbol
   - Prevent over-leveraging (total position would exceed limit)
   - Resolve collisions (configurable: oldest wins, newest wins, deny all)
   - Configurable per strategy or user preference

3. **Broker Resilience Layer** (NEW)
   - Exponential backoff with jitter (avoid thundering herd)
   - Circuit breaker (fail-fast on broker outage, auto-recover)
   - Timeout enforcement (per-call, not global)
   - Fallback to mock broker (graceful degradation)
   - Rate limit awareness (respect broker's 10 req/sec, 100 orders/day)

4. **Order Timeout Handler** (NEW)
   - Pending order TTL (configurable, default 5 minutes)
   - Auto-cancel stale orders (emit event, update state)
   - Event notification (alert user on timeout)

5. **Latency Budget** (NEW)
   - Strategy → Execution Orchestrator: <10ms
   - Execution Orchestrator → Broker API: <20ms
   - Total order placement: <50ms p99
   - Monitoring: Track latencies, alert on violations

#### Dependencies
- Phase 1: UserActor, ExecutionOrchestrator
- smarttrade-common: resilience patterns (already have CircuitBreaker)

#### Success Criteria
- [ ] 5 concurrent orders from same user: no conflicts
- [ ] Broker outage: graceful degradation (switch to mock)
- [ ] Latency <50ms p99 for order placement
- [ ] <10% order timeout rate
- [ ] Load test: 500 active users, 50 orders/sec

#### Services Affected
- **broker-adapter-service**: ConflictResolver, BrokerResilienceLayer
- **paper-broker-service**: Enhanced to handle concurrent requests

#### Effort
- Implementation: 8-12 days
- Testing & load validation: 10-15 days

---

### PHASE 2.5: Stabilization (Weeks 6-7)
**Goal**: Handle edge cases, improve reliability, real-world testing

#### Components
1. **Partial Fill Handling** (ENHANCE)
   - Multiple fills per order (sum to total)
   - Position tracking after partial fills (running balance)
   - Correct P&L calculation (FIFO / LIFO accounting)
   - Average entry price updates

2. **Order Amendment Support** (NEW)
   - Modify pending orders (price, qty)
   - Validate amendments vs broker (broker-specific rules)
   - Immutable amendment history (audit trail)
   - Idempotency on amendments

3. **Warm Restart Optimization** (NEW)
   - Periodic actor state snapshot (every 100 messages or 5 minutes)
   - Reload from snapshot on restart (instead of full replay)
   - Reduces startup latency, improves availability
   - Replay remaining messages from snapshot point

4. **Error Recovery** (ENHANCE)
   - Crash recovery (resume from last successful state / snapshot)
   - Broker outage recovery (automatic reconnect with backoff)
   - Network reconnection handling (WebSocket re-subscribe)
   - Graceful degradation (switch to mock broker if real unreachable)

5. **Enhanced Observability** (ENHANCE)
   - Structured logging (JSON with request_id, user_id, trade_id, actor_id)
   - Prometheus metrics:
     * Per-user actor queue depth (measure backpressure)
     * Order latencies (strategy → broker)
     * Broker API latencies (per-call)
     * Broker availability (success/failure rate)
     * Fill rate (actual fills vs orders)
   - Distributed tracing (OpenTelemetry, correlate order flow)
   - Alert thresholds (order timeout, risk breach, queue overflow)

#### Dependencies
- Phase 1-2: Complete execution layer

#### Success Criteria
- [ ] Partial fills: 100% accurate P&L
- [ ] Crash recovery: state consistent after restart
- [ ] Broker outage: <5min to auto-switch to mock
- [ ] All errors logged with context
- [ ] Metrics dashboard available

#### Services Affected
- **broker-adapter-service**: PartialFillHandler, AmendmentManager
- **smarttrade-common**: Enhanced logging, metrics

#### Effort
- Implementation: 8-10 days
- Real-world testing: 5-10 days

---

### PHASE 3: Broker Sync & Reconciliation (Weeks 8-10)
**Goal**: Handle external trades, reconcile with broker, prevent conflicts

**Broker as Source of Truth**: Broker state always overrides internal state.

#### Components
1. **Hybrid Broker Sync Model** (NEW)
   - Primary: WebSocket real-time feeds (fills, order updates)
   - Fallback: Polling (hourly reconciliation if WebSocket down)
   - Reduces latency (real-time via WebSocket), improves reliability (polling fallback)

2. **Trade Ingestion** (NEW)
   - Async capture of trades entered outside SmartTrade
   - Webhook (if broker supports) + polling (fallback)
   - Event emission (trade.external_entered)
   - Timestamp correlation (when was trade actually placed?)

3. **Broker Sync Engine** (NEW)
   - Periodic reconciliation (hourly)
   - Detect drift (SmartTrade position ≠ broker position)
   - Classify drift severity:
     * **LOW**: Minor mismatch (rounding errors, <0.1 share) → auto-correct
     * **MEDIUM**: Position mismatch (filled but not recorded) → alert user
     * **HIGH**: Unknown position (divergence >1% of total) → pause all strategies
   - Identify: missing fills, orphaned orders, external trades
   - Auto-correction (for LOW drift) or manual review (MEDIUM/HIGH)

4. **External Trade Safety Rules** (NEW)
   - Detect external trades on user's symbols
   - Pause ONLY affected strategies (not entire system)
   - Recalculate portfolio state (update P&L, position, margin)
   - Require user confirmation before next SmartTrade trade
   - Log external trade with audit trail
   - Reason: External trades affect portfolio risk; must recalibrate

5. **Portfolio Reconciliation** (NEW)
   - Balance sync (cash, margin utilization)
   - Corporate action handling (dividend reduces cash, split changes shares)
   - Multi-account aggregation (if user has multiple accounts)
   - Account-level reconciliation

6. **Audit Trail Verification** (ENHANCE)
   - Immutable event log (orders, fills, reconciliation decisions)
   - Event replay validation (can we reconstruct broker state from events?)
   - Regulatory compliance (audit trail stands up to audit)

#### Dependencies
- Phase 1-2: Execution layer complete

#### Success Criteria
- [ ] Hybrid sync working (WebSocket + polling fallback)
- [ ] External trade captured <2min after broker execution
- [ ] Drift detection: sensitivity 0.1 shares
- [ ] Drift classification: LOW/MEDIUM/HIGH working correctly
- [ ] Auto-correction (LOW drift): 100% accurate
- [ ] Manual review (MEDIUM/HIGH drift): alerts sent, no auto-action
- [ ] Reconciliation runs hourly, <5min duration
- [ ] Zero balance mismatches on 1-week test
- [ ] External trade pauses only affected strategies (others continue)
- [ ] User confirmation required after external trade
- [ ] Corporate action processing: 100% accurate
- [ ] Broker state always overrides internal state

#### Services Affected
- **broker-adapter-service**: SyncEngine, TradeIngestor, BalanceReconciler
- **market-data-service**: Corporate action ingestion

#### Effort
- Implementation: 12-15 days
- Testing: 8-10 days
- Integration with broker APIs: 5-7 days

---

### PHASE 4: Strategy Runtime + Signal Engine (Weeks 11-15)
**Goal**: Execute user-defined strategies, generate trading signals

**Service Boundary Note**: Strategy Runtime initially implemented in BAS (same process as ExecutionOrchestrator). Future: move to separate service as scale increases (isolate user code, independent scaling).

#### Components
1. **Strategy Runtime** (ENHANCE existing)
   - User strategy execution (Python, compiled sandbox)
   - Safe execution sandbox:
     * Timeout (max 5s per candle processing)
     * Memory limit (max 100MB per strategy instance)
     * No infinite loops, no blocking I/O
     * Restricted builtins (no file access, no subprocess)
   - Strategy state persistence (snapshots to DB)
   - Multi-strategy isolation (one crashing strategy doesn't affect others)
   - All execution MUST go through ExecutionOrchestrator (cannot bypass risk layer)

2. **Signal Engine** (NEW)
   - Indicator calculation (SMA, EMA, RSI, MACD, Bollinger Bands)
   - Cross-market pattern detection (correlation, divergence)
   - Signal generation (BUY, SELL, HOLD with confidence)
   - Signal deduplication (prevent spam)
   - Backtest integration (signal history for replay)

3. **Signal Deduplication** (NEW - EXPLICIT)
   - Prevent duplicate signals on same symbol:
     * Cooldown window (e.g., 1 hour)
     * Track last signal timestamp per strategy/symbol
     * Reject signal if within cooldown window
   - Duplicate suppression (same signal type on same symbol)
   - Signal strength decay (confidence decreases with age)
   - Cross-strategy conflict resolution (when signals conflict)

4. **Backtest Engine** (NEW)
   - Historical OHLC data feed (from MDS)
   - Slippage modeling (bid-ask spread, market impact)
   - Latency simulation (order placement delay, fill delay)
   - Realistic fill simulation (limit orders may not fill, slippage)
   - Commission deduction (per-trade + per-share)
   - Performance metrics: Sharpe ratio, max drawdown, win rate, trade count

5. **Scheduler** (ENHANCE)
   - Distributed task scheduling (APScheduler + Redis backend)
   - Strategy triggers:
     * Time-based (every minute, every hour, specific time)
     * Signal-based (on signal received)
     * Event-based (on fill, on order rejection)
   - Cron expressions (5-field: minute hour day month weekday)
   - Timezone handling (convert user timezone to UTC, execute UTC)

#### Dependencies
- Phase 1-3: Execution + sync complete

#### Success Criteria
- [ ] Simple strategy runs 1000+ candles without crash (sandbox working)
- [ ] Strategy cannot crash system (timeout + memory limit enforced)
- [ ] All strategy execution routed through ExecutionOrchestrator (risk layer enforced)
- [ ] Backtest matches live within 5% (realistic slippage + latency)
- [ ] Signal deduplication: cooldown prevents duplicate fires
- [ ] Signal latency <100ms (from data → signal generation)
- [ ] Multi-strategy: 10 strategies on same symbol work correctly (no conflicts)
- [ ] Strategy state persists across restarts (snapshots working)
- [ ] Cross-strategy conflict resolution working (same symbol, conflicting signals)

#### Services Affected
- **broker-adapter-service**: StrategyRuntime enhancement, SignalEngine
- **market-data-service**: Indicator calculation, backtest data
- **new micro-service (optional)**: Dedicated Strategy Scheduler

#### Effort
- Implementation: 15-20 days
- Testing & backtesting: 12-15 days
- Real-world validation: 10-15 days

---

### PHASE 5: Multi-Broker + Options + Advanced (Weeks 16-20)
**Goal**: Broker agnosticity, options trading, AI enhancement

#### Components
1. **Multi-Broker Support** (ENHANCE)
   - Zerodha, Interactive Brokers plugins
   - Unified order submission
   - Broker-specific quirk handling

2. **Options Trading** (NEW - depends on MDS)
   - Multi-leg order support (spreads, straddles, iron condor)
   - Greeks-based risk (delta, theta, vega)
   - Assignment handling
   - Margin optimization

3. **AI-Assisted Trading** (NEW - optional Phase 5B)
   - ML signal generation
   - Sentiment analysis
   - Anomaly detection
   - Chat interface to Claude API

4. **Portfolio Rebalancing** (NEW)
   - Target allocation maintenance
   - Drift detection
   - Automated rebalancing

5. **Advanced Risk** (NEW)
   - Value at Risk (VaR)
   - Greeks-based hedging
   - Correlation monitoring
   - Stress testing

#### Dependencies
- Phase 1-4: All execution layers complete

#### Success Criteria
- [ ] 2+ brokers switchable at runtime
- [ ] Iron condor strategy executes end-to-end
- [ ] Greeks calculation <1s for 100-leg strategy
- [ ] AI signal beats baseline by 10% (Sharpe)
- [ ] Portfolio drift <5% before rebalance triggered

#### Services Affected
- **broker-adapter-service**: Multi-broker adapters
- **market-data-service**: Option Greeks enhancement
- **new service**: PortfolioRebalancer
- **new service (optional)**: AI Signal Generator

#### Effort
- Multi-broker: 10-15 days
- Options: 20-25 days
- AI (optional): 15-20 days

---

## PART 3: SERVICE-WISE IMPLEMENTATION PLAN

### smarttrade-common (Shared Library)
**Current**: Auth, config, database, events, errors, resilience
**Phase 1 Additions**:
- IdempotencyService (key + replay detection)
- UserActor base class
- Order state machine (enums, validators)

**Phase 2 Additions**:
- ConflictResolver interface
- Timeout manager
- Enhanced logging + metrics

**Phase 3 Additions**:
- Audit trail validation
- Portfolio reconciliation schemas

**Phase 4 Additions**:
- Strategy runtime sandbox
- Signal deduplication rules

**Phase 5 Additions**:
- Multi-broker adapter interface
- Options Greeks calculator

### Authentication Service
**Current**: Complete ✅
**Phase 1+**: No changes (stable)
**Future Enhancements** (not critical):
- 2FA/MFA
- OAuth2 (social login)
- Service-to-service token rotation

### Broker Adapter Service (BAS)
**Current**: 70% complete (orders, positions, risk, kill switch, PIE)
**Phase 1**:
- Add ExecutionOrchestrator class
- Integrate UserActor message queue
- Order state machine

**Phase 2**:
- Add ConflictResolver
- Add BrokerResilienceLayer
- Enhance order timeout handling

**Phase 2.5**:
- PartialFillHandler
- AmendmentManager
- Enhanced error recovery

**Phase 3**:
- SyncEngine (reconciliation loop)
- TradeIngestor (external trade capture)
- BalanceReconciler

**Phase 4**:
- Enhance StrategyRuntime (already partial)
- Add SignalEngine
- Add SignalDeduplicator
- Integrate scheduler

**Phase 5**:
- Multi-broker adapters (Zerodha, IBKR)
- Options strategy support
- Portfolio rebalancer

### Market Data Service (MDS)
**Current**: 80% complete (instruments, quotes, options, Greeks)
**Phase 1**: No changes
**Phase 2**: No changes
**Phase 2.5**: Enhanced metrics/observability
**Phase 3**: Corporate action ingestion
**Phase 4**:
- Backtest data feed (historical candles)
- Indicator calculation library
- Signal pattern detection

**Phase 5**:
- Options Greeks enhancement (multi-leg)
- Volatility surface real-time updates
- Margin calculation (complex legs)

### Mock Service
**Current**: Basic paper trading
**Phase 1**: Concurrent request handling
**Phase 2**: Load testing support (1000 concurrent)
**Phase 3**: Synthetic trade injection
**Phase 4**: Backtest oracle
**Phase 5**: Multi-instrument simulation

### Frontend
**Current**: 30% complete (dashboard, order entry, chart)
**Phase 1**: Integration with ExecutionOrchestrator (order placement)
**Phase 2**: Concurrency feedback (pending orders status)
**Phase 2.5**: Error notifications (timeouts, broker outage)
**Phase 3**: Sync status dashboard
**Phase 4**:
- Strategy builder (UI)
- Backtest viewer
- Signal notifications

**Phase 5**:
- Options chain viewer + Greeks
- AI chat interface
- Portfolio analytics

---

## PART 4: REQUIRED SKILLS PER PHASE

### PHASE 1: Execution Foundation

#### Backend Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Async Python** (asyncio, locks, queues) | Deep | UserActor, message queue | MUST-HAVE |
| **Actor Model concepts** | Intermediate | System design | MUST-HAVE |
| **FastAPI** | Intermediate | Orchestrator routes | MUST-HAVE |
| **Database transactions (SQLAlchemy)** | Deep | Idempotency, audit logs | MUST-HAVE |
| **Event-driven architecture** | Intermediate | Message queue, events | GOOD-TO-HAVE |
| **Testing with async code** | Deep | Race condition tests | MUST-HAVE |

#### Domain Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Order lifecycle** (states, transitions) | Intermediate | State machine design | MUST-HAVE |
| **Broker API basics** (Fyers) | Basic | Validation layer | GOOD-TO-HAVE |

#### System Design Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Concurrency patterns** (locks, queues, actors) | Deep | UserActor | MUST-HAVE |
| **Error handling in distributed systems** | Intermediate | Idempotency, timeouts | MUST-HAVE |
| **Audit trail design** | Intermediate | Immutable logs | GOOD-TO-HAVE |

---

### PHASE 2: Concurrency & Resilience

#### Backend Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Concurrency** (race conditions, deadlocks) | Deep | Conflict resolution | MUST-HAVE |
| **Circuit breaker pattern** | Intermediate | Broker resilience | MUST-HAVE |
| **Exponential backoff** | Basic | Retry logic | GOOD-TO-HAVE |
| **Load testing** | Intermediate | Performance validation | MUST-HAVE |

#### Domain Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Conflict resolution strategies** | Intermediate | Multiple simultaneous orders | MUST-HAVE |
| **Broker rate limits** | Basic | Backoff calculation | GOOD-TO-HAVE |

---

### PHASE 2.5: Stabilization

#### Backend Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Observability** (logging, metrics, tracing) | Intermediate | Prometheus, OpenTelemetry | MUST-HAVE |
| **Crash recovery patterns** | Intermediate | State reconstruction | MUST-HAVE |
| **Time zone handling** | Basic | Order timing | GOOD-TO-HAVE |

#### Domain Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Order amendments** (broker API) | Intermediate | Fyers amendment API | GOOD-TO-HAVE |
| **Partial fill semantics** | Intermediate | P&L calculations | MUST-HAVE |

---

### PHASE 3: Broker Sync & Reconciliation

#### Backend Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Polling + webhooks** | Intermediate | Trade ingestion | MUST-HAVE |
| **Reconciliation algorithms** (diff, merge) | Intermediate | Balance sync | MUST-HAVE |
| **Idempotency revisited** | Deep | External trade handling | MUST-HAVE |

#### Domain Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Broker trade feeds** | Deep | Fyers WebSocket, REST | MUST-HAVE |
| **Corporate actions** (dividend, split) | Intermediate | MDS integration | GOOD-TO-HAVE |
| **Position reconciliation** | Deep | Book vs reality | MUST-HAVE |

#### System Design Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Eventual consistency** | Intermediate | Reconciliation cadence | GOOD-TO-HAVE |
| **Conflict resolution (external vs internal)** | Intermediate | When sync disagrees | MUST-HAVE |

---

### PHASE 4: Strategy Runtime + Signal Engine

#### Backend Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Sandbox execution** (timeout, resource limits) | Deep | Safe strategy execution | MUST-HAVE |
| **Historical data feeds** | Intermediate | Backtest engine | MUST-HAVE |
| **Scheduling** (APScheduler, cron) | Intermediate | Strategy triggers | MUST-HAVE |
| **Time series analysis** | Intermediate | Indicators (SMA, EMA) | GOOD-TO-HAVE |

#### Domain Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Trading strategies** (momentum, mean reversion) | Intermediate | Strategy examples | GOOD-TO-HAVE |
| **Technical indicators** (SMA, RSI, MACD) | Intermediate | Signal generation | MUST-HAVE |
| **Backtesting frameworks** | Deep | Realistic simulation | MUST-HAVE |
| **Slippage modeling** | Intermediate | Realistic fills | GOOD-TO-HAVE |
| **Sharpe ratio, drawdown metrics** | Basic | Performance eval | GOOD-TO-HAVE |

#### System Design Skills
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Signal deduplication** | Intermediate | Prevent spam | MUST-HAVE |
| **Strategy state persistence** | Intermediate | Crash recovery | MUST-HAVE |
| **Multi-strategy isolation** | Intermediate | Same symbol conflicts | MUST-HAVE |

---

### PHASE 5: Multi-Broker + Options + AI

#### Backend Skills (Multi-Broker)
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Adapter pattern** | Intermediate | Broker abstraction | MUST-HAVE |
| **Protocol translation** | Intermediate | Order DTO mapping | MUST-HAVE |

#### Backend Skills (Options)
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Black-Scholes model** | Deep | Greeks calculation | MUST-HAVE |
| **Implied volatility** | Deep | IV smile, surface | MUST-HAVE |
| **Multi-leg order logistics** | Deep | Spread execution | MUST-HAVE |

#### Domain Skills (Multi-Broker)
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Zerodha API** (order, position, session) | Intermediate | Adapter implementation | GOOD-TO-HAVE |
| **Interactive Brokers API** | Intermediate | Adapter implementation | GOOD-TO-HAVE |
| **Broker-specific quirks** | Intermediate | Workaround engineering | MUST-HAVE |

#### Domain Skills (Options)
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Option strategies** (spreads, straddles, iron condor) | Deep | Multi-leg execution | MUST-HAVE |
| **Greeks trading** (delta, theta, vega hedging) | Deep | Risk management | MUST-HAVE |
| **Assignment logistics** (early assignment risk) | Intermediate | Exercise handling | GOOD-TO-HAVE |
| **Volatility smile + skew** | Intermediate | Pricing edge detection | ADVANCED |

#### Backend Skills (AI)
| Skill | Depth | Used In | Priority |
|-------|-------|---------|----------|
| **Claude API + SDKs** | Intermediate | AI signal generation | OPTIONAL |
| **ML model serving** | Intermediate | Signal pipeline | OPTIONAL |
| **Sentiment analysis** | Basic | News ingestion | OPTIONAL |

---

## PART 5: KNOWLEDGE DOMAINS (DETAILED)

### Domain 1: Broker APIs (Fyers + Future Multi-Broker)

**Why Needed**: Direct broker integration is the critical path for execution. Every order goes through broker API; misunderstanding causes lost trades or financial damage.

**Fyers API Coverage** (PHASE 1-3):
- Order placement (market, limit, SL, SL-M, bracket)
- Order statuses + transitions
- Websocket real-time fills
- Rate limits (10 req/sec, 100 orders/day)
- Error codes + handling
- Session management (access token refresh)
- Account info (balance, margin, holdings)

**Advanced Topics** (PHASE 5):
- Zerodha Kite API
- Interactive Brokers IBAPI
- Broker-specific order types
- Margin calculations per broker

**Learning Path**:
1. Read Fyers API docs (2 hours)
2. Implement basic order placement in mock service (4 hours)
3. Real-world test with small capital (2 days)
4. Edge case handling (broker goes down, slow responses) (3 days)

---

### Domain 2: Trading Fundamentals

**Why Needed**: Core domain knowledge required for correct order execution, position tracking, P&L calculation.

**Core Concepts** (PHASE 1-2):
- Order types: market, limit, stop-loss, stop-loss-market
- Order states: new, pending, partial, filled, cancelled, rejected
- Position lifecycle: open → modify → close
- P&L calculation: entry price, exit price, slippage
- Margin: initial, maintenance, utilization
- Brokerage: per order, per share, percentage
- T+1 settlement (T = trade day)
- Corporate actions: dividend (cash), split (share), bonus

**Advanced Topics** (PHASE 3-4):
- Partial fills + cost averaging
- Short selling + borrow costs
- Margin call triggers
- Portfolio rebalancing
- Multi-leg settlement

**Learning Path**:
1. Trading basics (book: "The Intelligent Trader") (5 days)
2. Order lifecycle walkthrough (with Fyers API) (2 days)
3. P&L calculation exercises (1 day)
4. Real trading observation (paper trading 1 week)

---

### Domain 3: Options Trading (PHASE 5+)

**Why Needed**: Options are complex but high-value. Required for advanced traders; optional for stocks-only traders.

**Core Concepts**:
- Call vs put (directional bets)
- Strike price, expiration date
- Intrinsic value + time value
- Greeks: delta (directional), theta (time decay), vega (volatility), gamma (delta change)
- Implied volatility (market expectation)
- IV smile / skew (non-uniform IV across strikes)

**Strategies**:
- Covered call (sell upside for income)
- Protective put (buy downside protection)
- Iron condor (sell upside + downside, keep middle)
- Butterfly (bet on low volatility)
- Spread (buy far strike, sell near strike)

**Advanced Topics**:
- Multi-leg assignment (early exercise risk)
- Volatility surface modeling
- Greeks-based hedging
- Correlation trading

**Learning Path**:
1. Options basics (book: "Options as a Strategic Investment") (7 days)
2. Greeks calculation + Black-Scholes (2 days)
3. Volatility analysis (IV smile, surface) (2 days)
4. Strategy backtesting (3 days)
5. Real options paper trading (1 week)

---

### Domain 4: Algo Trading & Strategy Design

**Why Needed**: Core feature; users want to automate strategies. SmartTrade's primary value is strategy execution.

**Core Concepts** (PHASE 4):
- Signal definition (technical indicator → BUY/SELL)
- Strategy state (position state, signal state)
- Entry conditions (when to open position)
- Exit conditions (when to close position, stop loss, take profit)
- Risk per trade (max loss on position)
- Position sizing (Kelly criterion, fixed %)

**Technical Indicators**:
- Simple moving average (SMA): trend following
- Exponential moving average (EMA): faster response
- Relative strength index (RSI): overbought/oversold
- Moving average convergence divergence (MACD): momentum
- Bollinger bands (volatility-based entry)
- Volume-weighted average price (VWAP)

**Advanced Topics** (PHASE 5):
- Multi-timeframe analysis
- Correlation between assets
- Regime detection (bull vs bear)
- Dynamic position sizing
- ML signal generation

**Backtesting**:
- Historical data simulation
- Slippage modeling (realistic fills)
- Latency simulation (execution delay)
- Commission deduction
- Metrics: Sharpe ratio, max drawdown, win rate

**Learning Path**:
1. Strategy design (book: "Algorithmic Trading") (5 days)
2. Indicator calculations + library (TA-Lib) (2 days)
3. Backtesting framework design (2 days)
4. Implement 3 strategies + backtest (5 days)
5. Real-money testing (paper trading 2 weeks)

---

### Domain 5: Distributed Systems & Concurrency

**Why Needed**: Multi-order concurrency, race conditions, broker interactions are distributed system problems. Correctness is critical.

**Core Concepts** (PHASE 1-2):
- Race conditions (concurrent access to shared state)
- Mutual exclusion (locks, semaphores)
- Deadlocks (circular wait)
- Actor model (isolated state, message passing)
- Message queues (FIFO ordering, backpressure)
- Idempotency (same request → same result)

**Advanced Topics** (PHASE 3-4):
- Eventual consistency
- Conflict resolution (when replication diverges)
- Distributed consensus (complex, not needed for SmartTrade v1)
- Circuit breaker pattern
- Timeout + retry strategies

**Learning Path**:
1. Concurrency fundamentals (book: "The Art of Multiprocessor Programming") (5 days)
2. asyncio deep dive (Python docs + exercises) (3 days)
3. Actor model in Python (2 days)
4. Distributed systems patterns (2 days)
5. Load testing + race condition testing (3 days)

---

### Domain 6: Async Python & Event-Driven Systems

**Why Needed**: SmartTrade is async-first. All I/O (DB, broker API, WebSocket) is async. Blocking I/O = dead orders.

**Core Concepts**:
- asyncio: event loop, coroutines, tasks
- async/await syntax
- Locks (asyncio.Lock, asyncio.Semaphore)
- Queues (asyncio.Queue with priorities)
- Event (asyncio.Event)
- WebSocket (aiohttp, websockets library)
- Database (asyncpg, async SQLAlchemy)

**Advanced Topics**:
- Context variables (async context injection)
- Cancellation tokens (graceful shutdown)
- Timeouts + timeout handling
- Error propagation in async chains

**Learning Path**:
1. asyncio tutorial (Real Python) (2 days)
2. Async web server (FastAPI) (1 day)
3. Database async access (asyncpg/SQLAlchemy) (2 days)
4. WebSocket real-time data (2 days)
5. Error handling in async chains (1 day)
6. Race condition debugging (2 days)

---

### Domain 7: Broker Reconciliation & Sync

**Why Needed**: Prevent split-brain scenarios (SmartTrade position ≠ broker position). Critical for multi-day trading.

**Core Concepts** (PHASE 3):
- Book vs reality (internal state vs broker state)
- Drift detection (periodic reconciliation)
- External trade capture (trades entered outside SmartTrade)
- Orphaned order handling (order placed but not tracked)
- Balance sync (cash balance, margin)
- Corporate action impact (dividend reduces cash, split changes share count)

**Algorithms**:
- Reconciliation: iterate SmartTrade positions + broker positions, find mismatches
- Drift classification: missing fills, orphaned orders, miscalculation
- Auto-correction: safe fixes (mark as filled), manual review (override)
- Audit trail: immutable log of all reconciliation decisions

**Learning Path**:
1. Broker API deep dive (session management, WebSocket) (3 days)
2. Reconciliation algorithm design (1 day)
3. External trade ingestion (webhook/polling) (2 days)
4. Corporate action modeling (1 day)
5. Real-world testing (1 week)

---

### Domain 8: Financial Risk Management

**Why Needed**: Prevent catastrophic losses. Risk limits are guardrails.

**Core Concepts**:
- Daily loss limit (max loss per calendar day)
- Per-trade risk (max loss per trade)
- Position limit (max quantity per symbol)
- Margin utilization (max leverage)
- VaR (value at risk, probability of loss)
- Greeks-based risk (delta, vega, theta)

**Risk Engines**:
- Pre-execution checks (will this order breach limits?)
- Post-execution monitoring (are we within limits?)
- Kill switch (close all on limit breach)

**Learning Path**:
1. Risk management basics (book: "Risk Management") (3 days)
2. VaR + Greeks calculation (2 days)
3. Risk rules design (YAML-driven rules) (1 day)
4. Testing edge cases (limit breaches) (2 days)

---

## PART 6: SKILLS MATRIX (Phase × Service × Skill)

| Skill | Phase 1 | Phase 2 | Phase 2.5 | Phase 3 | Phase 4 | Phase 5 | Services |
|-------|--------|--------|-----------|---------|---------|---------|----------|
| **Async Python** | MUST | MUST | MUST | MUST | MUST | MUST | All |
| **Actor Model** | MUST | MUST | — | — | — | — | BAS |
| **FastAPI** | MUST | MUST | MUST | MUST | MUST | MUST | All |
| **Database (Async)** | MUST | MUST | MUST | MUST | MUST | MUST | All |
| **Order Lifecycle** | MUST | MUST | MUST | MUST | MUST | MUST | BAS |
| **Fyers API** | GOOD | GOOD | GOOD | MUST | MUST | — | BAS |
| **Concurrency** | MUST | MUST | MUST | GOOD | GOOD | GOOD | BAS |
| **Circuit Breaker** | — | MUST | MUST | GOOD | — | — | BAS |
| **Observability** | — | — | MUST | MUST | MUST | MUST | All |
| **Partial Fills** | — | — | MUST | MUST | MUST | MUST | BAS |
| **Broker Sync** | — | — | — | MUST | MUST | — | BAS, MDS |
| **Trading Domain** | GOOD | GOOD | GOOD | MUST | MUST | MUST | BAS, MDS |
| **Backtesting** | — | — | — | MUST | MUST | MUST | BAS, MDS |
| **Indicators (SMA, RSI)** | — | — | — | MUST | MUST | MUST | MDS, BAS |
| **Black-Scholes** | — | — | — | — | — | MUST | MDS |
| **Multi-broker Adapters** | — | — | — | — | — | MUST | BAS |
| **Options Strategies** | — | — | — | — | — | MUST | BAS |
| **Load Testing** | — | MUST | MUST | GOOD | GOOD | GOOD | All |

---

## PART 7: KNOWLEDGE GAPS (Current State → Production)

### Critical Gaps (Block Phase 1-2)
| Gap | Severity | Impact | Mitigation |
|-----|----------|--------|-----------|
| **Actor model in Python** | Critical | Concurrency correctness | Study asyncio + design pattern (2-3 days) |
| **Async race conditions** | Critical | Lost orders, data corruption | Implement tests, use tools (pytest-asyncio) |
| **Broker API edge cases** | Critical | Exchange errors not handled | Fyers API docs + real testing (1 week) |

### Important Gaps (Phase 3-4)
| Gap | Severity | Impact | Mitigation |
|-----|----------|--------|-----------|
| **Broker reconciliation** | Important | Split-brain scenarios | Algo design + real testing (2 weeks) |
| **Backtesting realism** | Important | Backtest-to-live divergence | Implement slippage + latency models (1 week) |
| **Greeks calculation** | Important | Options edge cases | Black-Scholes + IV surface (1 week) |
| **Multi-broker APIs** | Important | Vendor lock-in | Read Zerodha + IBKR docs (2 days) |

### Optional Gaps (Phase 5B+)
| Gap | Severity | Impact | Mitigation |
|-----|----------|--------|-----------|
| **ML signal generation** | Optional | Competitive edge | Study ML trading (2-3 weeks) |
| **Sentiment analysis** | Optional | News-driven signals | Use transformers library (1 week) |

---

## PART 8: LEARNING PLAN (Parallel to Implementation)

### Timeline: Weeks 1-20 (20-week roadmap)

#### WEEKS 1-3: Phase 1 (Execution Foundation)

**Learning** (parallel to implementation):
1. **Async Python Deep Dive** (3 days effort)
   - Read: Real Python asyncio tutorial
   - Code: Implement priority queue with asyncio.Queue
   - Test: Race condition tests with pytest-asyncio
   - Outcome: Can identify and fix async bugs

2. **Actor Model Patterns** (2 days effort)
   - Read: Actor model concepts (papers / blogs)
   - Code: Simple actor system in Python
   - Outcome: Understand message queues + state isolation

3. **Order Lifecycle Design** (1 day effort)
   - Read: Trading basics (order states)
   - Code: State machine in Python enums
   - Outcome: Can validate order transitions

4. **Fyers API Walkthrough** (2 days effort)
   - Read: Fyers API documentation
   - Code: Order placement in mock service
   - Outcome: Know order placement flow

**Learning Checkpoints**:
- [ ] Can identify race condition in concurrent order code
- [ ] Can design simple actor system
- [ ] Can list all order states + transitions
- [ ] Can place order via Fyers mock API

---

#### WEEKS 4-5: Phase 2 (Concurrency & Resilience)

**Learning** (2 weeks):
1. **Concurrency Patterns** (3 days)
   - Read: Distributed systems book (concurrency chapter)
   - Code: Implement conflict resolver
   - Test: 5 concurrent orders on same symbol
   - Outcome: Understand conflict scenarios + resolution

2. **Circuit Breaker + Retry** (2 days)
   - Read: Release It! (circuit breaker chapter)
   - Code: Implement circuit breaker for broker
   - Test: Broker outage scenario
   - Outcome: Can build resilient broker calls

3. **Load Testing** (1 day)
   - Read: Load testing best practices
   - Code: Load test with locust
   - Outcome: Know latency targets (p99 <100ms)

**Learning Checkpoints**:
- [ ] Can explain 5 conflict scenarios
- [ ] Can design conflict resolution rule
- [ ] Can implement circuit breaker
- [ ] Can run load test + measure p99

---

#### WEEKS 6-7: Phase 2.5 (Stabilization)

**Learning** (2 weeks):
1. **Observability Stack** (2 days)
   - Read: Prometheus + OpenTelemetry basics
   - Code: Add structured logging
   - Code: Add Prometheus metrics
   - Outcome: Can log with context + measure latency

2. **Crash Recovery** (1 day)
   - Read: State machine recovery patterns
   - Code: Design recovery from DB state
   - Outcome: Can trace execution from last successful state

3. **Partial Fill Handling** (1 day)
   - Read: Partial fill scenarios
   - Code: P&L recalculation logic
   - Test: Multiple fills, P&L accuracy
   - Outcome: Can handle partial fills correctly

**Learning Checkpoints**:
- [ ] Can add JSON logging with request_id
- [ ] Can add Prometheus metrics
- [ ] Can recover actor state after crash
- [ ] Can calculate P&L with partial fills

---

#### WEEKS 8-10: Phase 3 (Broker Sync & Reconciliation)

**Learning** (3 weeks):
1. **Broker Feeds + Webhooks** (3 days)
   - Read: Fyers WebSocket / REST API
   - Code: Ingest external trades
   - Test: Simulate external trade flow
   - Outcome: Can receive + parse external trades

2. **Reconciliation Algorithms** (2 days)
   - Read: Reconciliation patterns
   - Code: Diff algorithm + merge logic
   - Test: 10 scenarios (missing fills, orphans, etc.)
   - Outcome: Can detect all drift types

3. **Corporate Actions** (1 day)
   - Read: Dividend + split impact on positions
   - Code: Adjust position for corporate action
   - Test: Dividend payout reduces cash correctly
   - Outcome: Can handle dividends + splits

**Learning Checkpoints**:
- [ ] Can receive external trade via webhook
- [ ] Can detect 5+ types of position drift
- [ ] Can auto-correct safe drift cases
- [ ] Can log all reconciliation decisions

---

#### WEEKS 11-15: Phase 4 (Strategy Runtime + Signal Engine)

**Learning** (5 weeks):
1. **Technical Indicators** (3 days)
   - Read: TA-Lib documentation
   - Code: Implement SMA, EMA, RSI, MACD
   - Test: Indicator calculations vs reference
   - Outcome: Can calculate all common indicators

2. **Backtesting Framework** (3 days)
   - Read: Event-driven backtest design
   - Code: OHLC replay engine
   - Code: Slippage + latency modeling
   - Test: Backtest vs paper trading (should match)
   - Outcome: Can run realistic backtest

3. **Strategy Design + Sandbox** (2 days)
   - Read: Strategy design patterns
   - Code: User strategy sandbox (timeout, resource limits)
   - Test: Malicious strategy (infinite loop) → timeout
   - Outcome: Can safely execute user strategies

4. **Scheduling + State Persistence** (1 day)
   - Read: APScheduler documentation
   - Code: Strategy state → Redis/DB
   - Test: Crash + recovery scenario
   - Outcome: Can persist + recover strategy state

**Learning Checkpoints**:
- [ ] Can calculate indicators on live candles
- [ ] Can backtest strategy with 100%+ realism (vs paper)
- [ ] Can execute user strategy without crashes
- [ ] Can recover strategy state after restart

---

#### WEEKS 16-20: Phase 5 (Multi-Broker + Options + AI)

**Learning** (5 weeks):
1. **Multi-Broker Adapters** (2 days)
   - Read: Zerodha Kite API + IB IBAPI
   - Code: Adapter for Zerodha
   - Test: Orders on 2 brokers, same instrument
   - Outcome: Can add new broker in 2 days

2. **Options Greeks** (3 days)
   - Read: Black-Scholes model + IV
   - Code: Greeks calculator (delta, theta, vega)
   - Code: IV smile + surface modeling
   - Test: Greeks vs Fyers Greeks (should match)
   - Outcome: Can price complex option strategies

3. **Multi-Leg Execution** (2 days)
   - Read: Option strategies (spreads, straddles)
   - Code: Multi-leg order orchestration
   - Test: Iron condor placement + settlement
   - Outcome: Can execute complex option trades

4. **AI Signal Generation** (Optional, 2 days)
   - Read: ML trading basics + sentiment analysis
   - Code: Claude API integration for signals
   - Test: Signal generation accuracy
   - Outcome: Can generate AI-assisted signals

**Learning Checkpoints**:
- [ ] Can place orders on 2+ brokers
- [ ] Can calculate option Greeks within 1% of market
- [ ] Can execute iron condor end-to-end
- [ ] Can generate AI signals (optional)

---

## PART 9: EXECUTION STRATEGY

### Safe Execution Approach

#### Principle 1: Start Small, Scale Safely
```
Week 1:  Paper trading (mock service)
Week 2-3: Paper trading (1000 INR capital max)
Week 4-5: Real capital (10,000 INR, small positions)
Week 6-7: Real capital (50,000 INR, moderate positions)
Week 8+:  Scaling (no hard cap, but monitored)
```

#### Principle 2: Feature Flags for High-Risk Changes
```python
# In config:
ENABLE_EXECUTION_ORCHESTRATOR = env.get("ENABLE_EXECUTION_ORCHESTRATOR", False)
ENABLE_BROKER_SYNC = env.get("ENABLE_BROKER_SYNC", False)

# In code:
if config.ENABLE_EXECUTION_ORCHESTRATOR:
    # Route through new orchestrator
else:
    # Use legacy path
```

#### Principle 3: Validation Checkpoints After Each Phase
```
Phase 1: ✅ 100 orders, 0 lost orders, 0 race conditions
Phase 2: ✅ 500 concurrent orders, <100ms p99, 0% timeout
Phase 2.5: ✅ Crash recovery works, state consistent
Phase 3: ✅ Drift detection catches 100% of external trades
Phase 4: ✅ Backtest matches live within 5%
Phase 5: ✅ 2+ brokers working, options Greeks <1% error
```

#### Principle 4: Testing at Multiple Levels
```
Unit Tests:     Pure logic (risk engine, P&L calc)
Integration:    Services + DB + event bus
E2E:            Full workflow (order → fill → settle)
Load Tests:     Concurrency (500 users, 50 orders/sec)
Real Money:     Paper trading → small capital → scale
```

#### Principle 5: Observability Throughout
```
Phase 1: Structured logging (JSON + request_id)
Phase 2: Prometheus metrics (latency, throughput)
Phase 3: Distributed tracing (OpenTelemetry)
Phase 4: Strategy metrics (Sharpe, drawdown)
Phase 5: Portfolio analytics dashboard
```

---

### Development Process Per Phase

#### 1. **Kickoff Meeting** (0.5 days)
- Review phase goals + success criteria
- Design API contracts (requests / responses)
- Plan task breakdown
- Identify blockers

#### 2. **Implementation** (5-15 days per phase)
- Code in feature branch
- Commit early + often
- Use feature flags for integration
- Daily check-ins on blockers

#### 3. **Testing** (3-10 days per phase)
- Unit tests (100% of business logic)
- Integration tests (service boundaries)
- E2E tests (full workflows)
- Load tests (performance targets)

#### 4. **Real-World Validation** (2-7 days per phase)
- Paper trading (mock service)
- Small capital trading (1,000-10,000 INR)
- Monitor for errors + edge cases
- Iterate on findings

#### 5. **Review + Merge** (1-2 days per phase)
- Code review for correctness
- Architecture review for design
- Performance review vs targets
- Merge to main, tag release

---

### Risk Mitigation Strategies

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|-----------|
| **Race condition → lost order** | Medium | Critical | Async tests, locks verification, load test |
| **Broker API changes** | Low | High | Version pinning, wrapper layer |
| **Broker outage** | Low | High | Mock broker fallback, circuit breaker |
| **Partial fill edge case** | Medium | High | Comprehensive test cases, real testing |
| **Split-brain (drift)** | Medium | High | Hourly reconciliation, audit trail |
| **Strategy infinite loop** | Low | High | Sandbox with timeout, resource limits |
| **Capital loss** | Low | Critical | Kill switch, daily loss limit, position limit |

---

## PART 10: SUCCESS CRITERIA (Go/No-Go per Phase)

### PHASE 1: Execution Foundation ✅ GO
- [x] UserActor system tested (100+ concurrent messages)
- [x] ExecutionOrchestrator idempotency proven
- [x] Order state machine covers edge cases
- [x] <100 orders/sec throughput

**Go-Live Decision**: Deploy to staging, 100 paper trades

---

### PHASE 2: Concurrency & Resilience ✅ GO
- [ ] 500 concurrent users, 50 orders/sec
- [ ] Broker outage: <5min failover to mock
- [ ] Conflict resolution: tested 10+ scenarios
- [ ] <100ms p99 order placement latency

**Go-Live Decision**: Deploy to production, small capital (10k INR)

---

### PHASE 2.5: Stabilization ✅ GO
- [ ] Crash recovery: state consistency 100%
- [ ] Partial fills: P&L accuracy within 0.1%
- [ ] Observability: all errors logged + metrics available
- [ ] 1-week production trading, 0 data loss

**Go-Live Decision**: Increase capital (50k INR)

---

### PHASE 3: Broker Sync ✅ GO
- [ ] External trade capture: <2min latency
- [ ] Drift detection: 100% accuracy (1000 test cases)
- [ ] Auto-reconciliation: zero manual fixes
- [ ] 2-week production trading, 0 balance mismatches

**Go-Live Decision**: Full capital deployment

---

### PHASE 4: Strategy Runtime ✅ GO
- [ ] Backtest realism: matches live within 5%
- [ ] Strategy execution: 10+ strategies, 0 crashes
- [ ] Signal deduplication: <1% false positives
- [ ] 1-month backtesting + paper trading

**Go-Live Decision**: Enable user-defined strategies

---

### PHASE 5: Multi-Broker + Options ✅ GO
- [ ] 2+ brokers, switchable at runtime
- [ ] Option Greeks: <1% error vs market
- [ ] Iron condor execution: end-to-end working
- [ ] AI signals: backtest shows 10%+ Sharpe improvement

**Go-Live Decision**: Feature-complete platform

---

## SUMMARY TABLE

| Phase | Duration | Key Components | Critical Skills | Success Metric | Go-Live | Capital |
|-------|----------|---|---|---|---|---|
| 1 | Weeks 1-3 | UserActor, Orchestrator, Order States (9), Idempotency (24h), Minimal Broker Sync, Rate Limiting | Async, Actor Model, Order Lifecycle, Idempotency | 100+ orders/sec, 0 race conditions, 0 broker conflicts | Paper trades | Mock |
| 2 | Weeks 4-5 | Execution Queue, Conflict Resolver, Circuit Breaker, Latency Budget (<50ms p99), Timeout Handler | Concurrency, Load Testing, Circuit Breaker | 500 users, 50 orders/sec, <100ms p99, rate limits respected | 10k INR | Small |
| 2.5 | Weeks 6-7 | Warm Restart, Crash Recovery, Partial Fills, Enhanced Observability (per-user metrics) | Observability, Warm Restart, Partial Fills | 1-week zero loss, all errors logged, metrics available | 50k INR | Moderate |
| 3 | Weeks 8-10 | Hybrid Sync (WebSocket+Polling), Drift Classification, External Trade Safety, Trade Ingestion | Broker APIs, Reconciliation, Drift Detection | 0 balance mismatches, external trades detected <2min, safety rules enforced | Full | Full |
| 4 | Weeks 11-15 | Strategy Sandbox, Signal Deduplication, Backtest, Scheduler, Risk Enforcement | Indicators, Backtesting, Sandbox Security, Deduplication | Backtest matches live ±5%, 0 crashes, cooldown working | Enable strategies | Full |
| 5 | Weeks 16-20 | Multi-broker Adapters, Options Support, AI Signals (optional) | Multi-broker, Greeks, ML | 2+ brokers working, Greeks <1% error, AI signals improve Sharpe | Production | Full |

---

## KEY SAFETY FEATURES (Across All Phases)

| Feature | Phase | Purpose | Impact |
|---------|-------|---------|--------|
| Concurrency Model (Actor per user) | 1-5 | No race conditions | Critical |
| Idempotency (24h window) | 1-5 | No duplicate orders | Critical |
| Broker as Source of Truth | 3-5 | Prevent split-brain | Critical |
| Kill Switch Hierarchy | 1-5 | Emergency stop capability | Critical |
| Rate Limiting | 1-5 | Respect broker limits | Important |
| Latency Budget | 2-5 | Performance guardrail | Important |
| Observability | 2.5-5 | Monitor system health | Important |
| Feature Flags | 1-5 | Safe rollout | Important |

---

## APPENDIX: RESOURCES

### Books
- **Async Python**: "Using asyncio in Python" (Real Python)
- **Distributed Systems**: "Designing Data-Intensive Applications" (Martin Kleppmann)
- **Trading**: "The Intelligent Trader" + "Algorithmic Trading"
- **Risk Management**: "Value at Risk" (Jorion)
- **Options**: "Options as a Strategic Investment" (McMillan)

### Documentation
- Fyers API: https://api.fyers.in/v3/docs
- Zerodha Kite: https://kite.trade
- Interactive Brokers: https://www.interactivebrokers.com/en/trading/ibapi
- asyncio: https://docs.python.org/3/library/asyncio
- FastAPI: https://fastapi.tiangolo.com
- APScheduler: https://apscheduler.readthedocs.io

### Tools
- Pytest: Unit testing
- Locust: Load testing
- Prometheus: Metrics
- Jaeger: Distributed tracing
- TA-Lib: Technical indicators
- Backtrader / bt: Backtesting (reference)

---

## CRITICAL EXECUTION SAFEGUARDS (Non-Negotiable)

**NO SHORTCUTS ON THESE** — They prevent data loss, financial loss, and platform failures:

1. **Idempotency (Execution Layer)**
   - Every order operation must be idempotent
   - Same key = same result, never duplicated
   - Enforcement: Idempotency service in smarttrade-common

2. **Broker Sync Fundamentals**
   - Broker state is always correct; SmartTrade state is approximation
   - Reconciliation must run hourly minimum
   - External trades must be detected and handled
   - Enforcement: Sync engine with drift classification

3. **Race Condition Prevention**
   - No shared mutable state across requests
   - Actor model per user_id (strict serialization)
   - Database transactions for state updates
   - Enforcement: Async tests, ThreadSanitizer, load tests

4. **Financial Correctness**
   - Decimal only (never float) for all monetary amounts
   - ACID transactions for orders, trades, settlements
   - Immutable audit logs (no data deletion)
   - Enforcement: Code review, test assertions

5. **User Isolation**
   - Every query filters by user_id
   - No cross-user data leakage
   - Per-user actors (not shared state)
   - Enforcement: API middleware, test coverage

6. **Kill Switch Functionality**
   - Must close all positions in <5 seconds
   - Must cancel all pending orders in <10 seconds
   - Must prevent further order placement
   - Enforcement: Hardware panic button OR API endpoint

---

**Version**: 3.2 FINAL
**Last Updated**: 2026-03-26
**Status**: Production-Ready (after patches applied)
**Next Review**: After Phase 1 completion
**Predecessor**: v3.1 (added 11 patches for production reliability)
