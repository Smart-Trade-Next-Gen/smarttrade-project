# BAS — PIE (Position Intelligence Engine) Design

**Version:** 1.0
**Component:** `broker_adapter_service/` (strategies, auto_entry, kill_switch, action_orchestrator, pie_status)

---

## 1. Overview

PIE is SmartTrade's automation layer. It allows traders to define rules that automatically manage positions — entering trades on signal, exiting on conditions, and triggering emergency stops.

**Core capabilities:**
- **Strategies**: Named rule sets that activate/deactivate position management behaviors
- **Auto-entry**: Trigger entry orders when configurable conditions are met
- **Kill switch**: Emergency stop — cancel all open orders, close all positions immediately
- **Action Orchestrator**: Execute multi-step action plans triggered by risk or strategy rules
- **PIE Status**: Real-time aggregated view of all active automation state

---

## 2. Component Map

```
PIE Layer
    ├── StrategyService          CRUD for strategy definitions
    ├── ActiveStrategy (model)   Running strategy state
    ├── AutoEntryService         Auto-entry rule management
    ├── AutoEntry (model)        Entry trigger rules
    ├── KillSwitchService        Emergency stop management
    ├── KillSwitch (model)       Kill switch state
    ├── PieStatusService         Aggregated PIE status
    ├── ActionOrchestrator       Executes action plans
    └── RiskMonitorService       Continuous rule evaluation (calls ActionOrchestrator)

PIE Events (published to Redis)
    ├── pie.strategy.activated
    ├── pie.strategy.stopped
    ├── pie.entry.triggered
    ├── pie.entry.executed
    ├── pie.kill_switch.triggered
    └── pie.kill_switch.reset
```

---

## 3. Strategy System

### Model

```python
class ActiveStrategy(Base):
    id: UUID
    user_id: UUID
    name: str
    description: str
    config: dict                # JSON: entry rules, exit rules, position sizing
    status: str                 # "pending" | "active" | "paused" | "stopped"
    started_at: datetime
    stopped_at: datetime | None
    stop_reason: str | None
```

### Strategy Config Schema

```json
{
  "entry_rules": [
    {
      "condition": "price_crosses_above",
      "params": { "symbol": "NSE:NIFTY50-IDX", "level": 22500 },
      "action": "place_order",
      "order_params": {
        "symbol": "NSE:NIFTY2540022500CE",
        "side": "BUY",
        "qty": 1,
        "order_type": "MARKET",
        "product_type": "INTRADAY"
      }
    }
  ],
  "exit_rules": [
    {
      "condition": "pnl_pct",
      "params": { "target_pct": 50, "stop_loss_pct": -25 },
      "action": "close_position"
    },
    {
      "condition": "time_based",
      "params": { "exit_at": "15:15" },
      "action": "close_position"
    }
  ],
  "position_sizing": {
    "type": "fixed_qty",
    "qty": 1
  }
}
```

### Strategy Lifecycle

```
POST /api/v1/strategies          → Create strategy (status=pending)
POST /api/v1/strategies/{id}/activate  → status=active, published to event bus
                                         RiskMonitorService starts evaluating entry rules
PIE loop evaluates entry rules
    → condition met → AutoEntryService.trigger_entry()
    → OrderHandler.place_order()
    → EventBus.publish(pie.entry.executed)

POST /api/v1/strategies/{id}/stop  → status=stopped, cleanup
```

---

## 4. Auto-Entry System

### Model

```python
class AutoEntry(Base):
    id: UUID
    user_id: UUID
    name: str
    instrument: str             # e.g., "NSE:RELIANCE-EQ"
    trigger_config: dict        # JSON: trigger conditions
    order_template: dict        # JSON: order parameters to execute
    is_active: bool
    max_entries: int            # 0 = unlimited
    entry_count: int            # times triggered so far
    cooldown_seconds: int       # min seconds between triggers
    last_triggered_at: datetime | None
```

### Trigger Config Types

```json
{ "type": "price_alert", "condition": "crosses_above", "price": 2850 }
{ "type": "price_alert", "condition": "crosses_below", "price": 2800 }
{ "type": "time_trigger", "time": "09:20" }
{ "type": "pnl_trigger", "pnl_change": -1000 }
```

### Auto-Entry Flow

```
RiskMonitorService polling loop
    → AutoEntryService.evaluate_pending(user_id)
    → For each active AutoEntry:
        → QuoteStore.get_ltp(entry.instrument)
        → EvaluateTriggerCondition(ltp, entry.trigger_config)
        → If triggered AND cooldown elapsed AND entry_count < max_entries:
            → OrderHandler.place_order(entry.order_template)
            → AutoEntry.entry_count += 1
            → AutoEntry.last_triggered_at = now()
            → ActionLog.record(AUTO_ENTRY_TRIGGERED)
            → EventBus.publish(pie.entry.triggered)
```

---

## 5. Kill Switch

### Model

```python
class KillSwitch(Base):
    id: UUID
    user_id: UUID
    is_active: bool             # True = triggered, blocking new orders
    triggered_at: datetime | None
    reset_at: datetime | None
    trigger_reason: str | None  # "manual" | "daily_loss_limit" | "risk_breach"
```

### Kill Switch Activation

```python
class KillSwitchService:
    async def trigger(self, user_id: UUID, reason: str):
        async with transaction():
            # 1. Set kill switch active
            await self.repo.activate(user_id, reason)

            # 2. Cancel all open orders
            open_orders = await self.order_handler.get_open_orders(user_id)
            for order in open_orders:
                await self.order_handler.cancel(order.id)

            # 3. Close all open positions at market
            positions = await self.position_repo.get_open_positions(user_id)
            for position in positions:
                await self.order_handler.place_order(
                    symbol=position.symbol,
                    side=OPPOSITE(position.side),
                    qty=position.net_qty,
                    order_type="MARKET"
                )

            # 4. Log + notify
            await self.action_log.record(KILL_SWITCH_TRIGGERED, user_id, reason)
            await self.event_bus.publish("pie.kill_switch.triggered", {...})

    async def reset(self, user_id: UUID):
        await self.repo.deactivate(user_id)
        await self.event_bus.publish("pie.kill_switch.reset", {...})
```

### AccountStateGuard Integration

When kill switch is active, `AccountStateGuard.assert_can_trade()` raises an error, blocking ALL new order placement until reset.

---

## 6. PIE Status

`PieStatusService` aggregates real-time state for the frontend PIE dashboard:

```python
class PieStatus(BaseModel):
    user_id: UUID
    kill_switch_active: bool
    kill_switch_reason: str | None
    active_strategies: list[StrategyStatus]
    active_auto_entries: list[AutoEntryStatus]
    recent_actions: list[ActionLogEntry]    # last 20 PIE actions
    risk_snapshot: RiskSnapshot
```

Endpoints:
- `GET /api/v1/pie/status` — Full PIE status
- `WS /ws/pie/{user_id}` — Real-time PIE status push (strategy activations, triggers, alerts)

---

## 7. Action Log

All PIE actions are written to an immutable append-only log:

```python
class ActionLog(Base):
    id: UUID
    user_id: UUID
    action_type: str            # "KILL_SWITCH_TRIGGERED" | "AUTO_ENTRY_TRIGGERED" | ...
    payload: dict               # JSON: full context of the action
    source: str                 # "manual" | "risk_engine" | "strategy" | "auto_entry"
    created_at: datetime        # immutable — no UPDATE ever
```

Queried for:
- PIE dashboard "Monitor Log" panel
- Compliance/audit trail
- Debugging PIE behavior

---

## 8. Frontend Integration

**PIE Dashboard components:**

| Component | Data Source |
|-----------|-------------|
| `PIEDashboard.tsx` | `GET /api/v1/pie/status` |
| `StrategyPanel.tsx` | `GET /api/v1/strategies`, activate/stop |
| `AutoEntryPanel.tsx` | `GET /api/v1/auto_entry`, create/delete/toggle |
| `KillSwitchButton.tsx` | `POST /api/v1/kill_switch/trigger`, `/reset` |
| `MonitorLog.tsx` | `GET /api/v1/actions`, WebSocket push |

**Zustand Store:** `pieStore.ts` holds strategy state, kill switch state, recent actions.

**WebSocket Subscription:** `usePIESubscription.ts` subscribes to `pie.*` events and updates store in real-time.

---

## 9. PIE Rule Evaluators (unit tested)

The rule evaluation logic (trigger conditions) is in separate evaluator functions, tested independently of the service layer:

```python
# test_pie_rule_evaluators.py
def test_crosses_above_rule():
    rule = {"type": "price_alert", "condition": "crosses_above", "price": 2850}
    assert evaluate_trigger(rule, prev_ltp=2840, curr_ltp=2855) == True
    assert evaluate_trigger(rule, prev_ltp=2840, curr_ltp=2845) == False
    assert evaluate_trigger(rule, prev_ltp=2860, curr_ltp=2870) == False  # already above
```

---

## 10. Test Coverage

| Test | Scope |
|------|-------|
| `test_strategy_service.py` | Strategy CRUD, activation, state transitions |
| `test_auto_entry_service.py` | Trigger evaluation, cooldown, max_entries |
| `test_kill_switch_service.py` | Activation, order cancellation, position close |
| `test_action_log_service.py` | Append-only writes, query |
| `test_pie_rule_evaluators.py` | Trigger condition logic (crosses, time, P&L) |
| `test_action_orchestrator_pie.py` | Multi-action plan execution |
| `test_pie_strategy_lifecycle.py` | Integration: create → activate → trigger → stop |
| `test_pie_kill_switch.py` | Integration: trigger → cancel orders → close positions |
| `test_pie_rule_to_exit.py` | Integration: rule breach → exit strategy |
