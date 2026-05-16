# Implementation Summary: Replicated Instrument Master Architecture

**Date**: 2026-04-24  
**Status**: Design Complete; Ready for Implementation  
**Owner**: Architecture & Platform Team  
**Deliverables**: 3 comprehensive documents + architecture updates

---

## What Was Delivered

### 1. Architecture Decision Record (ADR)
**File**: `ARCHITECTURE_DECISION_RECORD_instrument_master.md`

**Purpose**: Formal decision documentation + business justification

**Contents**:
- Problem statement (BAS needs instruments without runtime MDS calls)
- Decision: Treat instruments as replicated reference data (not centralized, not event-driven CDC)
- Implementation architecture with 6 reusable modules
- Canonical Instrument schema (with validation logic)
- Replication model (snapshot bootstrap + 6h periodic refresh)
- Service-specific integration examples (BAS, PBS, Strategy)
- Risk analysis (5 risks with mitigations)
- Rationale vs. alternatives considered
- Testing strategy (unit, integration, e2e)
- Deployment & rollout plan
- **Scope**: 15 sections, ~700 lines, complete design

**Key Insight**: This is NOT a temporary hack or shortcut. It's an intentional architecture decision treating instrument metadata the same way Netflix treats configuration data or Google treats service definitions — replicated, versioned, resilient.

---

### 2. High-Level & Low-Level Design (HLD/LLD)
**File**: `DESIGN_smarttrade_common_instrument_master.md`

**Purpose**: Implementation-ready design specification for engineers

**Contents**:
- Executive summary (2-page overview)
- Complete data model (`Instrument` dataclass with 20+ fields)
  - Contract details (tick_size, lot_size, multiplier, etc.)
  - Validation rules (price bounds, trading status)
  - Versioning & integrity (checksums for replica detection)
  
- **InstrumentRegistry** (in-memory fast lookup)
  - O(1) symbol lookup
  - Filtered search by exchange/type/status
  - Order validation (tick, lot, price bounds)
  - Thread-safe snapshots
  
- **InstrumentCache** (persistent storage)
  - Pluggable backends (DB, Redis, file-based)
  - Versioning + checksum tracking
  - Idempotent upsert logic (merge strategy)
  - Staleness detection
  
- **InstrumentSyncService** (replication orchestration)
  - Bootstrap from MDS (`GET /api/v1/instruments`)
  - Periodic refresh (configurable interval)
  - Retry logic with exponential backoff
  - Failure tracking + health monitoring
  - Optional event-driven updates (Phase 2)
  
- **InstrumentValidator** (shared validation APIs)
  - Symbol existence + tradeable status
  - Order validation (comprehensive)
  - Trading hours validation
  
- Service integration examples
  - BAS OrderHandler (pre-order validation)
  - PBS OrderSimulator (fill validation)
  - Strategy Service (signal filtering)
  
- Testing strategy
  - Unit tests (models, registry, cache, sync)
  - Integration tests (bootstrap, refresh, idempotency)
  - E2E tests (cache miss handling, startup recovery)
  - Monitoring (metrics, alerting, health checks)
  
- Deployment & rollout (Phase 1-3)
  - Week 1-2: Library implementation (12-16h)
  - Week 3-4: BAS integration (4-6h)
  - Week 5-6: PBS integration (2-4h)
  - Week 7+: Prod rollout with canary (10% → 50% → 100%)

**Scope**: 30 sections, complete Python code examples, 2000+ lines

---

### 3. Architecture Document Patches
**File**: `FINAL_TARGET_ARCHITECTURE_v4.0.md` (updated)

**Changes Made**:
1. **BAS Service Definition** (line ~18)
   - Added: "Local replicated instrument master" to MUST OWN
   - Added: "Zero runtime MDS calls during execution" to MUST NOT DO
   
2. **MDS Service Definition** (line ~66)
   - Clarified: MDS is authoritative source of truth (ingestion, normalization)
   - Added: Replication responsibility (snapshot + events)
   - Noted: Intentional duplication (not eventual consistency)
   
3. **ExecutionContext Model** (line ~46)
   - Clarified: Instrument snapshot from local InstrumentRegistry (replicated from MDS)
   - Added: Replication sources (async background jobs)
   
4. **Rule 2: BAS Data Provisioning** (line ~474)
   - Expanded: Defined replicated reference data model explicitly
   - Added: Snapshot bootstrap + 6h refresh pattern
   - Clarified: No network calls as fallback; cache miss = fail-fast rejection
   
5. **Rule 3: MDS Constraint** (line ~506)
   - Strengthened: Forbidden calls detail (quotes, instruments, calendar)
   - Clarified: Even fallback calls are forbidden
   
6. **MDS REST Contract** (line ~939)
   - Added: `/api/v1/instruments` endpoint specification
   - Noted: Used for bootstrap/refresh (not per-order)
   
7. **Extraction Readiness** (line ~1054)
   - Updated: Gap 2 now resolved via smarttrade_common.instrument_master
   - Added: New component row for "Instrument Master Replication"
   
8. **Phase 1 Timeline** (line ~1066)
   - Updated: Merged Quote Cache + Instrument Master into one 20-28h phase
   - Clarified: All work centered on smarttrade_common replication

**Impact**: Clarified instrument ownership across entire architecture; resolved ambiguity in v4.0 where it said "BAS should not own" but didn't specify the distribution pattern.

---

## How This Resolves the Problem

### The Gap
**From BAS Alignment Analysis (2026-04-19)**:
> Gap 2: Instrument Cache — MEDIUM PRIORITY
> Current: BAS maintains InstrumentCache (symbol → instrument metadata)
> Target: BAS should not own instrument metadata; All instrument lookups → MDS (REST or cache)
> Problem: This leaves "how" underspecified — REST calls? Cache layer? What's the pattern?

### The Solution
**From this design**:
- **Clear ownership**: MDS owns authoritative master (ingestion from brokers)
- **Clear distribution**: Snapshot bootstrap + 6h periodic refresh
- **Clear execution**: Zero runtime MDS calls (all data in-memory)
- **Clear implementation**: Shared library (smarttrade_common.instrument_master)
- **Clear replication**: All services maintain identical replicas (not eventual consistency)

### The Benefit
1. **Execution Independence**: BAS never calls MDS for instruments (eliminating unpredictable latency)
2. **Simplicity**: Snapshot replication is a mature pattern (not event-driven CDC complexity)
3. **Code Reuse**: All services use same library (not duplicated symbol logic)
4. **Testability**: Services work offline without mocking MDS APIs
5. **Operability**: Scheduled jobs handle sync; no distributed-system CDC complexity

---

## Key Design Decisions

### 1. **Replicated vs. Centralized**
- ✅ **Chosen**: Replicated reference data
- All services maintain identical local copies
- Not eventual consistency; deterministic synchronization via versioning + checksums
- **Rationale**: Instruments rarely change; snapshot replication is robust and simple

### 2. **Snapshot vs. Event-Driven Replication**
- ✅ **Chosen**: Snapshot bootstrap + 6h periodic refresh
- Optional Phase 2: Add event-driven incremental updates (if needed)
- **Rationale**: Instruments don't change frequently; overkill to use event-driven CDC now; can add later if needed

### 3. **Shared Library vs. Per-Service**
- ✅ **Chosen**: Single smarttrade_common.instrument_master
- All services (BAS, PBS, Strategy, Portfolio) use same code
- **Rationale**: Eliminates duplicate symbol logic; ensures consistency; easier to maintain

### 4. **Persistent + In-Memory Caching**
- ✅ **Chosen**: InstrumentCache (persistent) + InstrumentRegistry (in-memory)
- Survives restarts (cache layer)
- <1ms lookups during execution (in-memory layer)
- **Rationale**: Best of both worlds — resilience + speed

---

## What This Means for Services

### Broker Adapter Service (BAS)
- Remove embedded InstrumentCache
- Use InstrumentSyncService from smarttrade_common
- Bootstrap at startup; refresh every 6 hours in background
- No MDS calls during order execution (all data in-memory)
- **Impact**: Cleaner code, zero MDS dependency, deterministic latency

### Paper Broker Service (PBS)
- Use InstrumentRegistry for fill validation
- Respect tick_size and lot_size rules
- Share InstrumentSyncService with BAS (optional; same cache)
- **Impact**: Improved fill accuracy; shared library reduces code

### Strategy Service
- Use InstrumentRegistry to filter tradeable symbols
- Validate signals respect lot_size and trading hours
- Use local copy (no MDS calls)
- **Impact**: Better signal quality; faster evaluation

### All Services
- Access instrument metadata via InstrumentValidator (shared APIs)
- No service-specific symbol logic
- All benefit from improvements made in one place
- **Impact**: Consistency, maintainability, code reuse

---

## Integration Timeline

**Phase 1 (Weeks 1-2)**: Build Library
- Implement 6 modules (models, registry, cache, sync_service, validators, bootstrap)
- 50+ unit tests
- Integration tests with mock MDS
- Documentation

**Phase 2 (Weeks 3-4)**: BAS Integration
- Replace BAS InstrumentCache with InstrumentSyncService
- Remove sync MDS calls from OrderHandler
- Verify execution latency <100ms
- Deploy to staging

**Phase 3 (Weeks 5-6)**: PBS Integration
- Use InstrumentRegistry for fill validation
- Add unit tests for tick/lot validation
- Canary deploy (10%)

**Phase 4 (Weeks 7-8)**: Production Rollout
- Ramp up traffic (10% → 50% → 100%)
- Monitor metrics (cache size, sync lag, failures)
- Keep rollback plan (feature flag)

---

## Critical Success Factors

1. **Zero MDS Calls in Execution Path**
   - Code review will verify
   - Static analysis will grep for MDS REST calls in OrderHandler
   - Integration tests will place orders with MDS unavailable

2. **Deterministic <100ms Latency**
   - Benchmark tests will measure latency
   - No fallback network calls on cache miss
   - Cache miss → fast fail (not retry)

3. **Replica Consistency**
   - Version + checksum verification
   - Monitoring for divergence
   - Ops manual refresh if needed

4. **Startup Resilience**
   - Bootstrap succeeds even if MDS unavailable
   - Uses previous cache + rejects orders with warning
   - Doesn't block service startup

---

## Implementation Checklist

**Before Implementation**:
- [ ] Review ADR + design with architecture team
- [ ] Identify backend implementation (DB, Redis, file)
- [ ] Estimate effort breakdown per module

**Phase 1 Implementation**:
- [ ] Create 6 modules in smarttrade_common
- [ ] Write 50+ unit tests
- [ ] Write integration tests with mock MDS
- [ ] Document in smarttrade_common README
- [ ] Code review + merge to main

**Phase 2-3 Integration**:
- [ ] Update BAS to use InstrumentSyncService
- [ ] Update PBS to use InstrumentRegistry
- [ ] Add health check endpoints
- [ ] Add monitoring metrics
- [ ] Integration tests (place order with MDS down)

**Phase 4 Production**:
- [ ] Staging deployment + testing
- [ ] Canary rollout (10% → 50% → 100%)
- [ ] Monitor metrics (no regressions)
- [ ] Gradual traffic increase

---

## Risks Identified & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Replicas go stale** | Orders fail for missing instruments | Monitor sync lag >12h; alert ops; auto-refresh every 6h |
| **New instruments lag** | Can't trade new symbols until next sync | Accept 6h lag; ops can trigger manual refresh anytime |
| **MDS unavailable at startup** | Service can't bootstrap | Retry with backoff; use previous cache; reject orders with warning; alert ops |
| **Sync failures accumulate** | Cache becomes progressively stale | Circuit breaker; alert after 3 consecutive failures |
| **Version mismatch between services** | Different services see different instruments | Sync on same schedule; monitor for divergence; alert if detected |

---

## Files Delivered

1. **ARCHITECTURE_DECISION_RECORD_instrument_master.md** (700 lines)
   - Complete decision + rationale
   - Problem → Solution → Implementation
   - Risk analysis + testing strategy

2. **DESIGN_smarttrade_common_instrument_master.md** (2000+ lines)
   - Complete HLD/LLD
   - Python code examples
   - Service integration patterns
   - Deployment guide

3. **FINAL_TARGET_ARCHITECTURE_v4.0.md** (updated)
   - 8 sections patched
   - Gap 2 resolved
   - Phase 1 timeline updated

4. **project_instrument_master_replicated_reference_data.md** (memory)
   - Quick reference for future sessions
   - Implementation roadmap
   - Success criteria

---

## Ready to Implement

The design is complete and ready for implementation. All ambiguities have been resolved:

✅ **What**: Instrument metadata as replicated reference data  
✅ **Why**: Execution path independence + simplicity + code reuse  
✅ **How**: Snapshot bootstrap + 6h refresh + shared smarttrade_common library  
✅ **When**: Phase 1 (Weeks 1-2, 20-28 hours)  
✅ **Who**: Architecture + BAS + PBS teams  
✅ **How to Validate**: Integration tests + metrics + monitoring  

**Next Action**: Schedule design review with architecture team, then begin Phase 1 implementation.

---

**Architecture Owner**: [Team Name]  
**Design Date**: 2026-04-24  
**Status**: Ready for Development
