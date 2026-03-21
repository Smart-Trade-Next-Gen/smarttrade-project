# Financial Correctness Rules

**This document is non-negotiable. Violations are bugs, not style preferences.**

---

## 1. Always Use Decimal

All monetary values, quantities involving money, and percentages in financial calculations must use Python's `decimal.Decimal`.

```python
# CORRECT
from decimal import Decimal

price = Decimal("2850.75")
qty = 10
pnl = (Decimal("2900.00") - price) * qty         # Decimal("493.50") ✓

# WRONG — float
price = 2850.75
pnl = (2900.00 - price) * qty                     # 494.9999999... ✗

# WRONG — int for price
pnl = (2900 - 2850) * 10                          # Loses precision ✗
```

### Why Decimal Matters

```python
>>> 0.1 + 0.2
0.30000000000000004          # float: WRONG

>>> Decimal("0.1") + Decimal("0.2")
Decimal('0.3')               # Decimal: CORRECT
```

In trading, accumulating float errors across thousands of trades is a financial loss. This is not theoretical.

---

## 2. Always Use String Literals for Decimal

```python
# CORRECT
Decimal("100")
Decimal("2850.75")
Decimal("0.02")

# WRONG
Decimal(100)                 # int → exact, but misleading
Decimal(100.0)               # float → Decimal("99.999999999999985786...") ✗
```

---

## 3. Division Must Use Decimal("100")

```python
# Risk percentage calculation
capital = Decimal("100000")
risk_amount = Decimal("2500")

# CORRECT
risk_pct = (risk_amount / capital) * Decimal("100")

# WRONG — introduces float
risk_pct = (risk_amount / capital) * 100.0         # Mixed types ✗
risk_pct = float(risk_amount) / float(capital) * 100  # float ✗
```

This is specifically what caused the `risk_engine/engine.py` bug (fixed in `fdb4039`): `100.0` was used instead of `Decimal("100")`.

---

## 4. ACID Transactions for All Mutations

Any operation that modifies order, trade, position, or settlement data must be wrapped in a transaction:

```python
# CORRECT
async with transaction(db):
    order = await order_repo.create(order_data)
    trade = await trade_repo.create(trade_data)
    position = await position_repo.update(position)
    await audit_repo.record(...)
# Commits all or rolls back all

# WRONG — partial writes
order = await order_repo.create(order_data)
# ← crash here: order exists but trade/position don't
trade = await trade_repo.create(trade_data)
```

---

## 5. Idempotency for All Operations

Order placement and settlement processing must be idempotent:

```python
# Order placement — idempotency key prevents duplicate orders
idempotency_key = sha256(f"{user_id}:{symbol}:{side}:{qty}:{price}:{nonce}")
existing = await idempotency.get(idempotency_key)
if existing:
    return existing  # Return cached result, don't re-place

result = await broker.place_order(...)
await idempotency.set(idempotency_key, result, ttl=86400)
```

Without idempotency: network timeout → client retries → duplicate order → double position.

---

## 6. Immutable Audit Records

Trade records and audit logs are **never updated or deleted**:

```python
class Trade(Base):
    __tablename__ = "trades"
    # No updated_at column
    # No delete method in repository
    # Row-level security in PostgreSQL prevents UPDATE/DELETE

class AuditRecord(Base):
    __tablename__ = "audit_log"
    # Append-only — no update, no delete
```

Any modification to a trade is represented as a new record, not an update.

---

## 7. User Isolation

Every database query that returns user data must filter by `user_id`:

```python
# CORRECT
positions = await position_repo.list(user_id=current_user.id)
order = await order_repo.get(order_id, user_id=current_user.id)

# WRONG — no user filter
positions = await db.execute(select(Position))  # Returns ALL users' positions ✗
order = await order_repo.get(order_id)          # Could return another user's order ✗
```

The base `repository.py` enforces this: `user_id=None` raises an assertion in non-admin contexts.

---

## 8. Monetary Values in API

API responses serialize Decimal as strings to preserve precision:

```python
class OrderResponse(BaseModel):
    price: str              # "2850.75" — not float
    filled_price: str | None

    @validator("price", pre=True)
    def serialize_decimal(cls, v):
        return str(v) if v is not None else None
```

Frontend receives `"2850.75"` (string), not `2850.75` (number). JavaScript floats have the same precision problem as Python floats.

---

## 9. No Raw SQL

Use SQLAlchemy ORM only. Never write raw SQL strings:

```python
# CORRECT
result = await db.execute(
    select(Order).where(Order.user_id == user_id, Order.status == "OPEN")
)

# WRONG — SQL injection risk
result = await db.execute(
    f"SELECT * FROM orders WHERE user_id = '{user_id}'"  # ✗
)
```

---

## 10. Settlement T+1 Calculation

Settlement date must use the trading calendar, not simple date arithmetic:

```python
# CORRECT — skips weekends and NSE holidays
settlement_date = calendar.next_trading_day("NSE", trade_date)

# WRONG — might land on holiday/weekend
settlement_date = trade_date + timedelta(days=1)  # ✗
```

---

## Checklist for Every PR Touching Money

- [ ] All new monetary variables use `Decimal`
- [ ] All Decimal literals use string form `Decimal("123.45")`
- [ ] Division multipliers use `Decimal("100")` not `100.0`
- [ ] Every DB mutation wrapped in `async with transaction(db)`
- [ ] All order/settlement operations have idempotency keys
- [ ] No raw SQL strings
- [ ] All queries filter by `user_id`
- [ ] New audit events added for regulatory traceability
- [ ] Settlement dates calculated via `TradingCalendarResolver`
