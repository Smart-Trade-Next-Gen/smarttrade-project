# SmartTrade API Overview

All APIs use JSON. All authenticated endpoints require `Authorization: Bearer <access_token>` header.

Base URLs:
- Auth: `http://localhost:8001`
- BAS: `http://localhost:8005`
- MDS: `http://localhost:8004`

---

## Authentication Service (Port 8001)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/register` | None | Register new user |
| POST | `/login` | None | Login, get access + refresh tokens |
| POST | `/logout` | Bearer | Revoke refresh token |
| POST | `/refresh` | Refresh token (body) | Rotate refresh token, new access token |
| POST | `/change-password` | Bearer | Update password |
| GET | `/me` | Bearer | Current user profile |
| GET | `/` | None | Liveness probe |
| GET | `/ready` | None | Readiness probe |

### Login Request/Response

```json
POST /login
{ "email": "user@example.com", "password": "Password@123" }

→ 200 OK
{
  "access_token": "eyJ...",
  "refresh_token": "uuid",
  "token_type": "bearer",
  "expires_in": 900
}
```

---

## Broker Adapter Service (Port 8005)

### Orders

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/orders` | Place order |
| GET | `/api/v1/orders` | List orders (filters: status, symbol, date) |
| GET | `/api/v1/orders/{id}` | Order detail |
| PUT | `/api/v1/orders/{id}` | Modify order (price, qty) |
| DELETE | `/api/v1/orders/{id}` | Cancel order |
| GET | `/api/v1/trades` | List trades |
| GET | `/api/v1/trades/{id}` | Trade detail |

**Place Order:**
```json
POST /api/v1/orders
{
  "symbol": "NSE:RELIANCE-EQ",
  "exchange": "NSE",
  "side": "BUY",
  "order_type": "LIMIT",
  "product_type": "INTRADAY",
  "qty": 10,
  "price": "2850.50",
  "idempotency_key": "sha256-hash"
}

→ 201 Created
{
  "id": "uuid",
  "status": "OPEN",
  "broker_order_id": "BR123456",
  "created_at": "2026-03-21T09:30:00Z"
}
```

### Portfolio

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/portfolio` | All open positions |
| GET | `/api/v1/portfolio/summary` | P&L summary by segment |
| GET | `/api/v1/funds` | Available + used margin |
| GET | `/api/v1/settlement` | Settlement history |
| GET | `/api/v1/settlement/today_pnl` | Today's realized P&L |

### Position Groups

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/position_groups` | Create group |
| GET | `/api/v1/position_groups` | List groups |
| GET | `/api/v1/position_groups/{id}` | Group detail |
| PUT | `/api/v1/position_groups/{id}` | Update rules |
| DELETE | `/api/v1/position_groups/{id}` | Archive group |
| POST | `/api/v1/position_groups/{id}/positions` | Add position |
| DELETE | `/api/v1/position_groups/{id}/positions/{pos_id}` | Remove position |
| POST | `/api/v1/position_groups/{id}/exit` | Exit all positions |
| GET | `/api/v1/position_groups/{id}/pnl` | Group P&L |

### Risk

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/risk/snapshot` | Current risk state |
| GET | `/api/v1/risk/settings` | User's risk limits |
| PUT | `/api/v1/risk/settings` | Update limits |
| GET | `/api/v1/risk/estimate` | Pre-trade risk estimate |

### PIE (Position Intelligence Engine)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/pie/status` | Full PIE status |
| POST | `/api/v1/strategies` | Create strategy |
| GET | `/api/v1/strategies` | List strategies |
| POST | `/api/v1/strategies/{id}/activate` | Activate strategy |
| POST | `/api/v1/strategies/{id}/stop` | Stop strategy |
| POST | `/api/v1/auto_entry` | Create auto-entry rule |
| GET | `/api/v1/auto_entry` | List auto-entry rules |
| PUT | `/api/v1/auto_entry/{id}` | Update rule |
| DELETE | `/api/v1/auto_entry/{id}` | Delete rule |
| POST | `/api/v1/kill_switch/trigger` | Trigger kill switch |
| POST | `/api/v1/kill_switch/reset` | Reset kill switch |
| GET | `/api/v1/actions` | Action log (recent PIE events) |

### Broker Connection + Session

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/trading_account` | List trading accounts |
| POST | `/api/v1/trading_account` | Create trading account |
| GET | `/api/v1/trading_account/{id}` | Account detail |
| DELETE | `/api/v1/trading_account/{id}` | Delete account |
| GET | `/api/v1/broker_connection` | List broker connections |
| POST | `/api/v1/broker_connection` | Create connection |
| DELETE | `/api/v1/broker_connection/{id}` | Remove connection |
| GET | `/oauth/authorize` | Start broker OAuth flow |
| GET | `/oauth/callback` | Handle OAuth callback |
| POST | `/api/v1/session/start` | Start broker session |
| POST | `/api/v1/session/stop` | Stop broker session |
| GET | `/api/v1/session/status` | Session status |

### Preferences

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/preferences` | Get user preferences |
| PUT | `/api/v1/preferences` | Update preferences |

---

## Market Data Service (Port 8004)

### Instruments

| Method | Path | Description |
|--------|------|-------------|
| GET | `/instruments` | Search instruments (`?q=RELIANCE&exchange=NSE`) |
| GET | `/instruments/{id}` | Instrument detail |
| GET | `/broker_instruments/{broker_id}` | Broker-specific symbols |

### Data

| Method | Path | Description |
|--------|------|-------------|
| GET | `/data/historical` | OHLCV candles (`?symbol=NSE:RELIANCE-EQ&resolution=5&from=...&to=...`) |
| GET | `/margin` | Margin requirements (`?symbols=NSE:RELIANCE-EQ,NSE:INFY-EQ`) |

### Options

| Method | Path | Description |
|--------|------|-------------|
| GET | `/option_chain` | Option chain snapshot (`?underlying=NIFTY&expiry=2026-04-24`) |
| GET | `/option_chain/expiries` | Available expiries (`?underlying=NIFTY`) |
| GET | `/greeks` | Greeks for option (`?symbol=NSE:NIFTY2540022500CE`) |
| GET | `/iv_metrics` | IV rank + percentile (`?symbol=NSE:NIFTY50-IDX`) |
| GET | `/iv_metrics/history` | IV history chart data |

### WebSocket

```
WS /ws/{broker_id}/{user_id}

# Subscribe to quotes
→ { "type": "subscribe", "symbols": ["NSE:RELIANCE-EQ"] }

# Incoming tick
← { "type": "tick", "symbol": "NSE:RELIANCE-EQ", "ltp": "2847.50", ... }

# Subscribe to options (includes Greeks)
→ { "type": "subscribe_options", "symbols": ["NSE:NIFTY2540022500CE"] }
← { "type": "option_tick", "symbol": "...", "ltp": "145.50", "delta": "0.42", ... }

# Unsubscribe
→ { "type": "unsubscribe", "symbols": ["NSE:RELIANCE-EQ"] }
```

---

## Common Response Formats

### Error Response

```json
{
  "error_code": "AUTH_001",
  "message": "Invalid credentials",
  "details": null
}
```

### Paginated Response

```json
{
  "items": [...],
  "total": 100,
  "page": 1,
  "page_size": 20,
  "has_next": true
}
```

---

## Order Types & Enums

```
side:         BUY | SELL
order_type:   MARKET | LIMIT | SL | SL-M
product_type: INTRADAY | DELIVERY | MTF
status:       PENDING | OPEN | PARTIALLY_FILLED | FILLED | CANCELLED | REJECTED | SETTLED
exchange:     NSE | BSE | NFO | MCX
```

---

## Rate Limits

| Endpoint Group | Limit |
|----------------|-------|
| `/login` | 20/min per IP |
| `/api/v1/orders` (POST) | 60/min per user |
| All other authenticated | 300/min per user |
| WebSocket connections | 5 per user |

Rate limit headers: `X-RateLimit-Remaining`, `X-RateLimit-Reset`
Exceeded: `429 Too Many Requests`
