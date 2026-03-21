# BAS — Settlement Design (T+1)

**Version:** 1.0
**Component:** `broker_adapter_service/services/settlement_service.py`

---

## 1. Overview

Settlement is the process of finalizing executed trades into realized P&L records. SmartTrade follows the Indian equity market T+1 settlement cycle (trades executed on day T are settled on T+1).

**Why settlement matters:**
- Separates "execution P&L" (fill price vs. entry cost) from "settled P&L" (final cash movement)
- Required for accurate available margin calculation
- Regulatory compliance: immutable settlement records

---

## 2. Settlement Model

```python
class Settlement(Base):
    id: UUID
    trade_id: UUID              # FK → Trade (immutable)
    user_id: UUID
    account_id: UUID
    symbol: str
    side: str                   # "BUY" | "SELL"
    qty: int
    execution_price: Decimal
    settlement_price: Decimal   # closing price on settlement date
    realized_pnl: Decimal       # (settlement_price - execution_price) * qty * direction
    status: str                 # "pending" | "processing" | "completed" | "failed"
    trade_date: date            # T (execution day)
    settlement_date: date       # T+1 (settlement day, skips holidays)
    settled_at: datetime | None
    created_at: datetime
```

---

## 3. Settlement Flow

### Step 1: Trade Capture

When a trade is executed (fill received), a Settlement record is immediately created with `status=pending`:

```python
# In OrderHandler.on_fill():
trade = await self.trade_repo.create(...)
settlement = Settlement(
    trade_id=trade.id,
    user_id=trade.user_id,
    trade_date=date.today(),
    settlement_date=calendar.next_trading_day("NSE", date.today()),
    status="pending"
)
await self.settlement_repo.create(settlement)
```

### Step 2: Settlement Processor (Background Job)

`SettlementProcessor` runs as a background task at market close and again at T+1 open:

```python
class SettlementProcessor:
    async def run_daily_settlement(self):
        # Run at market close (15:30 IST)
        today = date.today()
        settlement_date = today  # for T+1, this processes yesterday's T trades

        pending = await self.settlement_repo.get_pending_for_date(settlement_date)
        for settlement in pending:
            await self.process_one(settlement)

    async def process_one(self, settlement: Settlement):
        async with transaction():
            settlement.status = "processing"
            # Fetch settlement price (closing price from MDS)
            close_price = await self.mds_client.get_closing_price(
                settlement.symbol,
                settlement.settlement_date
            )
            settlement.settlement_price = close_price
            settlement.realized_pnl = self._compute_pnl(settlement, close_price)
            settlement.status = "completed"
            settlement.settled_at = datetime.utcnow()
            await self.settlement_repo.update(settlement)

        await self.event_bus.publish("settlement.completed", ...)
```

### P&L Computation

```python
def _compute_pnl(self, settlement: Settlement, close_price: Decimal) -> Decimal:
    direction = Decimal("1") if settlement.side == "SELL" else Decimal("-1")
    return (settlement.execution_price - close_price) * Decimal(str(settlement.qty)) * direction
```

Note: For a SELL trade, positive settlement P&L = execution price > close price.

---

## 4. Settlement Date Calculation

```python
from market_data_service.calendar import TradingCalendarResolver

def get_settlement_date(trade_date: date) -> date:
    # T+1 settlement, skip weekends and NSE holidays
    calendar = TradingCalendarResolver()
    return calendar.next_trading_day("NSE", trade_date)
```

---

## 5. Settlement Service

High-level service (used by routes + OrderHandler):

```python
class SettlementService:
    async def get_today_realized_pnl(self, user_id: UUID) -> Decimal:
        """Returns sum of completed settlement P&L for today's settlements."""
        settlements = await self.repo.get_completed_for_user_today(user_id)
        return sum(s.realized_pnl for s in settlements)

    async def get_pending_count(self, user_id: UUID) -> int:
        """Number of trades awaiting settlement."""
        return await self.repo.count_pending(user_id)

    async def get_settlement_history(
        self, user_id: UUID, from_date: date, to_date: date
    ) -> list[Settlement]:
        return await self.repo.get_for_user_in_range(user_id, from_date, to_date)
```

---

## 6. ACID Guarantees

Settlement processing uses explicit transactions:

```python
# CORRECT pattern — all-or-nothing
async with transaction():
    settlement.status = "processing"
    settlement.settlement_price = close_price
    settlement.realized_pnl = computed_pnl
    settlement.status = "completed"
    settlement.settled_at = now()
    await self.repo.update(settlement)
    # If any step fails, entire transaction rolls back
    # settlement stays "pending" and will be retried

# FORBIDDEN — partial updates
settlement.status = "processing"
await self.repo.update(settlement)   # partial write
# ... crash here leaves inconsistent state
settlement.status = "completed"
await self.repo.update(settlement)
```

---

## 7. Retry Handling

Failed settlements (status=`failed`) are retried by the processor:

- Max 3 attempts per settlement
- Retry delay: 5 minutes
- After 3 failures: alert + manual review required
- Failure reasons logged in `settlement.error_message` (not in schema yet — backlog item)

---

## 8. Settlement API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/settlement` | Settlement history (date range) |
| GET | `/api/v1/settlement/pending` | Pending settlements |
| GET | `/api/v1/settlement/today_pnl` | Today's realized P&L |

---

## 9. Test Coverage

| Test | Scope |
|------|-------|
| `test_settlement_service.py` | Service methods, P&L calculation |
| `test_settlement_full.py` | Full lifecycle: trade → pending → completed |
| `test_order_settlement_workflow.py` | Integration: order fill → settlement creation |
