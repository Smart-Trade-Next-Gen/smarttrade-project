# BAS — Order Lifecycle Design

**Version:** 1.0
**Component:** `broker_adapter_service/services/order_handler.py`

---

## 1. Order States

```
PENDING → OPEN → FILLED → SETTLED
               ↓
           PARTIALLY_FILLED → FILLED
               ↓
           CANCELLED
               ↓
           REJECTED
```

| State | Description |
|-------|-------------|
| `PENDING` | Created locally, not yet sent to broker |
| `OPEN` | Sent to broker, awaiting fill |
| `PARTIALLY_FILLED` | Some quantity filled, rest pending |
| `FILLED` | Fully filled |
| `CANCELLED` | Cancelled before full fill |
| `REJECTED` | Broker rejected the order |
| `SETTLED` | T+1 settlement complete |

---

## 2. Order Model

```python
class Order(Base):
    id: UUID                        # SmartTrade internal ID
    user_id: UUID                   # owner — mandatory for all queries
    account_id: UUID                # FK → TradingAccount
    symbol: str                     # normalized SmartTrade symbol
    exchange: str
    side: OrderSide                 # BUY | SELL
    order_type: OrderType           # MARKET | LIMIT | SL | SL-M
    product_type: ProductType       # INTRADAY | DELIVERY | MTF
    qty: int
    price: Decimal | None           # None for MARKET orders
    trigger_price: Decimal | None   # for SL/SL-M
    status: OrderStatus
    broker_order_id: str | None     # broker's order ID (after submission)
    filled_qty: int
    avg_fill_price: Decimal | None
    rejection_reason: str | None
    idempotency_key: str            # SHA-256 of order params + client nonce
    created_at: datetime
    updated_at: datetime
```

---

## 3. Placement Flow

```python
class OrderHandler:
    async def place_order(self, user_id: UUID, request: PlaceOrderRequest) -> Order:
        # Step 1: Pre-conditions
        await self.account_state_guard.assert_can_trade(user_id)

        # Step 2: Idempotency check (Redis)
        existing = await self.idempotency.get(request.idempotency_key)
        if existing:
            return existing  # Return previous result, don't re-place

        # Step 3: Pre-trade risk check
        snapshot = await self.risk_snapshot_builder.build(user_id, pending_order=request)
        risk_results = await self.risk_engine.evaluate(snapshot)
        self._assert_no_blocking_risk(risk_results)

        # Step 4: Map to broker DTO
        plugin = self.adapter_manager.get_plugin(user_id, request.broker_id)
        broker_order = plugin.mapper.to_broker_order(request)

        # Step 5: Persist (PENDING) + submit to broker
        async with transaction():
            order = await self.order_repo.create(user_id, request, status=PENDING)
            broker_response = await plugin.place_order(broker_order)
            order.broker_order_id = broker_response.order_id
            order.status = OPEN
            await self.order_repo.update(order)

        # Step 6: Register idempotency key
        await self.idempotency.set(request.idempotency_key, order)

        # Step 7: Audit + events
        await self.action_log.record(ORDER_PLACED, user_id, order)
        await self.event_bus.publish("order.placed", order_to_event(order))

        return order
```

---

## 4. Order Updates (Fill Notifications)

Fill notifications arrive from the broker via WebSocket:

```python
# In event_consumers.py or ws/adapter_websocket_client.py
async def on_fill_event(fill: BrokerFillEvent):
    order = await self.order_repo.get_by_broker_id(fill.broker_order_id)
    async with transaction():
        order.filled_qty += fill.fill_qty
        order.avg_fill_price = recalculate_avg(order, fill)
        if order.filled_qty >= order.qty:
            order.status = FILLED
        else:
            order.status = PARTIALLY_FILLED
        await self.order_repo.update(order)

        # Create trade record (immutable)
        trade = Trade(
            order_id=order.id,
            user_id=order.user_id,
            fill_price=fill.price,
            fill_qty=fill.fill_qty,
            timestamp=fill.timestamp
        )
        await self.trade_repo.create(trade)

        # Update position
        await self.position_graph_engine.on_fill(order, trade)

    await self.event_bus.publish("order.filled", ...)
    await self.event_bus.publish("trade.executed", ...)
```

---

## 5. PositionGraphEngine

Tracks positions at fill granularity using a graph model:

```
Positions are nodes; fills are edges connecting BUY fills to SELL fills.
Each SELL fill consumes BUY fill(s) using FIFO to compute realized P&L.
```

```python
class PositionGraphEngine:
    async def on_fill(self, order: Order, trade: Trade):
        if order.side == BUY:
            await self.add_buy_node(order.user_id, order.symbol, trade)
        else:
            # FIFO match against buy fills
            buy_fills = await self.get_open_buy_fills(order.user_id, order.symbol)
            realized_pnl = Decimal("0")
            remaining_sell = trade.fill_qty
            for buy_fill in buy_fills:
                matched_qty = min(buy_fill.remaining_qty, remaining_sell)
                realized_pnl += (trade.fill_price - buy_fill.price) * matched_qty
                buy_fill.remaining_qty -= matched_qty
                remaining_sell -= matched_qty
                if remaining_sell == 0:
                    break
            await self.update_position_snapshot(order.user_id, order.symbol)
```

**Why graph model?**
- Supports partial fills correctly
- Enables per-lot cost basis tracking
- Handles complex scenarios: add to position, partial exit, position reversal

---

## 6. Cancellation

```python
async def cancel_order(self, user_id: UUID, order_id: UUID):
    order = await self.order_repo.get(order_id, user_id=user_id)
    self._assert_cancellable(order)  # must be OPEN or PENDING

    plugin = self.adapter_manager.get_plugin(user_id, order.broker_id)
    await plugin.cancel_order(order.broker_order_id)

    order.status = CANCELLED
    await self.order_repo.update(order)
    await self.event_bus.publish("order.cancelled", ...)
```

---

## 7. Broker Plugin Interface

```python
class BrokerPlugin(ABC):
    @abstractmethod
    async def place_order(self, order: BrokerOrderDTO) -> BrokerOrderResponse
        # Returns: broker_order_id, initial_status

    @abstractmethod
    async def cancel_order(self, broker_order_id: str) -> None

    @abstractmethod
    async def modify_order(self, broker_order_id: str, modifications: dict) -> None

    @abstractmethod
    async def get_order_status(self, broker_order_id: str) -> BrokerOrderStatus

    @abstractmethod
    async def get_positions(self) -> list[BrokerPosition]

    @abstractmethod
    async def get_funds(self) -> BrokerFunds
```

**Fyers plugin:** Maps SmartTrade DTOs ↔ Fyers API v3 format via `dto_to_fyers_mapper.py` / `fyers_to_dto_mapper.py`.

**Paper plugin:** Routes to Mock Service via `mock_service_client.py` HTTP calls.

---

## 8. Idempotency

Order placement is idempotent. Client passes an `idempotency_key`:

```
Key format: SHA-256(user_id + symbol + side + qty + price + client_nonce)
TTL: 24 hours (Redis)
```

If the same key arrives twice:
1. Check Redis: found → return cached order response
2. Not found → proceed with placement, store result

This prevents duplicate orders on client retry (network timeout, etc.)

---

## 9. Order API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/orders` | Place new order |
| GET | `/api/v1/orders` | List orders (paginated, filterable) |
| GET | `/api/v1/orders/{id}` | Single order detail |
| PUT | `/api/v1/orders/{id}` | Modify open order (price/qty) |
| DELETE | `/api/v1/orders/{id}` | Cancel order |
| GET | `/api/v1/trades` | List executed trades |
| GET | `/api/v1/trades/{id}` | Trade detail |

---

## 10. Test Coverage

| Test | Scope |
|------|-------|
| `test_order_handler.py` | Place, cancel, modify; state transitions |
| `test_order_handler_financial.py` | Decimal correctness, avg price calc |
| `test_order_schemas.py` | Pydantic validation: required fields, enums |
| `test_order_routes.py` | HTTP: POST/GET/PUT/DELETE endpoints |
| `test_e2e_trading_workflow.py` | Integration: full order → fill → position |
| `test_order_settlement_workflow.py` | Integration: fill → trade → settlement T+1 |
