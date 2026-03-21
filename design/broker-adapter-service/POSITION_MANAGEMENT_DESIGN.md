# BAS — Position Management Design

**Version:** 1.0
**Component:** `broker_adapter_service/services/position_management_service.py`

---

## 1. Overview

Position Management allows traders to group related positions (e.g., legs of an option strategy) and manage them collectively — setting P&L targets, stop-losses, and automating group-level exits.

This is distinct from the raw position tracking (which is per-symbol, per-account). Position Management is a user-defined organizational layer on top.

---

## 2. Data Model

### PositionManagement (Group)

```python
class PositionManagement(Base):
    id: UUID
    user_id: UUID
    name: str                   # e.g., "NIFTY Bull Call Spread 22500-23000"
    description: str | None
    rules: dict                 # JSON: exit rules configuration
    status: str                 # "active" | "exited" | "archived"
    created_at: datetime
    exited_at: datetime | None
    exit_reason: str | None
```

### PositionGroupMember (junction table)

```python
class PositionGroupMember(Base):
    group_id: UUID
    position_id: UUID           # FK → Position
    added_at: datetime
    removed_at: datetime | None
```

### Position (snapshot)

```python
class Position(Base):
    id: UUID
    user_id: UUID
    account_id: UUID
    symbol: str
    net_qty: int                # positive=long, negative=short
    avg_price: Decimal
    current_price: Decimal      # updated from live quotes
    unrealized_pnl: Decimal     # (current_price - avg_price) * net_qty
    realized_pnl: Decimal       # from closed fills
    product_type: ProductType
    updated_at: datetime
```

---

## 3. Group Rules Schema

```json
{
  "exit_rules": [
    {
      "type": "pnl_target",
      "target": 2000,
      "action": "exit_all"
    },
    {
      "type": "stop_loss",
      "loss": -1000,
      "action": "exit_all"
    },
    {
      "type": "trailing_sl",
      "trail_pct": 25,
      "action": "exit_all"
    },
    {
      "type": "time_exit",
      "exit_at": "15:15",
      "action": "exit_all"
    }
  ],
  "position_sizing": {
    "max_legs": 4,
    "max_notional": 100000
  }
}
```

---

## 4. Service Operations

```python
class PositionManagementService:
    async def create_group(self, user_id, name, rules) -> PositionManagement

    async def add_position(self, user_id, group_id, position_id) -> None
        # Validates ownership of both group and position

    async def remove_position(self, user_id, group_id, position_id) -> None

    async def get_group_pnl(self, user_id, group_id) -> GroupPnLSummary:
        # Aggregates all member positions
        positions = await self.get_group_positions(user_id, group_id)
        unrealized = sum(p.unrealized_pnl for p in positions)
        realized = sum(p.realized_pnl for p in positions)
        return GroupPnLSummary(unrealized=unrealized, realized=realized, total=unrealized+realized)

    async def exit_group(self, user_id, group_id, reason="manual") -> None:
        # Place market orders to close all group positions
        positions = await self.get_group_positions(user_id, group_id)
        for position in positions:
            if position.net_qty != 0:
                await self.order_handler.place_order(
                    symbol=position.symbol,
                    side=SELL if position.net_qty > 0 else BUY,
                    qty=abs(position.net_qty),
                    order_type=MARKET
                )
        group.status = "exited"
        group.exited_at = now()
        group.exit_reason = reason
        await self.repo.update(group)

    async def evaluate_exit_rules(self, user_id, group_id) -> bool:
        """Called by RiskMonitorService. Returns True if any exit rule triggered."""
        group = await self.get_group(user_id, group_id)
        pnl = await self.get_group_pnl(user_id, group_id)
        for rule in group.rules.get("exit_rules", []):
            if self._check_rule(rule, pnl):
                await self.exit_group(user_id, group_id, reason=rule["type"])
                return True
        return False
```

---

## 5. Aggregated P&L

The `AggregatedPnLPanel` in the frontend shows group-level P&L refreshed on each quote update:

```
WebSocket tick arrives (NSE:NIFTY2540022500CE LTP updated)
    ↓
marketDataStore.updateLTP("NSE:NIFTY2540022500CE", 145.50)
    ↓
positionGroupSelectors.ts recomputes group P&L
    ↓
AggregatedPnLPanel re-renders
```

Position P&L is computed client-side from stored avg_price + live LTP, avoiding a server round-trip on every tick.

---

## 6. API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/position_groups` | Create group |
| GET | `/api/v1/position_groups` | List user's groups |
| GET | `/api/v1/position_groups/{id}` | Group detail + members |
| PUT | `/api/v1/position_groups/{id}` | Update rules |
| DELETE | `/api/v1/position_groups/{id}` | Delete (archive) group |
| POST | `/api/v1/position_groups/{id}/positions` | Add position to group |
| DELETE | `/api/v1/position_groups/{id}/positions/{pos_id}` | Remove from group |
| POST | `/api/v1/position_groups/{id}/exit` | Exit all positions in group |
| GET | `/api/v1/position_groups/{id}/pnl` | Group P&L summary |
| GET | `/api/v1/portfolio` | Full portfolio (all positions) |
| GET | `/api/v1/portfolio/summary` | P&L summary by segment |

---

## 7. Test Coverage

| Test | Scope |
|------|-------|
| `test_position_management_service.py` | CRUD, add/remove, P&L aggregation |
| `test_position_management_integration.py` | Integration: group creation → position add → exit |
| `test_portfolio_service.py` | Portfolio aggregation, P&L by segment |
| `test_position_repo.py` | DB: open positions query, update |
| `test_portfolio_and_position_routes.py` | HTTP: 26 tests across portfolio + position group routes |
