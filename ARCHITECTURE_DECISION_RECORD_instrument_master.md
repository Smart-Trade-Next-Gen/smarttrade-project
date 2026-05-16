# Architecture Decision Record: Replicated Instrument Master

**Decision**: Treat instrument metadata as intentionally replicated reference data, distributed via snapshot bootstrap + periodic refresh from MDS.

**Status**: Approved  
**Date**: 2026-04-24  
**Severity**: High (execution path dependency; affects BAS design)

---

## 1. Problem Statement

**Current Challenge**: BAS needs instrument metadata (symbols, exchange, tick size, lot size, contract rules) to validate and execute orders, but maintaining runtime synchronous dependency on MDS violates execution path constraints.

**Requirements**:
- BAS execution path must have **zero runtime MDS lookups** (latency + determinism)
- Instrument metadata **must be available** during order validation
- **Metadata must stay fresh** to catch new instruments and instrument changes
- No synchronous network calls in order execution hot path
- No over-engineered event-driven CDC patterns

**Trade-off**: Current architecture (v4.0) leaves this underspecified — says "BAS should not own" but doesn't define the distribution pattern.

---

## 2. Decision

**Instrument metadata is replicated reference data**, not centralized reference data.

### Key Principles

1. **MDS is authoritative source of truth**
   - MDS owns instrument ingestion and normalization
   - MDS publishes instrument updates via events
   - MDS is source for new instruments, metadata corrections

2. **Each service maintains local replica**
   - BAS, PBS, Strategy, and any service needing instruments maintains local cache
   - Replicas are intentionally identical copies (not eventual consistency)
   - Services do NOT call MDS to resolve instruments at runtime

3. **Replication is snapshot-based**
   - Startup: Full bootstrap load from MDS (`/api/v1/instruments`)
   - Runtime: Periodic refresh job (e.g., every 6 hours)
   - Optional: Event-driven incremental updates (low priority for v1)

4. **Shared library approach**
   - `smarttrade_common.instrument_master` provides unified implementation
   - All services use same InstrumentRegistry, InstrumentCache, SyncService
   - No duplication of symbol logic across services
   - Reduces coupling through shared contracts (schema, APIs)

5. **Fail-fast execution model**
   - If instrument not in local cache → reject order with `instrument_not_found` error
   - No fallback network calls during execution
   - Operational processes must ensure cache is always fresh (alerting on staleness)

---

## 3. What This Changes From v4.0

### Previous Design (v4.0)
- Line 48: "Resolve instrument metadata → MDS (REST or cache)"
- Line 1070-1074: "Instrument Cache is GAP that needs removing; BAS should not own"
- Implied: BAS calls MDS REST API for instrument resolution (during execution or fallback)

### New Design
- Instrument ownership: **BAS has local replicated copy**
- Runtime lookup: **In-memory cache only** (no MDS calls)
- Synchronization: **Snapshot bootstrap + periodic refresh**, not request/response
- Implementation: **Shared library** (smarttrade_common.instrument_master)

### Why This is Better
1. **Execution determinism**: No runtime MDS dependency; guaranteed <1ms lookups
2. **Operational simplicity**: Scheduled jobs handle sync; no CDC event patterns
3. **Code reuse**: All services use same library instead of custom implementations
4. **Explicit ownership**: Clear that instruments are replicated, not centralized
5. **Testability**: Services work in isolation without mocking MDS

---

## 4. Implementation Architecture

### smarttrade_common.instrument_master Package

#### Modules

```
smarttrade_common/
  instrument_master/
    __init__.py
    models.py              # Canonical Instrument schema
    cache.py               # InstrumentCache class
    repository.py          # InstrumentRepository (DB access)
    sync_service.py        # InstrumentSyncService (bootstrap + refresh)
    validators.py          # Validation APIs (tick size, lot size, etc.)
    bootstrap.py           # Startup loader
```

#### Core Abstractions

**InstrumentRegistry** (in-memory, read-only):
```python
class InstrumentRegistry:
    """Provides fast lookups by symbol or instrument_id."""
    
    def get(self, symbol: str) -> Instrument | None:
        """O(1) lookup by symbol."""
    
    def search(self, exchange: str, instrument_type: str) -> List[Instrument]:
        """Filtered search."""
    
    def validate_order(self, symbol: str, quantity: int, price: float) -> ValidationResult:
        """Check tick_size, lot_size, price range, etc."""
    
    def snapshot(self) -> Dict[str, Instrument]:
        """Immutable snapshot for serialization."""
```

**InstrumentCache** (persistent cache with versioning):
```python
class InstrumentCache:
    """Persistent cache for instrument master; survives restarts."""
    
    async def load() -> InstrumentRegistry:
        """Load from local DB or Redis."""
    
    async def upsert(instruments: List[Instrument]) -> None:
        """Merge new/updated instruments; idempotent."""
    
    @property
    def version(self) -> int:
        """Cache version for staleness detection."""
    
    @property
    def checksum(self) -> str:
        """SHA256 of instrument list for integrity checking."""
    
    async def clear(self) -> None:
        """Full reset (dangerous; used for recovery)."""
```

**InstrumentSyncService** (orchestrates bootstrap + refresh):
```python
class InstrumentSyncService:
    """Manages instrument master synchronization from MDS."""
    
    async def bootstrap(mds_client: BaseServiceClient) -> InstrumentRegistry:
        """One-time startup load from MDS /api/v1/instruments."""
    
    async def refresh(mds_client: BaseServiceClient, force: bool = False) -> bool:
        """Periodic refresh job; idempotent; returns True if updated."""
    
    async def subscribe_to_events(redis: Redis) -> None:
        """Optional: listen to market_data.instrument_updated.v1 events."""
    
    @property
    def last_sync_time(self) -> datetime:
        """Track staleness."""
    
    @property
    def sync_lag(self) -> timedelta:
        """Current lag from MDS source of truth."""
```

#### Canonical Schema

```python
@dataclass
class Instrument:
    """Canonical instrument master record."""
    
    # Identity
    instrument_id: str  # UUID
    symbol: str         # e.g., "NSE:INFY"
    exchange: str       # "NSE", "BSE", "MCX"
    isin: str | None    # ISIN code
    
    # Contract details
    instrument_type: str  # "EQUITY", "FUTURES", "OPTIONS", "ETF"
    expiry_date: date | None  # For derivatives
    strike_price: float | None  # For options
    option_type: str | None  # "CALL" or "PUT"
    
    # Validation rules
    tick_size: float      # Minimum price increment
    lot_size: int         # Standard lot size
    
    # Metadata
    sector: str | None
    industry: str | None
    status: str           # "ACTIVE", "SUSPENDED", "DELISTED"
    
    # Versioning
    version: int          # Increment on change
    checksum: str         # SHA256 for integrity
    updated_at: datetime
    created_at: datetime
```

#### Shared Validation APIs

```python
class InstrumentValidator:
    """Validation helper used by all services."""
    
    @staticmethod
    def validate_tick(symbol: str, price: float, registry: InstrumentRegistry) -> bool:
        """Check if price respects tick_size rules."""
    
    @staticmethod
    def validate_lot(symbol: str, quantity: int, registry: InstrumentRegistry) -> bool:
        """Check if quantity is multiple of lot_size."""
    
    @staticmethod
    def validate_trading_hours(
        symbol: str, 
        timestamp: datetime, 
        registry: InstrumentRegistry
    ) -> bool:
        """Check if symbol is tradeable at this time."""
    
    @staticmethod
    def batch_validate_order(
        order: Order, 
        registry: InstrumentRegistry
    ) -> ValidationResult:
        """Comprehensive order validation."""
```

---

## 5. Replication Model

### Startup Sequence

```
1. Service starts (BAS, PBS, Strategy, etc.)
2. InstrumentCache.load() from local DB/Redis
3. If cache empty or stale (>6h old):
   - InstrumentSyncService.bootstrap(mds_client)
   - Fetch full list from MDS /api/v1/instruments
   - Persist to local DB with version/checksum
   - Load into InstrumentRegistry (in-memory)
4. Service ready; accept orders using registry

Fallback:
- If MDS unreachable during bootstrap:
  - Retry with exponential backoff (3x)
  - If still fails: Start with empty registry
  - Reject orders with "instrument_master_unavailable"
  - Alert ops to investigate
```

### Runtime Refresh

```
Scheduled Job (every 6 hours):
1. InstrumentSyncService.refresh(mds_client)
2. Fetch full list from MDS
3. Compare with local version/checksum
4. If changed:
   - Merge into InstrumentCache (idempotent upsert)
   - Update in-memory InstrumentRegistry
   - Log change details
5. If unchanged: No-op (checksum match)

Optional Event-Driven (low priority):
- Subscribe to market_data.instrument_updated.v1 events
- Update individual instruments (incremental)
- Verify consistency with next full refresh
- Add only AFTER v1 is stable
```

### Staleness Detection & Recovery

```
Monitoring:
- Track InstrumentSyncService.last_sync_time in each service
- Alert if lag > 12 hours (conservative window)
- Expose /healthz endpoint: synced? checksum matches?

Recovery (manual):
- Admin API: POST /api/v1/instruments/reset
- Force immediate InstrumentSyncService.bootstrap()
- Restart all consumers of instrument cache

Auto-Recovery (future):
- If order placement fails due to instrument_not_found
- Trigger immediate refresh in background
- Retry order placement (if idempotent)
```

---

## 6. Data Flow Diagram

```
┌──────────────────┐
│  MDS             │
│  • Fetch real    │
│    instrument    │
│    master from   │
│    Fyers/PBS     │
│  • Normalize     │
│  • Publish via   │
│    /api/v1/inst  │
└────────┬─────────┘
         │
         │ Every 6h
         │ Full snapshot
         │ fetch
         │
         ▼
┌──────────────────────────┐
│ InstrumentSyncService    │
│ (all services)           │
│                          │
│ bootstrap() + refresh()  │
│ Idempotent merge logic   │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ InstrumentCache          │
│ (persistent store)       │
│                          │
│ • Local DB / Redis       │
│ • Version tracking       │
│ • Checksum validation    │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ InstrumentRegistry       │
│ (in-memory, per service) │
│                          │
│ • BAS InstrumentRegistry │
│ • PBS InstrumentRegistry │
│ • Strategy InstrumentReg │
│                          │
│ <1ms lookups             │
└──────────────────────────┘
         │
    ┌────┴────┬─────────┬─────────┐
    ▼         ▼         ▼         ▼
  Order    Quote    Risk      Symbol
  Validation Valid  Limits    Mapping
```

---

## 7. Service-Specific Integration

### BAS (Broker Adapter Service)

```python
class BASOrderHandler:
    def __init__(self, instrument_registry: InstrumentRegistry):
        self.registry = instrument_registry
        self.sync_service = InstrumentSyncService(cache=InstrumentCache())
        # Startup
        asyncio.create_task(self.sync_service.refresh())  # Background refresh job
    
    async def place_order(self, order: Order) -> OrderResponse:
        # Lookup from in-memory registry
        instrument = self.registry.get(order.symbol)
        
        if not instrument:
            raise OrderValidationError("instrument_not_found")
        
        # Validate tick, lot, price range
        validator = InstrumentValidator()
        if not validator.validate_order(order, instrument):
            raise OrderValidationError("invalid_order_params")
        
        # Continue with risk validation, broker execution...
        # No MDS calls; all data in-memory
```

### PBS (Paper Broker Service)

```python
class PBSOrderSimulator:
    def __init__(self, instrument_registry: InstrumentRegistry):
        self.registry = instrument_registry
        self.sync_service = InstrumentSyncService(cache=InstrumentCache())
    
    async def simulate_fill(self, order: Order) -> Fill:
        instrument = self.registry.get(order.symbol)
        
        if not instrument:
            raise ExecutionError("instrument_not_found")
        
        # Use instrument metadata to simulate realistic fills
        # (e.g., respect tick_size, lot_size)
        fill = self.apply_tick_size(order, instrument.tick_size)
        return fill
```

### Strategy Service

```python
class StrategyEvaluator:
    def __init__(self, instrument_registry: InstrumentRegistry):
        self.registry = instrument_registry
        self.sync_service = InstrumentSyncService(cache=InstrumentCache())
    
    async def evaluate_signal(self, symbol: str, quote: Quote) -> Decision:
        instrument = self.registry.get(symbol)
        
        if not instrument:
            # Skip signal evaluation for unknown instrument
            return Decision.HOLD
        
        # Use instrument metadata for signal validation
        # (e.g., only trade active instruments, respect lot_size)
```

---

## 8. Conflict Resolution: v4.0 → v4.1

### What Changes

**Line 48 (ExecutionContext)**:
```python
# OLD (v4.0):
instrument_snapshot: Dict[symbol, Instrument]  # from local InstrumentCache

# NEW (v4.1):
# Same; clarify that this is from replicated local cache
instrument_snapshot: Dict[symbol, Instrument]  # from local InstrumentRegistry
                                               # (replicated from MDS)
```

**Line 1070-1074 (Gap 2: Instrument Cache)**:
```
# OLD (v4.0):
Gap 2: Instrument Cache — Duplicates MDS responsibility
Target: BAS should not own instrument metadata
        All instrument lookups → MDS (REST or cache)

# NEW (v4.1):
Gap 2 RESOLVED: Instrument Metadata as Replicated Reference Data
Target: BAS owns replicated copy (like all services)
        Synced from MDS via snapshot bootstrap + 6h refresh
        Zero runtime MDS calls during execution
```

**Line 476-490 (Rule 2: BAS Data Provisioning)**:
```
# OLD (v4.0):
Instruments: In-memory cache (preloaded at startup, refreshed every 6h)

# NEW (v4.1):
Instruments: Replicated reference data
  - Replicated from MDS authoritative source
  - Populated via InstrumentSyncService (snapshot bootstrap + 6h refresh)
  - All services maintain identical copy (intentional duplication)
  - Lookup: InstrumentRegistry.get(symbol) → O(1) in-memory
  - Failure: reject order with instrument_not_found (no fallback to MDS)
```

### What Stays the Same

- BAS execution path remains synchronous, <100ms
- No runtime MDS calls in hot path
- Pre-cached data model is unchanged
- Fail-fast rejection on cache miss

---

## 9. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Replicas go stale** | Orders fail due to missing/outdated instruments | 1. Monitor sync lag (alert >12h) 2. Auto-refresh every 6h 3. Alert ops if MDS unavailable |
| **New instruments lag** | Can't trade new NSE/BSE instruments until sync | Accept 6h lag; ops can trigger manual refresh |
| **Metadata contradiction** | Different services have different versions | Use checksums; sync all services on same schedule; log divergence |
| **MDS unavailable at startup** | Services can't bootstrap | Retry with backoff; start with empty registry; reject orders; alert ops |
| **Sync failures accumulate** | Cache becomes progressively stale | Circuit breaker on sync failures; alert after 3 consecutive failures |

---

## 10. Testing Strategy

### Unit Tests
- InstrumentValidator: Tick, lot, price validation
- InstrumentRegistry: Lookups, search, snapshot
- InstrumentCache: Upsert, versioning, checksum

### Integration Tests
- InstrumentSyncService: Bootstrap from mock MDS
- Order placement: Hit order, verify tick_size validation
- Strategy: Signal evaluation with/without instrument

### End-to-End Tests
- Service startup: Full bootstrap flow
- Periodic refresh: Verify cache updated
- Cache miss: Order rejected with instrument_not_found
- Metadata change: Sync new version; existing trades unaffected

### Monitoring
- Gauge: `instrument_master_sync_lag_seconds`
- Gauge: `instrument_cache_size`
- Gauge: `instrument_cache_version`
- Counter: `instrument_lookups_total` (for perf tracking)
- Counter: `instrument_not_found_errors_total` (operational visibility)

---

## 11. Rationale vs. Alternatives

### Alternative 1: Centralized Reference Data (MDS Only)
- ❌ Violates execution path constraint (runtime MDS calls)
- ❌ Adds unpredictable latency
- ❌ Couples BAS to MDS availability

### Alternative 2: Event-Driven CDC (Change Data Capture)
- ❌ Over-engineered for simple snapshot replication
- ❌ Requires event streaming for every metadata change
- ❌ Adds operational complexity (consumer lag, idempotency)
- Use ONLY if instrument master changes frequently (currently not true)

### Alternative 3: Shared DB with Read Replicas
- ❌ Adds database dependency
- ❌ Increases infrastructure complexity
- ❌ Still requires async sync to stay fresh
- Replicated reference data is simpler and more predictable

### Chosen: Snapshot Replication + Shared Library
- ✅ Execution path independence (no runtime MDS calls)
- ✅ Operational simplicity (scheduled jobs, no CDC)
- ✅ Code reuse (single implementation in smarttrade_common)
- ✅ Tested pattern (used by all distributed systems)

---

## 12. Future Enhancements

### Phase 1 (v4.1)
- Snapshot bootstrap + 6h refresh
- Shared library implementation
- Fail-fast rejection model

### Phase 2 (Q3)
- Event-driven incremental updates (`market_data.instrument_updated.v1`)
- Conflict resolution if event-driven diverges from snapshot
- Monitor for "stale replica" anomalies

### Phase 3 (Q4)
- Support multiple instrument masters (non-NSE exchanges)
- Fallback to alternative exchanges if primary is down
- Instrument alias mapping (ticker → NSE symbol)

---

## 13. Sign-Off

**Architecture Review**: ✅ Aligned with v4.0 execution plane constraints  
**Operations**: ✅ Simple to monitor; scheduled jobs only  
**Implementation**: ✅ Shared library pattern verified in smarttrade_common  
**Testing**: ✅ Unit, integration, e2e tests planned  

**Next Step**: Implement smarttrade_common.instrument_master (Phase 1, 3-4 weeks)
