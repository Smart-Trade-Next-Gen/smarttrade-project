# BAS — Risk Engine Design

**Version:** 1.0
**Component:** `broker_adapter_service/risk_engine/`

---

## 1. Overview

The Risk Engine enforces user-defined trading limits before, during, and after order execution. It operates in three modes:

1. **Pre-trade** — Block orders that would breach limits (synchronous, on order placement)
2. **Monitor** — Continuous background evaluation, trigger actions on breach
3. **Estimate** — Compute hypothetical risk for UI display (non-blocking)

All rules are config-driven (YAML). Adding a new rule requires implementing a single function + adding a YAML entry.

---

## 2. Architecture

```
RiskMonitorService          # Background polling loop
    ↓ (every N seconds)
RiskSnapshotBuilder         # Gathers current state
    ↓
RuntimeRiskInput            # Frozen dataclass: positions, P&L, open orders
    ↓
RiskEngine.evaluate()       # Applies all active rules
    ↓
[RuleResult(passed=False)]
    ↓
ActionOrchestrator          # Executes configured actions
    ├── kill_switch.trigger()
    ├── order_handler.cancel_all()
    └── EventBus.publish("risk.breach")
```

---

## 3. Rule System

### Config (risk_config.yaml)

```yaml
rules:
  - id: max_daily_loss
    enabled: true
    class: broker_adapter_service.risk_engine.rules.max_daily_loss.MaxDailyLossRule
    params:
      default_limit: -5000.00
    on_breach:
      - action: trigger_kill_switch
      - action: publish_alert
        params:
          severity: critical

  - id: max_open_positions
    enabled: true
    class: broker_adapter_service.risk_engine.rules.max_open_positions.MaxOpenPositionsRule
    params:
      default_limit: 10
    on_breach:
      - action: block_new_orders
      - action: publish_alert
        params:
          severity: warning

  - id: max_risk_per_trade
    enabled: true
    class: broker_adapter_service.risk_engine.rules.max_risk_per_trade.MaxRiskPerTradeRule
    params:
      default_pct: 2.0          # % of account capital
    on_breach:
      - action: reject_order
```

### RuntimeRiskInput (frozen dataclass)

```python
@dataclass(frozen=True)
class RuntimeRiskInput:
    user_id: UUID
    realized_pnl: Decimal       # today's closed P&L
    unrealized_pnl: Decimal     # open positions MTM
    open_position_count: int
    account_capital: Decimal    # available + used margin
    pending_order: OrderDTO | None   # None for monitor mode
    positions: list[PositionSnapshot]
    evaluated_at: datetime
```

Note: frozen=True enforces immutability. Tests must create new instances rather than mutating fixtures.

### Rule Interface

```python
class BaseRiskRule(ABC):
    rule_id: str
    params: dict

    @abstractmethod
    def evaluate(self, input: RuntimeRiskInput) -> RuleResult:
        ...

class RuleResult:
    passed: bool
    rule_id: str
    message: str
    current_value: Decimal | int
    limit_value: Decimal | int
    severity: str               # "info" | "warning" | "critical"
```

### Implemented Rules

#### MaxDailyLossRule

```python
def evaluate(self, input: RuntimeRiskInput) -> RuleResult:
    total_pnl = input.realized_pnl + input.unrealized_pnl
    limit = self.get_user_limit(input.user_id) or self.params["default_limit"]
    return RuleResult(
        passed=total_pnl >= limit,
        current_value=total_pnl,
        limit_value=Decimal(str(limit))
    )
```

#### MaxOpenPositionsRule

```python
def evaluate(self, input: RuntimeRiskInput) -> RuleResult:
    limit = self.get_user_limit(input.user_id) or self.params["default_limit"]
    return RuleResult(
        passed=input.open_position_count < limit,
        current_value=input.open_position_count,
        limit_value=limit
    )
```

#### MaxRiskPerTradeRule

```python
def evaluate(self, input: RuntimeRiskInput) -> RuleResult:
    # Only evaluated in pre-trade mode (pending_order != None)
    if input.pending_order is None:
        return RuleResult(passed=True, ...)
    risk_amount = (pending_order.qty * pending_order.stop_loss_distance)
    pct = (risk_amount / input.account_capital) * Decimal("100")
    limit_pct = Decimal(str(self.params["default_pct"]))
    return RuleResult(passed=pct <= limit_pct, ...)
```

---

## 4. RiskSnapshotBuilder

Assembles the `RuntimeRiskInput` from live data sources:

```python
class RiskSnapshotBuilder:
    async def build(self, user_id: UUID, pending_order=None) -> RuntimeRiskInput:
        positions = await self.position_repo.get_open_positions(user_id)
        realized_pnl = await self.settlement_repo.get_today_realized_pnl(user_id)
        unrealized_pnl = sum(p.unrealized_pnl for p in positions)
        account = await self.account_repo.get_active_account(user_id)
        capital = account.available_margin + account.used_margin
        return RuntimeRiskInput(
            user_id=user_id,
            realized_pnl=realized_pnl,
            unrealized_pnl=unrealized_pnl,
            open_position_count=len(positions),
            account_capital=capital,
            pending_order=pending_order,
            positions=positions,
            evaluated_at=datetime.utcnow()
        )
```

---

## 5. RiskMonitorService

Background task that runs continuously during market hours:

```python
class RiskMonitorService:
    poll_interval: int = 30     # seconds

    async def run(self):
        while self.running:
            for user_id in self.active_user_ids():
                snapshot = await self.snapshot_builder.build(user_id)
                results = await self.engine.evaluate(snapshot)
                breaches = [r for r in results if not r.passed]
                if breaches:
                    await self.action_orchestrator.execute(user_id, breaches)
            await asyncio.sleep(self.poll_interval)
```

Only evaluates users with active broker sessions (avoids unnecessary DB load).

---

## 6. ActionOrchestrator

Maps rule breach actions from YAML config to concrete service calls:

```python
class ActionOrchestrator:
    async def execute(self, user_id: UUID, breaches: list[RuleResult]):
        for breach in breaches:
            rule_config = self.config.get_rule(breach.rule_id)
            for action in rule_config.on_breach:
                await self._dispatch(action.action, user_id, breach, action.params)

    async def _dispatch(self, action_name, user_id, breach, params):
        match action_name:
            case "trigger_kill_switch":
                await self.kill_switch_service.trigger(user_id, reason=breach.message)
            case "block_new_orders":
                await self.account_state_guard.set_block(user_id)
            case "reject_order":
                raise RiskBreachError(breach.message)
            case "publish_alert":
                await self.event_bus.publish("risk.alert", {...})
            case "cancel_all":
                await self.order_handler.cancel_all_open(user_id)
```

---

## 7. Pre-Trade Risk Check

Called synchronously during order placement:

```python
# In OrderHandler.place_order():
snapshot = await self.snapshot_builder.build(user_id, pending_order=order)
results = await self.risk_engine.evaluate(snapshot)
for result in results:
    if not result.passed and result.rule_id == "max_risk_per_trade":
        raise SmartTradeError("RISK_001", f"Order rejected: {result.message}")
    if not result.passed and result.rule_id == "max_open_positions":
        raise SmartTradeError("RISK_002", f"Order rejected: {result.message}")
```

---

## 8. Risk API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/risk/snapshot` | Current risk state for user |
| GET | `/api/v1/risk/settings` | User's risk limits |
| PUT | `/api/v1/risk/settings` | Update risk limits |
| GET | `/api/v1/risk/estimate` | Pre-trade risk estimate (query params) |
| GET | `/api/v1/risk/history` | Historical breach log |

---

## 9. Financial Correctness

All monetary calculations use `Decimal`:

```python
# Correct:
total_pnl = input.realized_pnl + input.unrealized_pnl
pct = (risk_amount / input.account_capital) * Decimal("100")

# NEVER:
total_pnl = float(realized) + float(unrealized)  # FORBIDDEN
pct = risk_amount / account_capital * 100.0       # FORBIDDEN
```

The `Decimal("100")` (not `100.0`) is critical — Decimal division with float coercion is undefined behavior in trading systems.

---

## 10. Test Coverage

| Test | Scope |
|------|-------|
| `test_risk_engine_unit.py` | Rule evaluation logic, frozen input immutability |
| `test_risk_orchestrator.py` | Action dispatch for each breach type |
| `test_exposure.py` | Exposure classification by segment/product |
| `test_risk_routes.py` | HTTP: snapshot, settings GET/PUT |
