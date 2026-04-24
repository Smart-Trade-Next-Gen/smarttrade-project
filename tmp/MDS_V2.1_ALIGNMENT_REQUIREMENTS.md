# MDS v2.1 Production Hardening — Architecture & Roadmap Alignment

**Date**: 2026-04-18  
**Status**: CRITICAL ALIGNMENT ITEMS IDENTIFIED  
**Action**: Update architecture v3.4, ROADMAP, and Phase 0-4 implementation plans

---

## Executive Summary

The finalized MDS v2.1 (Production Hardening) introduces **7 critical production fixes** that require:
1. **Architecture document updates** (v3.4 needs MDS service responsibilities refresh)
2. **Roadmap timeline changes** (MDS is NOT complete; Phase 0-4 implementation needed)
3. **Infrastructure capability additions** (PostgreSQL partitioning, monitoring, scheduler resources)
4. **Event contract definitions** (deterministic idempotency keys, candle schema versioning)
5. **Testing strategy expansion** (determinism tests, memory leak tests, distributed system tests)

---

## 1. Architecture Document Updates (smarttrade-architecture-v3.4-current.md)

### Current State
MDS is listed as **"✅ Complete"** with just:
- Real-time quotes
- Instrument resolution
- Trading calendar
- WebSocket feeds

### Required Changes

#### 1.1 Update MDS Service Responsibilities Matrix (Section 2)

**BEFORE:**
```markdown
| **MDS** | Real-time quotes, instrument resolution, trading calendar | broker.session.v1 | market_data.quote.v1 |
```

**AFTER:**
```markdown
| **MDS** | Real-time quotes, instrument resolution, trading calendar, **1m/5m/15m/1h/1d OHLC aggregation, IV calculation, backtest data feed** | broker.session.v1 | market_data.quote.v1, **market_data.candle.finalized.v1, market_data.candle_gap.v1, market_data.iv.calculated.v1** |
```

#### 1.2 Add New Section: "MDS Architecture Decisions (v3.4 Update)"

```markdown
## 2.6 Market Data Service — Production-Grade Architecture

### Core Principles (v2.1 Finalization)

1. **Determinism**: Same input always produces identical output (critical for reproducible backtests)
2. **Idempotency**: All operations safe to retry; deduplication via SHA256 keys + DB unique constraints
3. **Memory Safety**: No unbounded growth; TTL cleanup for finalized bucket tracking (10-minute retention)
4. **Distributed Safety**: Stable idempotency keys across processes; time-driven finalization independent of tick arrival
5. **Graceful Degradation**: Circuit breaker on broker outages; configurable late tick policies (discard/log_only)

### MDS Responsibilities (Strict Boundaries)

**OWNS:**
- ✅ Real-time quote ingestion & WebSocket fan-out
- ✅ Instrument resolution & broker mapping
- ✅ Trading calendar management (market hours, holidays)
- ✅ 1m candle aggregation from ticks (bucket-scoped buffering)
- ✅ Multi-interval derivation (5m/15m/1h/1d from 1m candles)
- ✅ IV calculation (Black-Scholes, config-driven, no hardcoded params)
- ✅ Volatility surface caching
- ✅ Backtest data feed (historical OHLC with gap detection)
- ✅ Candle versioning & source tracking (live, derived, broker_daily, broker_1m_backfill)

**DOES NOT OWN:**
- ❌ Trading events (orders, positions, trades) → BAS responsibility
- ❌ Signal generation → Strategy Service responsibility
- ❌ Risk calculation → BAS Risk Engine responsibility
- ❌ User authentication → Auth Service responsibility

### Critical Implementation Details (Phase 0)

| Item | v3.4 Status | v2.1 Requirement | Impact |
|------|-------------|------------------|--------|
| **Bucket-scoped tick buffering** | Not mentioned | Time-driven finalization every 60s (independent of tick arrival) | REQUIRED for production stability |
| **Idempotency keys** | Not mentioned | SHA256-based (symbol:timestamp:price:volume), stable across processes | REQUIRED for distributed deployments |
| **Deduplication strategy** | Not mentioned | PostgreSQL UNIQUE constraints (DB is authoritative) | REQUIRED for idempotency |
| **Memory management** | Not mentioned | TTL cleanup for finalized buckets (10-min retention), metrics monitoring | REQUIRED to prevent memory leaks |
| **Late tick handling** | Not mentioned | Configurable policy (discard/log_only), no real-time correction | REQUIRED for determinism |
| **Derived candle idempotency** | Not mentioned | All 5m/15m/1h/1d have idempotency keys | REQUIRED for production safety |
| **Circuit breaker** | Not mentioned | Broker outage protection, rate limiting (10K ticks/sec) | REQUIRED for resilience |

### Event Contracts (NEW)

MDS publishes these events with deterministic structure:

```yaml
# market_data.candle.finalized.v1
- symbol: "SBIN-EQ"
- interval: "1m" | "5m" | "15m" | "1h" | "1d"
- timestamp: ISO8601 (UTC)
- open, high, low, close: Decimal (not float)
- volume: int
- source: "live" | "derived" | "broker_daily"
- is_complete: bool
- confidence: "high" | "medium" | "low"
- quality_flags: ["outlier_price"] | []
- version: int (schema version)
- idempotency_key: SHA256 hash (stable across processes)
- created_at, updated_at, last_corrected_at: ISO8601

# market_data.candle_gap.v1
- symbol, interval, start_time, end_time
- reason: "no_ticks" | "late_ticks" | "broker_outage" | "incomplete_trading_day"
- severity: "low" | "medium" | "high"
- context: string (contextual info)

# market_data.iv.calculated.v1
- symbol, strike, option_type, expiry
- iv: Decimal (0.01 to 5.00)
- source: "calculated_from_market_price"
- timestamp, calculation_config_version
- is_valid: bool
- quality_flags: []
```

### Consumption Model (NOT Production Push)

BAS and Strategy Service **consume** MDS events via Redis Streams:
- ✅ Durable (persisted, retryable)
- ✅ Ordered (per symbol)
- ✅ Idempotent (consumer group offset management)
- ❌ NOT real-time push (use WebSocket for real-time)
```

#### 1.3 Update "Architectural Debt & Phase 1 Cleanup Items" (Section 1.5)

Add under existing cleanup items:

```markdown
| **MDS Phase 0-4 Foundation Gap** | Multiple files | Blocks all MDS functionality (no Phase 0 schema, no Phase 1 historical data feed) | 400 hrs | **BLOCKER** |
```

---

## 2. ROADMAP Updates (smarttrade-project/ROADMAP.md)

### Current Issues

**Line 11**: MDS marked as "Production Ready ✅" — **INCORRECT**
- v2.1 identifies Phase 0-4 implementation needed
- Only real-time quotes are production-ready; backtest data feed missing
- No historical OHLC aggregation in production

**Section Q2 2026**: Frontend integration and production hardening don't mention MDS schema/infrastructure
- PostgreSQL schema creation for candles not listed
- Prometheus metrics setup (buffer counts, TTL cleanup) not mentioned
- Scheduler resources (time-driven finalization every 60s) not budgeted

### Required Changes

#### 2.1 Update MDS Current Status (Line 9-20)

**REPLACE:**
```markdown
| Market Data Service | Production Ready ✅ | Unit + integration tests |
```

**WITH:**
```markdown
| Market Data Service (Real-time) | Production Ready ✅ | Real-time quotes, instruments, WebSocket |
| Market Data Service (Backtest) | Phase 0-4 In Progress 🔄 | PostgreSQL schema, OHLC aggregation, determinism, idempotency |
```

#### 2.2 Update Q2 2026 Schedule (After "Production Hardening")

**ADD NEW SUBSECTION:**

```markdown
### MDS Phase 0-4 Implementation (Parallel track: Apr–Jun 2026)

**Phase 0: Foundation (2 weeks)**
- [ ] PostgreSQL schema (partitioned candles table, idempotency keys, TTL indexes)
- [ ] Trading calendar seeding (market hours, holidays, expected_candles per symbol)
- [ ] Bucket-scoped tick buffer implementation (per-(symbol, bucket_start) isolation)
- [ ] Time-driven finalization scheduler (every 60 seconds)
- [ ] Memory cleanup service (TTL for finalized buckets, stale symbol pruning)
- [ ] 20+ determinism tests (same input → identical output, verified 10 times)

**Phase 1: Historical Data Feed (2 weeks)**
- [ ] Broker daily OHLC backfill (Fyers, Paper Broker)
- [ ] Gap detection & logging (missing candles, broker outages)
- [ ] Data validation & quality metrics
- [ ] 15+ integration tests (backfill, gap detection)

**Phase 2: Advanced Aggregation (1.5 weeks)**
- [ ] Multi-interval derivation (5m, 15m, 1h, 1d from 1m)
- [ ] Idempotent derived candle inserts (deterministic keys)
- [ ] Exact boundary conditions (watermark-based finalization)
- [ ] 15+ tests (interval boundaries, completeness checks)

**Phase 3: IV & Greeks (1.5 weeks)**
- [ ] Config-driven IV calculation (no hardcoded parameters)
- [ ] Real-time IV surface updates (on quote refresh)
- [ ] Multi-leg Greeks support (option spreads)
- [ ] Volatility skew/smile detection
- [ ] 20+ tests (Black-Scholes accuracy, edge cases)

**Phase 4: Backtest Data Feed (1 week)**
- [ ] Backtest API: `/api/v1/data/ohlc?symbol=SBIN-EQ&interval=5m&from=2025-01-01&to=2025-03-31`
- [ ] Replay cursor abstraction (seek, peek, progress)
- [ ] Corporate action application (dividends, splits, bonus)
- [ ] Gap rejection for incomplete data sets
- [ ] 10+ tests (seek, progress, corporate actions)

**Total Effort**: ~400 hours (8 weeks @ 50 hrs/week)  
**Blocking Dependencies**: None (parallel to Frontend Phase 5)  
**Go-Live Risk**: Must complete before production trading (Q3 launch)
```

#### 2.3 Update Q3 2026 (Zerodha Broker Integration)

**ADD NOTE:**
```markdown
**NOTE**: Zerodha integration will follow same MDS v2.1 patterns as Fyers:
- Broker-specific tick ingestion plugin
- Bucket-scoped buffer per (symbol, minute_boundary)
- Same idempotency key generation (SHA256)
- Same late tick handling policy (discard/log_only)
- Reuse Phase 0 determinism test suite
```

#### 2.4 Update Infrastructure Section (May-Jun Production Hardening)

**ADD MDS-specific items:**
```markdown
**MDS Infrastructure**
- [ ] PostgreSQL partitioning strategy (candles table by month)
- [ ] Prometheus metrics: mds_tick_buffers_count, mds_finalized_buckets_count, mds_active_symbols_count
- [ ] Redis Streams consumer group setup (for backtest data subscription)
- [ ] Scheduler CPU allocation (time-driven finalization loop)
- [ ] Load testing: 10K ticks/sec with circuit breaker active
- [ ] Memory profiling: verify TTL cleanup prevents unbounded growth
```

---

## 3. Infrastructure & Operations Impact

### New PostgreSQL Schema Requirements

**MISSING from current infra**: MDS historical_candles table (partitioned)

```sql
-- NEW: Required in Phase 0
CREATE TABLE historical_candles (
    id BIGSERIAL,
    symbol VARCHAR(20) NOT NULL,
    interval CANDLE_INTERVAL NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    open NUMERIC(20, 8), high NUMERIC(20, 8), low NUMERIC(20, 8), close NUMERIC(20, 8),
    volume BIGINT,
    
    -- v2.1 additions
    version INT DEFAULT 1,
    source VARCHAR(50) NOT NULL,
    is_complete BOOLEAN DEFAULT true,
    confidence VARCHAR(20) DEFAULT 'high',
    quality_flags TEXT[],
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    last_corrected_at TIMESTAMPTZ,
    idempotency_key VARCHAR(64) NOT NULL UNIQUE,
    
    PRIMARY KEY (id)
) PARTITION BY RANGE (timestamp);

-- Partitioning: monthly (2025-01, 2025-02, ...)
CREATE TABLE historical_candles_2025_01 PARTITION OF historical_candles
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

### New Monitoring Requirements

**MISSING from current Prometheus setup**: MDS memory metrics

```python
# Required in Phase 0
mds_tick_buffers_count = Gauge("mds_tick_buffers_count", "Active tick buffers")
mds_finalized_buckets_count = Gauge("mds_finalized_buckets_count", "Tracked finalized buckets (TTL 10min)")
mds_active_symbols_count = Gauge("mds_active_symbols_count", "Symbols with recent activity (last 6h)")
mds_late_ticks_total = Counter("mds_late_ticks_total", "Late ticks received", labels=["policy_applied"])
mds_bucket_finalization_duration_seconds = Histogram("mds_bucket_finalization_duration_seconds", "Time to finalize 1m bucket")
mds_circuit_breaker_state = Gauge("mds_circuit_breaker_state", "CB state: 0=CLOSED, 1=OPEN, 2=HALF_OPEN")
```

### Scheduler Resources

**NEW**: Time-driven bucket finalization loop (every 60 seconds)

```python
# Runs independently in MDS lifespan
class BucketFinalizationScheduler:
    async def finalize_overdue_buckets(self):
        # Runs every 60s, processes ~100 symbols typical
        # CPU: minimal (few ms per cycle)
        # Memory: fixed cost (no growth)
```

---

## 4. Event Contract Definitions (NEW)

### Missing from Current Architecture

**BAS consumes from MDS via Redis Streams:**
- Need explicit event contract for `market_data.candle.finalized.v1`
- Need to handle idempotency keys (SHA256) in BAS event consumers
- Need to validate candle schema version in BAS

**Current issue**: BAS maintains local quote_store and instrument_cache (v3.4, line 64-65)
- Should consume from MDS events instead
- v2.1 provides deterministic event stream with idempotency keys

**ACTION**: Update smarttrade-common event schemas to include:
- `idempotency_key: str` (SHA256 hash)
- `version: int` (schema version)
- `source: str` (live, derived, broker_daily)
- `confidence: str` (high, medium, low)
- `quality_flags: List[str]` (for data quality tracking)

---

## 5. Testing Strategy Expansion (NEW)

### Determinism Tests (Prerequisite)

**MISSING from current test suite**: Run same input 10 times, verify identical output

```python
# Required in Phase 0
async def test_candle_generation_is_deterministic():
    """Same ticks in different order → identical candle"""
    ticks = [tick_A, tick_B, tick_C]
    
    for iteration in range(10):
        # Shuffle ticks
        shuffled = random.sample(ticks, len(ticks))
        
        # Generate candle
        candle = await buffer.finalize_bucket_from_ticks(shuffled)
        
        # Verify identical to first iteration
        assert candle == first_iteration_candle
        assert candle.idempotency_key == first_iteration_key
```

### Memory Leak Tests (NEW)

**MISSING**: Verify TTL cleanup prevents unbounded memory growth

```python
# Required in Phase 0
async def test_finalized_bucket_tracking_ttl():
    """Finalized bucket entries expire after 10 minutes"""
    buffer = BucketScopedTickBuffer()
    
    # Finalize 1000 buckets
    for i in range(1000):
        await buffer._finalize_bucket(f"SYMBOL_{i}", bucket)
    
    # Verify tracking dict has 1000 entries
    assert len(buffer.finalized_buckets) == 1000
    
    # Wait 11 minutes
    await asyncio.sleep(11 * 60)
    
    # Run cleanup
    await memory_mgmt.cleanup_expired_finalized_buckets()
    
    # Verify all expired entries removed
    assert len(buffer.finalized_buckets) == 0
```

### Distributed System Tests (NEW)

**MISSING**: Idempotency keys stable across processes

```python
# Required in Phase 0
def test_idempotency_key_stability():
    """SHA256 keys are identical across processes"""
    # Process 1
    key_1 = generate_idempotency_key(
        symbol="SBIN-EQ",
        timestamp=datetime(2026, 4, 18, 9, 15, 0),
        price=Decimal("540.50"),
        volume=100
    )
    
    # Process 2 (simulate different process)
    key_2 = generate_idempotency_key(
        symbol="SBIN-EQ",
        timestamp=datetime(2026, 4, 18, 9, 15, 0),
        price=Decimal("540.50"),
        volume=100
    )
    
    # Keys must be identical
    assert key_1 == key_2
    assert key_1.startswith("a1b2c3")  # Deterministic prefix
```

---

## 6. Phase 0-4 Task Breakdown (For smarttrade-project/implementation-plan)

### Phase 0: Foundation (2 weeks, 100 hrs)

```
TASK_MDS_0_1: PostgreSQL schema creation (partitioned candles table)
TASK_MDS_0_2: Trading calendar seeding & market hours validation
TASK_MDS_0_3: Bucket-scoped tick buffer implementation
TASK_MDS_0_4: Time-driven finalization scheduler
TASK_MDS_0_5: Memory management & cleanup service
TASK_MDS_0_6: Determinism test suite (20+ tests)
TASK_MDS_0_7: Memory leak tests (10+ tests)
```

### Phase 1: Historical Data (2 weeks, 80 hrs)

```
TASK_MDS_1_1: Broker daily OHLC backfill (Fyers)
TASK_MDS_1_2: Broker daily OHLC backfill (Paper Broker)
TASK_MDS_1_3: Gap detection & logging
TASK_MDS_1_4: Data quality metrics & validation
TASK_MDS_1_5: Integration tests (backfill scenarios)
```

### Phase 2: Multi-Interval Derivation (1.5 weeks, 60 hrs)

```
TASK_MDS_2_1: 5m candle derivation from 1m
TASK_MDS_2_2: 15m/1h/1d derivation
TASK_MDS_2_3: Exact boundary conditions (watermark-based)
TASK_MDS_2_4: Idempotent derived candle inserts
TASK_MDS_2_5: Interval boundary tests (15+ tests)
```

### Phase 3: IV & Greeks (1.5 weeks, 60 hrs)

```
TASK_MDS_3_1: Config-driven IV calculation
TASK_MDS_3_2: Real-time IV surface updates
TASK_MDS_3_3: Multi-leg Greeks support
TASK_MDS_3_4: Volatility skew detection
TASK_MDS_3_5: Greeks accuracy tests (20+ tests)
```

### Phase 4: Backtest Data Feed (1 week, 40 hrs)

```
TASK_MDS_4_1: Backtest API (/api/v1/data/ohlc)
TASK_MDS_4_2: Replay cursor implementation
TASK_MDS_4_3: Corporate action application
TASK_MDS_4_4: Gap rejection for incomplete data
TASK_MDS_4_5: Replay tests (10+ tests)
```

---

## 7. Summary: Alignment Actions Required

| Item | Current State | Required Change | Priority | Effort |
|------|---------------|-----------------|----------|--------|
| **Architecture v3.4 MDS description** | "Complete" | Add Phase 0-4 details, event contracts, determinism guarantees | HIGH | 4 hrs |
| **ROADMAP MDS status** | "Production Ready" | Change to "Phase 0-4 In Progress", add 8-week schedule | HIGH | 6 hrs |
| **PostgreSQL schema** | Missing historical_candles | Add partitioned table with v2.1 fields | BLOCKER | 4 hrs |
| **Monitoring metrics** | No MDS memory metrics | Add Prometheus metrics for buffers/buckets/symbols | MEDIUM | 2 hrs |
| **Event contracts** | Basic only | Add idempotency_key, version, source, quality_flags | HIGH | 3 hrs |
| **Test suite** | Missing determinism tests | Add 50+ determinism, memory, distributed system tests | HIGH | 20 hrs |
| **Phase 0-4 task breakdown** | Not documented | Create detailed task list with pseudocode | HIGH | 8 hrs |

**TOTAL ALIGNMENT EFFORT**: ~47 hours (1 week dedicated work)  
**TIMELINE**: Complete before Phase 0 implementation starts  
**GO-LIVE DEPENDENCY**: CRITICAL (must align before production trading Q3 2026)

---

## 8. Next Steps

1. **This Week (2026-04-18 to 2026-04-25)**:
   - [ ] Update smarttrade-architecture-v3.4-current.md (Section 2.6)
   - [ ] Update ROADMAP.md (MDS status + Q2 schedule)
   - [ ] Create Phase 0-4 task breakdown in smarttrade-project/IMPLEMENTATION_PLAN

2. **Week After (2026-04-25 to 2026-05-02)**:
   - [ ] Create PostgreSQL schema migration (Phase 0, Task 1)
   - [ ] Implement bucket-scoped buffer (Phase 0, Task 3)
   - [ ] Create determinism test suite (Phase 0, Task 6)

3. **Go-Live Readiness**:
   - [ ] Phase 0-4 complete by 2026-06-30 (before Q3 Zerodha integration)
   - [ ] 100+ MDS tests passing (determinism, memory, distributed system)
   - [ ] Prometheus metrics validated (no memory leaks in 24h continuous run)
   - [ ] Production schema deployed (partitioning, indexes, TTL cleanup)

