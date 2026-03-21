# How to: Add a New Broker Integration

This guide walks through integrating a new broker (e.g., Zerodha) into SmartTrade. Both BAS (order execution) and MDS (market data) have plugin systems that need to be implemented.

---

## Overview

Adding a broker requires:
1. **BAS plugin** — order placement, cancellation, position sync
2. **MDS plugin** — historical data, option chains, real-time WebSocket
3. **OAuth flow** — broker authentication + session management
4. **Instrument mapping** — broker symbol ↔ SmartTrade symbol
5. **Tests** — unit + integration coverage

Estimated effort: 2–3 weeks for a full broker integration.

---

## Step 1: Create BAS Plugin

### File Structure

```
broker-adapter-service/src/broker_adapter_service/plugins/zerodha/
    __init__.py
    plugin.py               # ZerodhaBrokerPlugin (implements BrokerPlugin)
    dto_to_zerodha_mapper.py
    zerodha_to_dto_mapper.py
```

### Implement BrokerPlugin

```python
# plugins/zerodha/plugin.py
from broker_adapter_service.plugins.base import BrokerPlugin
from smarttrade_common.schemas.services.trading.order_dtos import (
    PlaceOrderRequest, OrderResponse, BrokerFunds
)

class ZerodhaBrokerPlugin(BrokerPlugin):
    broker_id = "zerodha"

    def __init__(self, api_key: str, access_token: str):
        self.kite = KiteConnect(api_key=api_key)
        self.kite.set_access_token(access_token)

    async def place_order(self, order: BrokerOrderDTO) -> BrokerOrderResponse:
        zerodha_params = DtoToZerodhaMapper.map(order)
        response = await asyncio.to_thread(
            self.kite.place_order,
            variety=zerodha_params.variety,
            **zerodha_params.dict()
        )
        return BrokerOrderResponse(
            broker_order_id=str(response["order_id"]),
            status="OPEN"
        )

    async def cancel_order(self, broker_order_id: str) -> None:
        await asyncio.to_thread(
            self.kite.cancel_order,
            variety="regular",
            order_id=broker_order_id
        )

    async def get_positions(self) -> list[BrokerPosition]:
        positions = await asyncio.to_thread(self.kite.positions)
        return [ZerodhaToDTO.map_position(p) for p in positions["net"]]

    async def get_funds(self) -> BrokerFunds:
        margins = await asyncio.to_thread(self.kite.margins)
        equity = margins["equity"]
        return BrokerFunds(
            available=Decimal(str(equity["available"]["cash"])),
            used=Decimal(str(equity["utilised"]["exposure"])),
        )
```

### Register the Plugin

```python
# plugins/plugin_registry.py
from broker_adapter_service.plugins.zerodha.plugin import ZerodhaBrokerPlugin

PLUGIN_REGISTRY = {
    "fyers": FyersBrokerPlugin,
    "paper": PaperBrokerPlugin,
    "zerodha": ZerodhaBrokerPlugin,   # ADD THIS
}
```

---

## Step 2: Create the DTO Mappers

### dto_to_zerodha_mapper.py

Map SmartTrade's normalized order DTO to Kite's API format:

```python
class DtoToZerodhaMapper:
    ORDER_TYPE_MAP = {
        "MARKET": "MARKET",
        "LIMIT": "LIMIT",
        "SL": "SL",
        "SL-M": "SL-M"
    }
    PRODUCT_MAP = {
        "INTRADAY": "MIS",
        "DELIVERY": "CNC",
        "MTF": "NRML"
    }

    @classmethod
    def map(cls, order: PlaceOrderRequest) -> ZerodhaOrderParams:
        return ZerodhaOrderParams(
            tradingsymbol=order.symbol.split(":")[-1],  # "NSE:RELIANCE-EQ" → "RELIANCE-EQ"
            exchange=order.exchange,
            transaction_type=order.side.value,
            order_type=cls.ORDER_TYPE_MAP[order.order_type],
            quantity=order.qty,
            price=float(order.price) if order.price else None,
            trigger_price=float(order.trigger_price) if order.trigger_price else None,
            product=cls.PRODUCT_MAP[order.product_type],
            variety="regular"
        )
```

### zerodha_to_dto_mapper.py

Map Kite API responses back to SmartTrade DTOs:

```python
class ZerodhaToDTO:
    STATUS_MAP = {
        "OPEN": OrderStatus.OPEN,
        "COMPLETE": OrderStatus.FILLED,
        "CANCELLED": OrderStatus.CANCELLED,
        "REJECTED": OrderStatus.REJECTED,
    }

    @classmethod
    def map_order(cls, kite_order: dict) -> OrderResponse:
        return OrderResponse(
            broker_order_id=str(kite_order["order_id"]),
            status=cls.STATUS_MAP.get(kite_order["status"], OrderStatus.OPEN),
            filled_qty=kite_order.get("filled_quantity", 0),
            avg_fill_price=Decimal(str(kite_order.get("average_price", 0))) or None,
        )
```

---

## Step 3: Create MDS Plugin

```
market-data-service/src/market_data_service/plugins/zerodha/
    plugin.py               # ZerodhaMDSPlugin (implements BaseMDSPlugin)
    zerodha_to_dto_mapper.py
    websocket.py            # KiteTicker WebSocket adapter
```

```python
# plugins/zerodha/plugin.py
class ZerodhaMDSPlugin(BaseMDSPlugin):
    broker_id = "zerodha"

    async def get_historical_candles(
        self, symbol: str, resolution: str, from_dt: datetime, to_dt: datetime
    ) -> list[Candle]:
        instrument_token = await self.get_instrument_token(symbol)
        data = await asyncio.to_thread(
            self.kite.historical_data,
            instrument_token=instrument_token,
            from_date=from_dt,
            to_date=to_dt,
            interval=self._resolution_to_kite(resolution)
        )
        return [ZerodhaToDTO.map_candle(c) for c in data]

    async def create_websocket(self, on_message_callback) -> BaseWebSocket:
        return ZerodhaWebSocketAdapter(
            api_key=self.api_key,
            access_token=self.access_token,
            on_tick=on_message_callback
        )
```

---

## Step 4: Add OAuth Flow

### Add Broker Type to Enum

```python
# smarttrade-common/src/smarttrade_common/schemas/types/broker.py
class BrokerType(str, Enum):
    FYERS = "fyers"
    PAPER = "paper"
    ZERODHA = "zerodha"    # ADD THIS
```

### Implement OAuth Handler

```python
# broker-adapter-service/src/broker_adapter_service/api/connection/routes_oauth.py
# Add a Zerodha-specific OAuth handler, or extend the existing one if it's generic

@router.get("/oauth/zerodha/callback")
async def zerodha_oauth_callback(
    request_token: str,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    kite = KiteConnect(api_key=settings.zerodha_api_key)
    data = kite.generate_session(request_token, api_secret=settings.zerodha_api_secret)
    access_token = data["access_token"]

    # Encrypt and store credentials
    encrypted = encrypt(access_token, settings.token_encryption_key)
    await broker_connection_service.save_credentials(
        user_id=current_user.id,
        broker_id="zerodha",
        credentials={"access_token": encrypted}
    )
    return RedirectResponse("/dashboard")
```

---

## Step 5: Instrument Mapping

Zerodha uses numeric instrument tokens. Create a sync script to map them:

```python
# services/instrument_sync_service.py (MDS)
async def sync_zerodha_instruments():
    instruments_csv = await kite.instruments()  # CSV download
    for row in instruments_csv:
        instrument = await instrument_repo.get_or_create_by_symbol(
            symbol=row["tradingsymbol"],
            exchange=row["exchange"],
            instrument_type=row["instrument_type"],
            lot_size=row["lot_size"],
        )
        await broker_instrument_repo.upsert(
            instrument_id=instrument.id,
            broker_id="zerodha",
            broker_symbol=f"{row['exchange']}:{row['tradingsymbol']}",
            broker_token=str(row["instrument_token"])
        )
```

Run this sync daily (or on first startup) to keep broker tokens current.

---

## Step 6: Add Broker Config

```python
# broker-adapter-service/src/broker_adapter_service/config.py
class BASSettings(CommonSettings):
    zerodha_api_key: str = ""
    zerodha_api_secret: str = ""
```

```bash
# .env
ZERODHA_API_KEY=your_api_key
ZERODHA_API_SECRET=your_api_secret
```

---

## Step 7: Write Tests

### BAS Unit Tests

```python
# tests/unit/test_zerodha_mapper.py
def test_limit_order_mapping():
    request = PlaceOrderRequest(
        symbol="NSE:RELIANCE-EQ",
        side="BUY",
        order_type="LIMIT",
        qty=10,
        price=Decimal("2850.00"),
        product_type="DELIVERY"
    )
    params = DtoToZerodhaMapper.map(request)
    assert params.tradingsymbol == "RELIANCE-EQ"
    assert params.order_type == "LIMIT"
    assert params.product == "CNC"
    assert params.price == 2850.0

def test_status_mapping():
    kite_order = {"status": "COMPLETE", "filled_quantity": 10, "average_price": 2855.5}
    dto = ZerodhaToDTO.map_order(kite_order)
    assert dto.status == OrderStatus.FILLED
    assert dto.avg_fill_price == Decimal("2855.5")
```

### Integration Tests

```python
# tests/integration/test_zerodha_plugin.py
@pytest.mark.integration
async def test_place_and_cancel_order(zerodha_sandbox_session):
    plugin = ZerodhaBrokerPlugin(**zerodha_sandbox_session)
    response = await plugin.place_order(sample_limit_order())
    assert response.broker_order_id is not None

    await plugin.cancel_order(response.broker_order_id)
    status = await plugin.get_order_status(response.broker_order_id)
    assert status == OrderStatus.CANCELLED
```

---

## Step 8: Update Frontend

### Add broker to selection UI

```typescript
// contracts/orderEnums.ts
export const BROKER_OPTIONS = [
    { value: 'fyers', label: 'Fyers' },
    { value: 'zerodha', label: 'Zerodha' },  // ADD
    { value: 'paper', label: 'Paper Trading' },
];
```

### Add OAuth redirect handler

```typescript
// pages/OAuthCallback.tsx — handle /oauth/zerodha/callback route
// Store broker connection after successful OAuth
```

---

## Checklist

- [ ] BAS plugin (`plugins/zerodha/plugin.py`) implements all `BrokerPlugin` methods
- [ ] DTO mappers created for both directions
- [ ] MDS plugin created with historical data + WebSocket
- [ ] `BrokerType` enum updated in `smarttrade-common`
- [ ] OAuth flow implemented
- [ ] Instrument sync script written
- [ ] Config variables added
- [ ] Unit tests: DTO mappers (both directions)
- [ ] Unit tests: Plugin methods (mocked HTTP)
- [ ] Integration tests: place/cancel/status with sandbox
- [ ] Frontend: broker selection updated
- [ ] E2E test: full order flow with new broker
