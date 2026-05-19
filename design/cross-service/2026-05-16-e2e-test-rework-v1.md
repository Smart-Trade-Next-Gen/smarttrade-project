# E2E Test Rework Plan — Stateless Architecture Alignment

**Status**: Design Ready for Implementation  
**Date**: 2026-05-16  
**Scope**: E2E test framework rework to align with v4.0 stateless architecture  
**Affected Services**: All services (BAS, MDS, PBS, Journal, Portfolio, Strategy, Notification, Auth)  
**Test Framework**: `smarttrade-tests/e2e/`

---

## Executive Summary

The current E2E test framework was designed for a stateful BAS architecture where BAS maintained local order/position/trade state and provided WebSocket account events to the frontend. The new v4.0 architecture makes BAS **stateless** with the broker as the single source of truth, removes BAS WebSocket account events, consolidates event schemas, and introduces the outbox pattern for critical events.

This rework plan aligns the E2E test framework with the new architecture while preserving test coverage and reliability.

**Key Changes:**
1. Remove BAS WebSocket client (account events no longer exposed)
2. Add broker state verification (broker is now source of truth)
3. Update event schemas (consolidated `order.updated` with status field)
4. Add outbox pattern testing (critical events durability)
5. Update instrument master testing (replication model)
6. Add synchronous BAS-PBS communication testing
7. Update read-side service testing (Journal/Portfolio event-driven)
8. Remove Strategy execution authority tests (advisory only)

**Success Criteria:**
- All existing test scenarios covered in new architecture
- Tests validate stateless BAS behavior
- Tests validate broker as source of truth
- Tests validate outbox pattern exactly-once semantics
- Tests validate event-driven read-side services
- Test execution time remains < 5s per test
- No regression in test reliability

---

## Architecture Changes Impact Analysis

### 1. BAS Stateless Execution Kernel

**Old Architecture:**
- BAS maintained local order/position/trade state
- BAS provided order/position queries via REST
- BAS tracked order lifecycle locally

**New Architecture:**
- BAS is stateless; broker is single source of truth
- BAS does NOT persist order/position/trade state
- BAS performs stateless operations only
- Broker state synchronized via WebSocket + API polling

**E2E Test Impact:**
- ❌ **BREAKING**: Cannot query BAS for order/position state
- ✅ **NEW**: Must query broker directly for state verification
- ✅ **NEW**: Must test hybrid state sync (WebSocket + polling)
- ✅ **NEW**: Must test BAS stateless behavior (no local persistence)

---

### 2. BAS WebSocket Account Events Removed

**Old Architecture:**
- BAS exposed WebSocket at `/api/v1/ws` for account events
- Frontend connected to BAS WebSocket for order/trade/position updates
- E2E tests used `bas_ws_client` for event collection

**New Architecture:**
- BAS has NO WebSocket to frontend
- Broker communication only (no user-facing WebSocket)
- Account events consumed internally by downstream services

**E2E Test Impact:**
- ❌ **BREAKING**: `bas_ws_client` is obsolete
- ❌ **BREAKING**: Cannot collect account events from BAS WebSocket
- ✅ **NEW**: Must collect events from Redis Streams directly
- ✅ **NEW**: Must validate downstream service event consumption
- ✅ **NEW**: Must test broker → BAS → event bus flow

---

### 3. Event Schema Consolidation

**Old Architecture:**
- Multiple event types: `order.placed`, `order.filled`, `trade.executed`, `position.updated`
- Separate events for each state transition

**New Architecture:**
- Consolidated `order.updated` event with status field
- Status values: `PLACED`, `ACCEPTED`, `FILLED`, `REJECTED`, `CANCELLED`
- Single event type for all order lifecycle transitions

**E2E Test Impact:**
- ❌ **BREAKING**: Event schema validation needs updating
- ❌ **BREAKING**: Event collection logic needs status field parsing
- ❌ **BREAKING**: Assertion engine needs status-based validation
- ✅ **NEW**: Must test status field transitions
- ✅ **NEW**: Must test event consolidation correctness

---

### 4. Outbox Pattern for Critical Events

**Old Architecture:**
- Fire-and-forget event publishing
- No transactional durability guarantees
- Event loss possible on crashes

**New Architecture:**
- Critical events use outbox pattern
- Exactly-once semantics via transactional outbox
- Outbox processor publishes events durably

**E2E Test Impact:**
- ✅ **NEW**: Must test outbox table writes
- ✅ **NEW**: Must test outbox processor behavior
- ✅ **NEW**: Must test exactly-once semantics
- ✅ **NEW**: Must test crash recovery (outbox replay)
- ✅ **NEW**: Must test idempotency with outbox

---

### 5. Instrument Master Replication

**Old Architecture:**
- MDS was authoritative source
- Services called MDS REST API at runtime for instrument resolution

**New Architecture:**
- MDS is still authoritative source
- Services maintain local replicated copy via `InstrumentSyncService`
- Bootstrap via snapshot + periodic refresh (6h)
- Zero runtime MDS calls during execution

**E2E Test Impact:**
- ✅ **NEW**: Must test instrument sync service
- ✅ **NEW**: Must test snapshot bootstrap
- ✅ **NEW**: Must test periodic refresh
- ✅ **NEW**: Must test fallback to MDS if sync fails
- ✅ **NEW**: Must test zero runtime MDS calls during execution

---

### 6. PBS Synchronous Communication

**Old Architecture:**
- PBS was internal service with async communication
- Event-driven communication between BAS and PBS

**New Architecture:**
- BAS calls PBS synchronously for paper account execution
- PBS behaves like external broker API
- Market data consumption via `market.quote`

**E2E Test Impact:**
- ✅ **NEW**: Must test BAS-PBS synchronous calls
- ✅ **NEW**: Must test PBS market data consumption
- ✅ **NEW**: Must test PBS subscription control plane
- ✅ **NEW**: Must test PBS event publishing (outbox)

---

### 7. Strategy Service Advisory Only

**Old Architecture:**
- Strategy Service could trigger execution (unclear authority)
- Potential distributed execution logic

**New Architecture:**
- Strategy Service is advisory only
- Decisions published as `strategy.decision` events
- No direct or indirect execution authority
- All execution originates from BAS-controlled flows

**E2E Test Impact:**
- ❌ **BREAKING**: Remove any tests assuming Strategy triggers execution
- ✅ **NEW**: Must test decision event publishing
- ✅ **NEW**: Must test advisory-only behavior
- ✅ **NEW**: Must test no execution authority

---

### 8. Journal/Portfolio Read-Only Event-Driven

**Old Architecture:**
- Mixed read/write patterns
- Some synchronous queries for state

**New Architecture:**
- Strictly read-only
- Event-driven aggregation
- No synchronous writes

**E2E Test Impact:**
- ✅ **NEW**: Must test event-driven aggregation
- ✅ **NEW**: Must test eventual consistency
- ✅ **NEW**: Must test read-only behavior (no writes)
- ✅ **NEW**: Must test consumer lag handling

---

## Current E2E Test Inventory

### Test Structure (Current)

```
smarttrade-tests/e2e/
├── clients/
│   ├── bas_client.py              # REST client (order placement, queries)
│   ├── bas_ws_client.py           # ❌ OBSOLETE: WebSocket account events
│   ├── mds_client.py              # WebSocket market data
│   ├── mds_rest_client.py         # REST instruments
│   ├── mock_client.py             # Mock service fill injection
│   ├── portfolio_client.py        # Portfolio Service queries
│   └── journal_client.py          # Journal Service queries
├── harness/
│   ├── event_collector.py         # Event collection (depends on bas_ws_client)
│   ├── assertions.py             # Order/position validation
│   ├── scenario_engine.py         # YAML scenario loading
│   ├── scenario_executor.py       # Scenario execution
│   └── redis_observer.py          # Redis stream observation
├── fixtures/
│   ├── market_data_stream.py      # Price injection
│   ├── chaos_engine.py            # Failure injection
│   └── instruments.py             # Instrument catalog
└── tests/
    ├── test_order_lifecycle_injection.py      # 4 tests
    ├── test_partial_fills_injection.py        # 3 tests
    ├── test_cancel_orders_injection.py        # 3 tests
    ├── test_error_paths_injection.py          # 5 tests
    ├── test_concurrent_orders_injection.py    # 3 tests
    ├── test_market_buy_real_execution.py      # 4 tests
    ├── test_partial_fills_real_execution.py   # 3 tests
    ├── test_execution_stress_scenarios.py     # 3 tests
    ├── test_resilience_timeouts.py            # 4 tests
    ├── test_resilience_event_handling.py      # 4 tests
    ├── test_resilience_partial_failures.py    # 3 tests
    ├── test_journal_integration.py            # 3 tests
    ├── test_portfolio_integration.py          # 3 tests
    ├── test_architecture_boundaries.py        # Architecture validation
    ├── test_websocket_client_routing.py        # WebSocket separation
    ├── test_websocket_separation_live.py      # Live WebSocket tests
    └── test_event_bus_validation.py           # Event bus validation
```

**Total: 39 tests**

### Test Categories (Current)

| Category | Count | Mode | Purpose |
|----------|-------|------|---------|
| Smoke | 2 | Injection | Quick sanity check |
| Injection | 18 | Deterministic | Financial validation |
| Real Execution | 10 | Price-driven | Realistic execution |
| Resilience | 11 | Chaos | Failure recovery |
| Integration | 6 | Event-driven | Service integration |

---

## Rework Plan

### Phase 1: Infrastructure Updates (Foundation)

#### 1.1 Remove BAS WebSocket Client

**File:** `e2e/clients/bas_ws_client.py`

**Action:** DELETE

**Rationale:** BAS no longer exposes WebSocket for account events. Broker communication is internal only.

**Dependencies:**
- `event_collector.py` (depends on bas_ws_client)
- `conftest.py` (bas_ws_client fixture)
- All tests using bas_ws_client

---

#### 1.2 Add Broker State Client

**File:** `e2e/clients/broker_state_client.py` (NEW)

**Purpose:** Query broker directly for order/position/trade state

**Interface:**
```python
class BrokerStateClient:
    async def get_order_state(broker_id, account_id, order_id) -> dict
    async def get_position_state(broker_id, account_id, instrument_id) -> dict
    async def get_trade_state(broker_id, account_id, order_id) -> dict
    async def get_account_state(broker_id, account_id) -> dict
```

**Implementation:**
- For Fyers: Use Fyers API directly
- For PBS: Use PBS internal API
- Abstract broker-specific differences behind unified interface

**Fixtures to Add:**
```python
@pytest.fixture
async def broker_state_client(config, auth_token) -> BrokerStateClient:
    client = BrokerStateClient(
        broker_type=config.broker_type,  # "fyers" or "pbs"
        base_url=config.broker_api_url,
        token=auth_token,
    )
    await client.connect()
    yield client
    await client.disconnect()
```

---

#### 1.3 Add Redis Stream Event Collector

**File:** `e2e/harness/redis_event_collector.py` (NEW)

**Purpose:** Collect events directly from Redis Streams (bypassing BAS WebSocket)

**Interface:**
```python
class RedisEventCollector:
    async def subscribe_to_streams(stream_patterns: list[str])
    async def wait_for_event(order_id: str, event_type: str, timeout: float) -> dict
    async def wait_for_completion(order_id: str, timeout: float) -> list[dict]
    def get_events(order_id: str) -> list[dict]
```

**Streams to Subscribe:**
- `events:order.updated` (consolidated order events)
- `events:trade.executed` (trade events)
- `events:position.updated` (position events)

**Implementation:**
- Use Redis XREADGROUP for consumer group-based reading
- Use service-scoped consumer groups per architecture
- Implement idempotency via event_id
- Handle stream trimming and message expiration

**Fixtures to Add:**
```python
@pytest.fixture
async def redis_event_collector(config) -> RedisEventCollector:
    collector = RedisEventCollector(
        redis_url=config.redis_url,
        consumer_group=f"e2e-tests-{uuid.uuid4().hex[:8]}",
    )
    await collector.subscribe_to_streams([
        "events:order.updated",
        "events:trade.executed",
        "events:position.updated",
    ])
    yield collector
    await collector.cleanup()
```

---

#### 1.4 Update Event Collector

**File:** `e2e/harness/event_collector.py`

**Changes:**
- Remove dependency on `bas_ws_client`
- Add support for new consolidated event schema (`order.updated`)
- Update status parsing (status field instead of event type)
- Update terminal status detection

**New Status Handling:**
```python
# Old: Multiple event types
event_types = ["order.placed", "order.filled", "trade.executed"]

# New: Single event type with status field
event_type = "order.updated"
status = event.get("status")  # PLACED, FILLED, REJECTED, etc.
```

---

#### 1.5 Update Assertion Engine

**File:** `e2e/harness/assertions.py`

**Changes:**
- Update order lifecycle assertions for consolidated event schema
- Update status-based validation
- Add broker state verification assertions
- Add outbox pattern validation

**New Assertions:**
```python
def assert_order_lifecycle_v2(events: list[dict], expected_status: str, expected_qty: int)
def assert_broker_state_matches_events(broker_state: dict, events: list[dict])
def assert_outbox_event_published(outbox_record: dict, event: dict)
def assert_status_transition_correct(events: list[dict], expected_transitions: list[str])
```

---

#### 1.6 Update Configuration

**File:** `e2e/config/config.py`

**Add Configuration:**
```python
class TestConfig:
    broker_type: str  # "fyers" or "pbs"
    broker_api_url: str  # Direct broker API URL
    redis_stream_consumer_group: str  # For event collection
    outbox_table_name: str  # For outbox testing
```

**Environment Variables:**
```bash
BROKER_TYPE=fyers  # or pbs
BROKER_API_URL=https://api.fyers.in  # or http://pbs:8002
REDIS_STREAM_CONSUMER_GROUP=e2e-tests
OUTBOX_TABLE_NAME=outbox
```

---

### Phase 2: Test Updates (Core Functionality)

#### 2.1 Update Order Lifecycle Tests

**File:** `e2e/tests/test_order_lifecycle_injection.py`

**Changes:**
- Replace `bas_ws_client` with `redis_event_collector`
- Replace BAS state queries with `broker_state_client` queries
- Update event schema validation
- Add broker state verification

**Example Test Update:**
```python
# OLD
async def test_market_buy_full_fill(
    bas_client, bas_ws_client, event_collector, ...
):
    # Place order
    order = await bas_client.place_order(...)
    
    # Wait for BAS WebSocket events
    events = await event_collector.wait_for_completion(order_id)
    
    # Assert BAS state
    bas_order = await bas_client.get_order(order_id)

# NEW
async def test_market_buy_full_fill(
    bas_client, broker_state_client, redis_event_collector, ...
):
    # Place order
    order = await bas_client.place_order(...)
    
    # Wait for Redis stream events
    events = await redis_event_collector.wait_for_completion(order_id)
    
    # Assert broker state (source of truth)
    broker_order = await broker_state_client.get_order_state(order_id)
    
    # Assert event matches broker state
    assertions.assert_broker_state_matches_events(broker_order, events)
```

**Test Count:** 4 tests (unchanged)

---

#### 2.2 Update Partial Fill Tests

**File:** `e2e/tests/test_partial_fills_injection.py`

**Changes:**
- Replace `bas_ws_client` with `redis_event_collector`
- Update for consolidated event schema
- Add broker state verification after each partial fill

**Test Count:** 3 tests (unchanged)

---

#### 2.3 Update Cancel Order Tests

**File:** `e2e/tests/test_cancel_orders_injection.py`

**Changes:**
- Replace `bas_ws_client` with `redis_event_collector`
- Verify broker cancellation state
- Test status transition: PLACED → CANCELLED

**Test Count:** 3 tests (unchanged)

---

#### 2.4 Update Error Path Tests

**File:** `e2e/tests/test_error_paths_injection.py`

**Changes:**
- Update for consolidated event schema
- Verify REJECTED status in events
- Verify broker rejection state

**Test Count:** 5 tests (unchanged)

---

#### 2.5 Update Concurrent Order Tests

**File:** `e2e/tests/test_concurrent_orders_injection.py`

**Changes:**
- Replace `bas_ws_client` with `redis_event_collector`
- Verify broker handles concurrent orders correctly
- Test event ordering guarantees

**Test Count:** 3 tests (unchanged)

---

### Phase 3: Real Execution Tests (Price-Driven)

#### 3.1 Update Real Execution Tests

**Files:** 
- `e2e/tests/test_market_buy_real_execution.py`
- `e2e/tests/test_partial_fills_real_execution.py`
- `e2e/tests/test_execution_stress_scenarios.py`

**Changes:**
- Replace `bas_ws_client` with `redis_event_collector`
- Update for consolidated event schema
- Add broker state verification
- Test PBS market data consumption (if using PBS)

**Test Count:** 10 tests (unchanged)

---

### Phase 4: Resilience Tests (Chaos)

#### 4.1 Update Resilience Tests

**Files:**
- `e2e/tests/test_resilience_timeouts.py`
- `e2e/tests/test_resilience_event_handling.py`
- `e2e/tests/test_resilience_partial_failures.py`

**Changes:**
- Replace `bas_ws_client` with `redis_event_collector`
- Test outbox pattern crash recovery
- Test exactly-once semantics under failures
- Test broker state consistency after recovery

**Test Count:** 11 tests (unchanged)

**New Test Scenarios:**
```python
async def test_outbox_crash_recovery():
    # Place order
    # Kill BAS before outbox processor publishes
    # Restart BAS
    # Verify outbox processor replays events
    # Verify exactly-once delivery

async def test_event_idempotency_under_failure():
    # Inject duplicate events
    # Verify downstream services handle idempotency
    # Verify no duplicate state updates
```

---

### Phase 5: Integration Tests (Service-Specific)

#### 5.1 Update Journal Integration Tests

**File:** `e2e/tests/test_journal_integration.py`

**Changes:**
- Verify Journal consumes events from Redis Streams
- Test event-driven aggregation
- Test eventual consistency
- Verify read-only behavior (no writes)

**Test Count:** 3 tests (unchanged)

---

#### 5.2 Update Portfolio Integration Tests

**File:** `e2e/tests/test_portfolio_integration.py`

**Changes:**
- Verify Portfolio consumes events from Redis Streams
- Test event-driven position aggregation
- Test eventual consistency
- Verify read-only behavior (no writes)

**Test Count:** 3 tests (unchanged)

---

#### 5.3 Add Instrument Sync Tests

**File:** `e2e/tests/test_instrument_sync.py` (NEW)

**Purpose:** Test instrument master replication

**Test Scenarios:**
```python
async def test_instrument_sync_bootstrap():
    # Verify InstrumentSyncService bootstraps from MDS
    # Verify local replica created
    # Verify zero runtime MDS calls during execution

async def test_instrument_sync_periodic_refresh():
    # Verify periodic refresh (6h)
    # Verify replica stays in sync
    # Verify fallback to MDS if sync fails

async def test_instrument_resolution_from_replica():
    # Verify BAS resolves instruments from local replica
    # Verify no MDS REST API calls during execution
```

**Test Count:** 3 tests (NEW)

---

#### 5.4 Add Outbox Pattern Tests

**File:** `e2e/tests/test_outbox_pattern.py` (NEW)

**Purpose:** Test outbox pattern exactly-once semantics

**Test Scenarios:**
```python
async def test_outbox_event_published():
    # Place order
    # Verify outbox record created
    # Verify event published to Redis
    # Verify outbox record marked as published

async def test_outbox_exactly_once_semantics():
    # Place order
    # Simulate outbox processor crash
    # Restart outbox processor
    # Verify event published exactly once

async def test_outbox_idempotency():
    # Place same order twice (same idempotency key)
    # Verify single outbox record
    # Verify event published once
```

**Test Count:** 3 tests (NEW)

---

#### 5.5 Add BAS-PBS Synchronous Tests

**File:** `e2e/tests/test_bas_pbs_sync.py` (NEW)

**Purpose:** Test synchronous BAS-PBS communication

**Test Scenarios:**
```python
async def test_bas_calls_pbs_synchronously():
    # Place order for paper account
    # Verify BAS calls PBS synchronously
    # Verify PBS returns execution result
    # Verify BAS publishes event based on PBS result

async def test_pbs_market_data_consumption():
    # Subscribe to instrument via PBS
    # Inject market data via MDS
    # Verify PBS consumes from market.quote
    # Verify PBS uses realistic pricing

async def test_pbs_subscription_control():
    # PBS publishes subscription request
    # Verify MDS acknowledges
    # Verify market data flows
```

**Test Count:** 3 tests (NEW)

---

#### 5.6 Update Strategy Service Tests

**File:** `e2e/tests/test_strategy_advisory.py` (NEW)

**Purpose:** Test Strategy Service advisory-only behavior

**Test Scenarios:**
```python
async def test_strategy_publishes_decision_events():
    # Trigger strategy condition
    # Verify strategy.decision event published
    # Verify no order execution triggered

async def test_strategy_no_execution_authority():
    # Strategy publishes decision
    # Verify BAS does NOT execute automatically
    # Verify execution only via explicit BAS call

async def test_strategy_load_control():
    # Flood market data
    # Verify Strategy throttles evaluation
    # Verify backpressure handling
```

**Test Count:** 3 tests (NEW)

---

### Phase 6: Architecture Boundary Tests

#### 6.1 Update Architecture Boundary Tests

**File:** `e2e/tests/test_architecture_boundaries.py`

**Changes:**
- Update for stateless BAS behavior
- Test broker as source of truth
- Test BAS stateless operations
- Test service boundary violations

**Test Count:** Existing tests updated

---

#### 6.2 Update WebSocket Separation Tests

**Files:**
- `e2e/tests/test_websocket_client_routing.py`
- `e2e/tests/test_websocket_separation_live.py`

**Changes:**
- Remove BAS WebSocket tests (no longer exists)
- Focus on MDS WebSocket (market data only)
- Test that BAS has NO user-facing WebSocket

**Test Count:** Reduced (BAS WebSocket tests removed)

---

#### 6.3 Update Event Bus Validation Tests

**File:** `e2e/tests/test_event_bus_validation.py`

**Changes:**
- Update for consolidated event schema
- Test service-scoped consumer groups
- Test event schema validation
- Test event routing

**Test Count:** Existing tests updated

---

## Implementation Sequence

### Sequence 1: Foundation (Week 1-2)
1. Remove `bas_ws_client.py`
2. Add `broker_state_client.py`
3. Add `redis_event_collector.py`
4. Update `event_collector.py`
5. Update `assertions.py`
6. Update `config.py`
7. Update `conftest.py` (remove bas_ws_client fixture, add new fixtures)

### Sequence 2: Core Tests (Week 3-4)
1. Update `test_order_lifecycle_injection.py`
2. Update `test_partial_fills_injection.py`
3. Update `test_cancel_orders_injection.py`
4. Update `test_error_paths_injection.py`
5. Update `test_concurrent_orders_injection.py`
6. Run and validate all injection tests

### Sequence 3: Real Execution (Week 5)
1. Update `test_market_buy_real_execution.py`
2. Update `test_partial_fills_real_execution.py`
3. Update `test_execution_stress_scenarios.py`
4. Run and validate all real execution tests

### Sequence 4: Resilience (Week 6)
1. Update `test_resilience_timeouts.py`
2. Update `test_resilience_event_handling.py`
3. Update `test_resilience_partial_failures.py`
4. Add outbox crash recovery tests
5. Run and validate all resilience tests

### Sequence 5: Integration (Week 7-8)
1. Update `test_journal_integration.py`
2. Update `test_portfolio_integration.py`
3. Add `test_instrument_sync.py`
4. Add `test_outbox_pattern.py`
5. Add `test_bas_pbs_sync.py`
6. Add `test_strategy_advisory.py`
7. Run and validate all integration tests

### Sequence 6: Architecture (Week 9)
1. Update `test_architecture_boundaries.py`
2. Update `test_websocket_client_routing.py`
3. Update `test_websocket_separation_live.py`
4. Update `test_event_bus_validation.py`
5. Run and validate all architecture tests

### Sequence 7: Full Regression (Week 10)
1. Run full test suite (all 39+ tests)
2. Fix any remaining issues
3. Update documentation
4. Validate CI/CD pipeline

---

## Test Count Summary

| Category | Old Count | New Count | Change |
|----------|-----------|-----------|--------|
| Smoke | 2 | 2 | 0 |
| Injection | 18 | 18 | 0 |
| Real Execution | 10 | 10 | 0 |
| Resilience | 11 | 14 | +3 |
| Integration | 6 | 15 | +9 |
| Architecture | 4 | 3 | -1 |
| **TOTAL** | **51** | **62** | **+11** |

**Note:** The old total was 39 tests in the README, but the actual count including integration/architecture tests is 51. The new total is 62 tests.

---

## Risks and Mitigations

### Risk 1: Broker API Rate Limits
**Risk:** Querying broker directly for state may hit rate limits during tests

**Mitigation:**
- Use test-specific broker credentials with higher limits
- Implement query throttling in `broker_state_client`
- Cache broker state responses where appropriate

---

### Risk 2: Redis Stream Consumer Group Complexity
**Risk:** Redis stream consumer groups add complexity to event collection

**Mitigation:**
- Use unique consumer group per test run
- Implement proper consumer group cleanup
- Add comprehensive logging for debugging
- Fall back to direct XREAD if consumer groups fail

---

### Risk 3: Event Schema Transition
**Risk:** Consolidated event schema may not match actual implementation

**Mitigation:**
- Verify event schema against actual service implementation
- Add schema validation tests
- Make event parsing flexible to handle schema variations
- Coordinate with service implementation team

---

### Risk 4: Test Execution Time Increase
**Risk:** Additional broker state queries may slow down tests

**Mitigation:**
- Optimize broker state queries (batch where possible)
- Use efficient Redis stream reading
- Maintain < 5s per test target
- Parallelize tests where possible

---

### Risk 5: PBS Synchronous Call Latency
**Risk:** Synchronous BAS-PBS calls may add latency to tests

**Mitigation:**
- Use local PBS for E2E tests (low latency)
- Configure PBS for fast execution in test mode
- Add timeout handling for synchronous calls

---

## Success Criteria

### Functional Criteria
- ✅ All 62 tests pass consistently
- ✅ Tests validate stateless BAS behavior
- ✅ Tests validate broker as source of truth
- ✅ Tests validate outbox pattern exactly-once semantics
- ✅ Tests validate event-driven read-side services
- ✅ Tests validate instrument master replication
- ✅ Tests validate BAS-PBS synchronous communication
- ✅ Tests validate Strategy advisory-only behavior

### Non-Functional Criteria
- ✅ Test execution time < 5s per test (average)
- ✅ Test reliability > 95% pass rate
- ✅ No flaky tests due to timing issues
- ✅ Clear error messages for failures
- ✅ Comprehensive logging for debugging

### Documentation Criteria
- ✅ E2E test README updated with new architecture
- ✅ Test strategy document updated
- ✅ Client documentation updated
- ✅ Fixture documentation updated
- ✅ CI/CD pipeline documentation updated

---

## Dependencies and Blocking

### External Dependencies
1. **BAS Implementation:** Stateless BAS must be fully implemented
2. **Event Schema:** Consolidated `order.updated` schema must be finalized
3. **Outbox Pattern:** Outbox table and processor must be implemented
4. **Instrument Sync:** InstrumentSyncService must be implemented
5. **PBS Synchronous:** PBS synchronous API must be implemented

### Internal Dependencies
1. **smarttrade-common:** EventBus updates for consolidated schema
2. **smarttrade-common:** Schema registry updates
3. **Service Implementations:** All services must align with v4.0 architecture

### Sequencing Dependencies
- Phase 1 (Infrastructure) must complete before Phase 2 (Tests)
- Service implementations must be complete before E2E test rework
- Outbox pattern must be implemented before resilience tests

---

## Open Questions

1. **Broker State Client Implementation:** Should we use Fyers API directly or wrap existing broker adapter code?
2. **Redis Stream Consumer Groups:** Should we use service-scoped groups or test-specific groups?
3. **Event Schema Versioning:** How do we handle event schema transitions during testing?
4. **Test Account Management:** How do we manage test accounts with broker credentials?
5. **PBS Test Mode:** Does PBS need a specific test mode for E2E tests?

---

## Next Steps

1. **Review this plan** with architecture and service teams
2. **Confirm event schema** with BAS team
3. **Confirm outbox implementation** with BAS team
4. **Confirm instrument sync** with all service teams
5. **Confirm PBS synchronous API** with PBS team
6. **Approve implementation sequence** with all stakeholders
7. **Begin Phase 1 implementation** (Infrastructure updates)

---

## Appendix: Event Schema Mapping

### Old Event Schemas
```yaml
order.placed:
  event_id: string
  order_id: string
  status: "PLACED"
  timestamp: datetime

order.filled:
  event_id: string
  order_id: string
  fill_qty: int
  fill_price: decimal
  status: "FILLED"
  timestamp: datetime

trade.executed:
  event_id: string
  order_id: string
  trade_id: string
  qty: int
  price: decimal
  timestamp: datetime

position.updated:
  event_id: string
  account_id: string
  instrument_id: string
  net_qty: int
  avg_price: decimal
  timestamp: datetime
```

### New Event Schema
```yaml
order.updated:
  event_id: string
  order_id: string
  status: "PLACED" | "ACCEPTED" | "FILLED" | "REJECTED" | "CANCELLED"
  fills: list[Fill]  # For FILLED status
  timestamp: datetime
  
Fill:
  fill_id: string
  qty: int
  price: decimal
  timestamp: datetime
```

**Note:** Trade and position events may remain separate or be consolidated. This needs confirmation from the architecture team.

---

**Document Status:** Design Ready for Review  
**Next Review Date:** 2026-05-17  
**Owner:** E2E Test Team  
**Approvers:** Architecture Team, Service Teams
