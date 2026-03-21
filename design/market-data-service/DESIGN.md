# Market Data Service — Design Document

**Version:** 1.0
**Port:** 8004
**Database:** `smarttrade_market_data_service`

---

## 1. Responsibility

Single source of truth for all market data within SmartTrade. Manages instrument master data, real-time quotes via WebSocket fan-out, historical OHLCV candles, options chains, Greeks, and IV metrics.

**In scope:**
- Instrument master (NSE/BSE/NFO symbol database)
- Broker-symbol mapping (SmartTrade symbol ↔ broker-specific token)
- Real-time quote streaming (WebSocket fan-out)
- Historical OHLCV candle data
- Option chain snapshots (scheduled refresh)
- Options Greeks (Black-Scholes: delta, gamma, theta, vega)
- IV metrics (IV rank, IV percentile)
- Volatility surface modeling
- Trading calendar (exchange holidays + session times)
- Margin requirements

**Out of scope:**
- Order execution (BAS)
- Position tracking (BAS)
- User-specific portfolio data

---

## 2. Plugin Architecture

MDS uses the same plugin pattern as BAS — broker-specific logic is encapsulated in plugins that implement a common interface.

```python
class BaseMDSPlugin(ABC):
    @abstractmethod
    async def get_historical_candles(symbol, resolution, from_dt, to_dt) -> list[Candle]

    @abstractmethod
    async def get_option_chain(underlying, expiry) -> OptionChain

    @abstractmethod
    async def get_margin(symbols) -> list[MarginInfo]

    @abstractmethod
    async def create_websocket(on_message_callback) -> BaseWebSocket
```

### Registered Plugins

| Plugin | Broker | WebSocket |
|--------|--------|-----------|
| `fyers/plugin.py` | Fyers (live) | Fyers Data Socket + Order Socket |
| `paper/paper_plugin.py` | Paper/Mock | Random walk simulator |

---

## 3. Data Models

### Instrument

```python
class Instrument(Base):
    id: UUID
    symbol: str                 # e.g., "RELIANCE", "NIFTY50"
    exchange: str               # "NSE" | "BSE" | "NFO" | "MCX"
    instrument_type: str        # "EQ" | "FUT" | "CE" | "PE" | "IDX"
    lot_size: int               # 1 for equities
    tick_size: Decimal
    expiry: date | None         # for derivatives
    strike: Decimal | None      # for options
    underlying_symbol: str | None
```

### BrokerInstrument

```python
class BrokerInstrument(Base):
    id: UUID
    instrument_id: UUID         # FK → Instrument
    broker_id: str              # "fyers" | "zerodha" | "paper"
    broker_symbol: str          # e.g., "NSE:RELIANCE-EQ" (Fyers format)
    broker_token: str           # numeric token for WebSocket subscription
```

### HistoricalCandle

```python
class HistoricalCandle(Base):
    id: UUID
    instrument_id: UUID
    resolution: str             # "1", "5", "15", "30", "60", "D", "W"
    open: Decimal
    high: Decimal
    low: Decimal
    close: Decimal
    volume: int
    timestamp: datetime         # UTC, start of candle
```

### ExchangeHoliday

```python
class ExchangeHoliday(Base):
    exchange: str
    date: date
    description: str
```

---

## 4. API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/instruments` | Search instruments by symbol/name |
| GET | `/instruments/{id}` | Instrument detail |
| GET | `/broker_instruments/{broker_id}` | Broker-mapped symbols |
| GET | `/data/historical` | OHLCV candles (symbol, resolution, from, to) |
| GET | `/margin` | Margin requirements for symbols |
| GET | `/option_chain` | Option chain snapshot (underlying, expiry) |
| GET | `/option_chain/expiries` | Available expiry dates |
| GET | `/greeks` | Greeks for option symbol |
| GET | `/iv_metrics` | IV rank + IV percentile |
| GET | `/iv_metrics/history` | IV history for charting |
| GET | `/notifications` | User notifications (market alerts) |
| WS | `/ws/{broker_id}/{user_id}` | Real-time market data stream |

---

## 5. WebSocket Architecture

### Connection Flow

```
Client CONNECT /ws/{broker_id}/{user_id}
    → JWT validation
    → WsManager.register(ws, user_id, broker_id)
    → BrokerScopedManager.get_or_create(broker_id)

Client SEND { type: "subscribe", symbols: ["NSE:RELIANCE-EQ"] }
    → OptionSubscriptionHandler.handle(message)
    → BrokerScopedManager.subscribe(broker_id, symbols)
    → Plugin WebSocket subscription sent to Fyers

Fyers WebSocket → tick data
    → InstrumentResolverManager.resolve(broker_token)
    → BrokerScopedManager.on_tick(tick)
    → WsManager.broadcast(tick, subscribed_clients)
```

### Key Components

**WsManager** — Registry of all active WebSocket connections. Handles broadcast, subscribe, unsubscribe, and connection cleanup on disconnect.

**BrokerScopedManager** — One instance per active broker session. Owns the plugin WebSocket connection and tracks which symbols are subscribed. Multiple users sharing the same broker session share a single upstream WebSocket connection.

**InstrumentResolverManager** — Async queue that resolves broker tokens (numeric) to instrument records. Uses a worker pattern with a `PendingEventBuffer` that holds ticks for unresolved instruments until resolution completes.

**OptionEventBroadcaster** — Specialized broadcaster for option chain data; sends structured option tick messages with Greeks attached.

### WebSocket Message Protocol

**Subscribe:**
```json
{ "type": "subscribe", "symbols": ["NSE:RELIANCE-EQ", "NSE:NIFTY2540019000CE"] }
```

**Tick (outbound):**
```json
{
  "type": "tick",
  "symbol": "NSE:RELIANCE-EQ",
  "ltp": "2847.50",
  "open": "2820.00",
  "high": "2865.00",
  "low": "2810.00",
  "volume": 1234567,
  "timestamp": "2026-03-21T09:30:00Z"
}
```

**Option Tick (outbound):**
```json
{
  "type": "option_tick",
  "symbol": "NSE:NIFTY2540019000CE",
  "ltp": "145.50",
  "iv": "0.1823",
  "delta": "0.42",
  "gamma": "0.0021",
  "theta": "-18.45",
  "vega": "34.12",
  "open_interest": 2340000,
  "timestamp": "2026-03-21T09:30:00Z"
}
```

---

## 6. Options System

### Option Chain Cache

Option chain snapshots are cached in Redis to avoid repeated expensive API calls to Fyers.

- TTL: configurable per underlying (default 30s during market hours)
- Cache key: `option_chain:{broker_id}:{underlying}:{expiry}`
- Cache miss: fetch from Fyers API → store in Redis + PostgreSQL

### Option Chain Refresh Scheduler

A background task that schedules periodic option chain refreshes:
- All ATM ± 10 strikes for active underlyings
- Refresh interval: 30s (market hours), 5min (pre-market)
- `OptionSchedulerManager` tracks active underlyings based on subscriptions

### Greeks Calculator (Black-Scholes)

Computes theoretical values for European options:

```python
class GreeksCalculator:
    def calculate(S, K, T, r, sigma, option_type) -> Greeks:
        # S: spot price
        # K: strike price
        # T: time to expiry (years)
        # r: risk-free rate (India: 6.5%)
        # sigma: implied volatility (solved from market price)
        # Returns: delta, gamma, theta, vega, rho, iv
```

IV is solved via Newton-Raphson iteration from market LTP.

### Volatility Surface

Fits a parametric surface to observed IVs across strikes and expiries:
- SVI (Stochastic Volatility Inspired) parameterization
- Used for IV interpolation at arbitrary strikes
- Rebuilt on each option chain refresh

### IV Metrics

| Metric | Definition |
|--------|-----------|
| IV Rank | `(current_iv - 52w_low_iv) / (52w_high_iv - 52w_low_iv) × 100` |
| IV Percentile | % of trading days in past year where IV was lower than current |

Historical IV stored in `iv_metrics` table for lookback queries.

---

## 7. Instrument Resolution

Fyers WebSocket sends numeric tokens (e.g., `1594`) not symbols. Resolution is async:

```
WebSocket tick arrives { token: "1594", ltp: 2847.50 }
    ↓
InstrumentResolutionQueue.enqueue(token)
    ↓ (async)
InstrumentResolverWorker.resolve(token)
    → BrokerInstrumentRepository.get_by_token("1594", broker_id="fyers")
    → Returns BrokerInstrument(broker_symbol="NSE:RELIANCE-EQ")
    ↓
PendingEventBuffer.flush(token) → broadcast buffered ticks
```

---

## 8. Trading Calendar

```python
class TradingCalendarResolver:
    def is_trading_day(exchange, date) -> bool
    def get_session_times(exchange) -> SessionTimes
    def next_trading_day(exchange, date) -> date
    def trading_days_between(exchange, from_date, to_date) -> int
```

Used by:
- `OptionChainRefreshScheduler` (only refresh during market hours)
- `SettlementService` (T+1 calculation skips holidays)
- Frontend `TradingTime` utility

---

## 9. Service → Service Clients

MDS exposes HTTP clients used by BAS:

**BAS → MDS** (via `market_data_client.py`):
- `resolve_symbol(broker_id, symbol)` → instrument_id
- `get_ltp(symbols)` → latest prices for risk estimation
- `get_margin(symbols)` → margin requirements

**MDS → BAS** (via `broker_adapter_client.py`):
- `get_active_sessions()` → which users have active broker sessions (for WebSocket management)

---

## 10. Test Coverage

| Test File | Scope |
|-----------|-------|
| `test_greeks_calculator.py` | BS formula correctness, edge cases (ATM, deep ITM/OTM) |
| `test_volatility_surface.py` | SVI surface fit, interpolation |
| `test_iv_metrics_calculator.py` | IV rank/percentile computation |
| `test_option_chain_cache.py` | Cache hit/miss/TTL |
| `test_option_chain_fetcher.py` | Fyers option chain parsing |
| `test_instrument_resolver.py` | Token → symbol resolution, buffering |
| `test_broker_instrument_cache.py` | Instrument cache invalidation |
| `test_trading_calendar.py` | Holiday detection, session times |
| `test_ws_manager.py` | Connection lifecycle, broadcast |
| `test_greeks_api.py` | HTTP endpoint integration |
| `test_option_chain_api.py` | Option chain API integration |
| `test_option_websocket.py` | WebSocket subscription + tick fan-out |
