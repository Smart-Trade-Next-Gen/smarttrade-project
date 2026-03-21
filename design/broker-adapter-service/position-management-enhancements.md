# Position Management Enhancements

## Status
APPROVED

## Problem
After an order fills in mock-service, BAS BrokerPosition table is not updated. The mock-service broadcasts position.updated only via WebSocket — BAS has no listener. As a result, BAS position graph and portfolio summaries are stale after fills unless `get_positions()` is called manually (bootstrap only, not continuous).

## Scope

IN:
- mock-service publishes `position.updated` to Redis event bus on every fill
- BAS subscribes to `position.updated` and upserts BrokerPosition table
- BrokerPosition auto-transitions to CLOSED when net_qty reaches 0
- Integration tests covering full order→fill→position→close flow

OUT:
- Fyers broker plugin changes
- Holdings sync (get_holdings already delegates, no gap)
- Real-time WebSocket push from BAS to frontend

## Data Models

### New: PositionUpdatedEvent (mock-service, shared schema)

```python
class PositionUpdatedEvent(BaseModel):
    event_type: str = "position.updated"
    user_id: str
    broker_id: str
    account_id: str
    instrument_id: str
    underlying_symbol: str
    position_type: str          # INTRADAY | OVERNIGHT
    net_qty: int
    avg_price: Decimal
    buy_qty: int
    sell_qty: int
    buy_avg: Optional[Decimal]
    sell_avg: Optional[Decimal]
    realized_pnl: Decimal
    timestamp: datetime
```

### Changed: BrokerPosition (BAS)
- No schema change; upsert_position already handles all fields
- CLOSED transition triggered when net_qty == 0 (new logic in consumer handler)

## Event Flow

```
Order fills in mock-service ExecuteOrderService.try_execute()
  → position_repo.add_or_update_position() (mock-service DB)
  → _publish_events():
      ws_hub.broadcast(position.updated)    [existing - keep]
      event_bus.publish(position.updated, PositionUpdatedEvent)  [NEW]

BAS PositionSyncConsumer (NEW, subscribed at lifespan startup)
  → receives position.updated event
  → idempotency check: skip if event.timestamp <= position.updated_at
  → position_repo.upsert_position(session, broker_id, account_id, user_id, ...)
  → if net_qty == 0: set status=CLOSED, log audit
```

## API Contract

No new endpoints. Existing routes unchanged:
- `GET /api/v1/portfolio/{broker_id}/{account_id}/positions` — lists BrokerPosition rows
- `GET /api/v1/portfolio/{broker_id}/{account_id}/positions/{id}` — single position

## Key Decisions

1. Event bus over HTTP callback: event bus is already wired in both services; avoids tight coupling and retry complexity.
2. PositionUpdatedEvent defined in mock-service (not smarttrade_common): mock-service owns this event; BAS deserializes with its own Pydantic model. Avoids polluting common lib with mock-specific types.
3. Idempotency by timestamp: compare event.timestamp to DB row updated_at; skip if event is older or equal. Prevents duplicate replays from overwriting newer state.
4. CLOSED transition in consumer, not repo: keeps upsert_position generic; consumer applies business rule (net_qty==0 → CLOSED).

## Edge Cases

1. Partial fill then cancel: position net_qty > 0 stays OPEN; only reaches CLOSED when buy_qty == sell_qty.
2. Redis unavailable at mock-service startup: lifespan logs warning, continues; positions still tracked in mock-service DB, get_positions() still works via HTTP polling.
3. BAS consumer fails to upsert (DB error): log error, do NOT re-raise (non-blocking); position will be re-synced on next bootstrap or manual refresh.
4. Duplicate events (Redis at-least-once): idempotency timestamp check prevents double-upsert.
5. User isolation: upsert_position always passes user_id; get_positions always filters by user_id (verified in existing repo).
