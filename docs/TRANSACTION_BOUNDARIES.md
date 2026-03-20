# Transaction Boundaries - SmartTrade Financial Operations

## Overview

This document defines ACID transaction boundaries for all critical financial operations in SmartTrade. Each transaction must be atomic, isolated, and durable to prevent data corruption and financial losses.

## Core Principle

**Every financial operation touching portfolio state must be within a single transaction.**

## Transaction Scopes

### 1. Order Placement (Atomic)

**Scope**: User submits order → Order created in DB

```python
@transactional
async def place_order(session, user_id, broker_id, order):
    # All within single transaction:
    # 1. Validate order against current risk snapshot
    # 2. Check risk limits (margin, position size, daily loss)
    # 3. Create Order record in DB
    # 4. Block/reserve margin in funds
    # 5. Create audit log entry
    # All succeed or all roll back
```

**Isolation**: `SERIALIZABLE` or `READ_COMMITTED`

**Constraints**:
- No dirty reads (can't see uncommitted orders)
- No phantom reads (position count consistent)
- Risk snapshot frozen during transaction

---

### 2. Order Fill / Trade Execution (Atomic)

**Scope**: Broker confirms order filled → Portfolio updated

```python
@transactional
async def execute_trade(session, order_id, fill_qty, fill_price):
    # All within single transaction:
    # 1. Verify order exists and is valid
    # 2. Create Trade record
    # 3. Update Position (quantity, avg_price, realized_pnl)
    # 4. Update Portfolio cash (debit/credit)
    # 5. Release blocked margin
    # 6. Update risk snapshot
    # 7. Create audit log
    # All succeed or all roll back
```

**Isolation**: `SERIALIZABLE` (critical for position accuracy)

**Constraints**:
- Position quantity must match trade quantity
- Cash balance must update with trade
- No lost fills

---

### 3. Position Close / Partial Close (Atomic)

**Scope**: Close order submitted → Position quantity reduced

```python
@transactional
async def close_position(session, position_id, close_qty, close_price):
    # All within single transaction:
    # 1. Verify position exists and has quantity
    # 2. Calculate realized PnL: (close_price - avg_price) × qty
    # 3. Update position (net_qty, realized_pnl)
    # 4. Update cash balance (add realized gains/losses)
    # 5. If full close, mark position CLOSED
    # 6. Create audit log
    # All succeed or all roll back
```

**Isolation**: `SERIALIZABLE`

**Constraints**:
- Close quantity ≤ open quantity
- Realized PnL must be exact
- Cash balance must reflect PnL

---

### 4. Risk Check + Order Validation (Distributed Lock + Transaction)

**Scope**: Before order placement, validate all risk limits

```python
async with distributed_lock(f"lock:risk:{user_id}:{broker_id}:{account_id}"):
    # Lock acquired: no concurrent orders for this account
    @transactional
    async def validate_and_place(session):
        # Within transaction + lock:
        # 1. Get current risk snapshot (frozen)
        # 2. Estimate this order's risk
        # 3. Check all limits:
        #    - Max daily loss: realized_pnl + estimated_loss
        #    - Position limit: current_position + new_qty ≤ limit
        #    - Margin check: available_margin ≥ required_margin
        # 4. If all pass: create order + reserve margin
        # All succeed atomically with lock held
```

**Isolation**: `READ_COMMITTED` + distributed lock

**Constraints**:
- Lock held until transaction commits
- No concurrent risk evals for same account
- Snapshot consistent during check

---

### 5. Settlement / T+1 Finalization (Atomic)

**Scope**: Settlement date arrives → Positions become "free" (no longer locked)

```python
@transactional
async def settle_trade(session, trade_id):
    # All within single transaction:
    # 1. Verify settlement conditions met (T+1 passed)
    # 2. Release blocked margin for this trade
    # 3. Update position settlement status
    # 4. Update portfolio available cash
    # 5. Create audit log
    # All succeed or all roll back
```

**Isolation**: `READ_COMMITTED`

**Constraints**:
- Settlement only after T+1 passed
- Margin freed up atomically
- No partial settlements

---

### 6. Risk Setting Update (Atomic)

**Scope**: User changes risk limits

```python
@transactional
async def update_risk_settings(session, user_id, broker_id, account_id, new_limits):
    # All within single transaction:
    # 1. Validate new limits are reasonable
    # 2. Check no active violations with new limits
    # 3. Update RiskConfigOverride
    # 4. Publish event
    # 5. Create audit log
    # All succeed or all roll back
```

**Isolation**: `READ_COMMITTED`

**Constraints**:
- New limits can't be more restrictive than current position
- Event published iff DB update succeeds

---

## Database Setup for Transactions

### PostgreSQL Configuration

```sql
-- Set isolation level for all transactions
-- (Individual services can override per-transaction)
SET default_transaction_isolation = 'serializable';

-- Ensure wal_level supports replication/recovery
-- wal_level = replica (minimum for safety)

-- Connection timeout prevents long-held locks
statement_timeout = 30000; -- 30 seconds

-- Deadlock detection (automatic retry recommended in app)
deadlock_timeout = 1000; -- 1 second
```

### SQLAlchemy Configuration

```python
@transactional
async def critical_operation(session: AsyncSession):
    # Automatically wrapped in transaction
    # On exception: automatic rollback
    # On success: automatic commit
    pass

# For explicit isolation level:
@transactional(isolation_level="SERIALIZABLE")
async def position_close(session):
    # Highest isolation level
    # Prevents all anomalies but may have deadlocks
    pass
```

---

## Isolation Levels

| Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Use Case |
|-------|------------|----------------------|---------------|----------|
| READ_UNCOMMITTED | ✗ | ✗ | ✗ | Not used (unsafe) |
| READ_COMMITTED | ✓ | ✗ | ✗ | Audit logs, settings |
| REPEATABLE_READ | ✓ | ✓ | ✗ | Most operations |
| SERIALIZABLE | ✓ | ✓ | ✓ | **Order fills, positions** |

**Recommendation**: Use `SERIALIZABLE` for all position/cash updates.

---

## Common Mistakes (NEVER DO)

❌ **Mistake 1**: Read position outside transaction, decide within it
```python
# WRONG: Race condition!
position = await get_position(pos_id)
if position.qty > 10:
    # Another trade could have reduced qty here!
    await close_position(pos_id, qty=position.qty)
```

✅ **Correct**: All within transaction
```python
# RIGHT: Atomic
@transactional
async def close_position(session, pos_id, qty):
    position = await repo.get(session, pos_id)
    if position.qty > 10:
        # Can't change anymore, transaction has lock
        await update(session, position, qty=position.qty - qty)
```

---

❌ **Mistake 2**: Calling external API within transaction
```python
# WRONG: Transaction locked while calling broker
@transactional
async def submit_order(session, order):
    await broker_api.place_order(order)  # Long wait!
    # Other transactions blocked, deadlock risk
```

✅ **Correct**: Async callback after transaction
```python
# RIGHT: Transaction completes, then notify
@transactional
async def create_order(session, order):
    # Create in DB (fast)
    order = await repo.create(session, order)
    # Transaction commits here

# Then async callback (no lock)
await broker_api.place_order(order)
```

---

❌ **Mistake 3**: Updating shared state without lock
```python
# WRONG: Two orders race to update funds
funds.available -= order1.margin  # Order 1 reads: 100k
funds.available -= order2.margin  # Order 2 reads: 100k
# Result: Only order 1's margin deducted!
```

✅ **Correct**: Distributed lock + transaction
```python
# RIGHT: No concurrent updates
async with distributed_lock(f"lock:funds:{account_id}"):
    @transactional
    async def reserve_margin(session):
        funds = await repo.get(session, account_id)
        funds.available -= margin
        await repo.update(session, funds)
```

---

## Testing Transactions

### Unit Test Pattern

```python
@pytest.mark.asyncio
async def test_order_placement_rolls_back_on_error(session):
    # Arrange: Setup initial state
    await setup_account_with_margin(session, 100_000)

    # Act: Try to place order that violates risk limit
    with pytest.raises(RiskViolation):
        await order_handler.place(session, invalid_order)

    # Assert: Order NOT in DB, margin NOT reserved
    orders = await repo.list_orders(session, user_id)
    assert len(orders) == 0  # Rolled back!

    funds = await funds_repo.get(session, account_id)
    assert funds.available == 100_000  # Unchanged!
```

### Integration Test Pattern

```python
@pytest.mark.asyncio
async def test_concurrent_orders_no_race():
    # Spawn 10 concurrent orders for same account
    tasks = [
        place_order(broker_id, account_id, order_i)
        for i in range(10)
    ]

    results = await asyncio.gather(*tasks, return_exceptions=True)

    # All should succeed (distributed lock prevents race)
    # OR some should fail (risk limit exceeded) deterministically
    # But NO data corruption
```

---

## Rollout Safety

When deploying transaction changes:

1. **Test locally** with `pytest`
2. **Load test** with 100+ concurrent orders
3. **Rollback plan**: Revert to previous transaction boundaries (data stays consistent)
4. **Monitoring**: Alert on transaction timeouts, deadlocks
5. **Gradual**: Deploy to dev/staging first, validate, then prod

---

## References

- SQLAlchemy Transactions: https://docs.sqlalchemy.org/en/20/orm/session_transaction.html
- PostgreSQL Isolation: https://www.postgresql.org/docs/current/transaction-iso.html
- Trading System Correctness: "Building a Trading System" by Andrew Younger
