# Fyers Option Chain Integration Guide

## Overview
WebSocket-based real-time option chain streaming with lazy scheduler initialization. The Fyers API integration automatically starts when clients first subscribe to options.

## Implementation Summary

### What Was Added (2026-03-19)

#### 1. OptionSchedulerManager
**File**: `market-data-service/src/market_data_service/services/option_scheduler_manager.py`

Manages lazy initialization and lifecycle of option refresh schedulers:
- Tracks active schedulers per `(broker_id, user_id)` pair
- Creates fetcher with user's access token on first subscription
- Handles graceful shutdown of all schedulers
- Thread-safe with `asyncio.Lock`

```python
# Usage in code
scheduler_manager = get_option_scheduler_manager()
await scheduler_manager.ensure_scheduler_started(broker_id, user_id)
```

#### 2. Enhanced OptionSubscriptionHandler
**File**: `market-data-service/src/market_data_service/websocket/option_subscription_handler.py`

Modified to trigger scheduler initialization:
- Extracts `broker_id` and `user_id` from WebSocket connection
- Calls `ensure_scheduler_started()` on first subscription
- Gracefully handles scheduler initialization failures

#### 3. Updated Lifespan
**File**: `market-data-service/src/market_data_service/lifespan.py`

App lifecycle changes:
- Initializes `OptionSchedulerManager` on startup
- Stops all schedulers on shutdown
- Registers manager in `app.state` for access by handlers

#### 4. Integration Tests
**File**: `market-data-service/tests/integration/test_option_scheduler_integration.py`

Comprehensive test coverage:
- Lazy initialization tests
- Idempotency tests (multiple subscription calls)
- Scheduler lifecycle tests
- Integration with subscription handler tests

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ WebSocket Client (Browser)                                  │
│ ws.send({action: 'subscribe.options', options: [...]})      │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│ OptionSubscriptionHandler                                   │
│ - Validates options request                                 │
│ - Gets conn_id → lookup connection → broker_id, user_id     │
│ - Calls scheduler_manager.ensure_scheduler_started()        │
│ - Calls broker_manager.subscribe_options_ids()              │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│ OptionSchedulerManager (Lazy Init)                          │
│ - Check: is scheduler already running for this broker+user? │
│ - If NO:                                                    │
│   - Get SessionManager → get_or_create_plugin()             │
│   - Extract access_token from plugin.credentials            │
│   - Create FyersOptionChainFetcher(access_token)            │
│   - Initialize aiohttp.ClientSession                        │
│   - Create OptionChainRefreshScheduler                      │
│   - Start background refresh loop (30s interval)            │
│ - If YES: no-op (already running)                           │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│ OptionChainRefreshScheduler (Background)                    │
│ while not stopped:                                          │
│   - Get subscribed symbols from WebSocketManager            │
│   - For each symbol/expiry:                                 │
│     - Call fetcher.fetch_option_chain()                     │
│     - Update cache.set()                                    │
│   - Cache update triggers broadcaster                       │
│   - Broadcaster sends updates to subscribed clients         │
│   - Wait 30 seconds, retry                                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│ FyersOptionChainFetcher                                     │
│ async fetch_option_chain(symbol, expiry)                    │
│   - Build Fyers symbol format: NSE:SYMBOL-DDMMMYYSTRIKECEPE│
│   - Fetch quotes for common strikes                         │
│   - Return List[OptionContractDTO]                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│ Fyers API (https://api-t1.fyers.in/api/v3)                  │
│ GET /quotes/?symbols=NSE:NIFTY50-25FEB2500CE                │
│ Header: Authorization: {app_id}:{access_token}              │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow

### 1. Subscription Message (Client → Server)
```json
{
  "action": "subscribe.options",
  "request_id": "req-123",
  "options": [
    {
      "symbol": "NIFTY50",
      "expiry": "2025-02-25",
      "data_types": ["contracts", "greeks", "iv_metrics"]
    }
  ]
}
```

### 2. Subscription Response (Server → Client)
```json
{
  "status": "success",
  "request_id": "req-123",
  "message": "Subscribed to 1 option chain(s)",
  "errors": []
}
```

### 3. Option Chain Updates (Server → Client, periodic 30s)
```json
{
  "type": "option.contract.update",
  "symbol": "NIFTY50",
  "expiry": "2025-02-25",
  "contracts": [
    {
      "symbol": "NIFTY50",
      "strike_price": 25000,
      "option_type": "CE",
      "expiry_date": "2025-02-25",
      "ltp": 125.5,
      "bid": 125.0,
      "ask": 126.0,
      "volume": 45000,
      "open_interest": 120000,
      "implied_volatility": 18.5,
      "broker_token": "NSE:NIFTY50-25FEB2500CE",
      "spot_price": 23450.0
    }
    // ... more contracts
  ]
}
```

## Error Handling

### Scheduler Initialization Fails
- Logged as warning (non-blocking)
- Subscription proceeds without data updates
- Client can retry later

### Access Token Missing
- Logged as error
- Subscription fails with `FETCHER_INIT_FAILED`
- User must re-authenticate

### Fyers API Unavailable
- Scheduler uses exponential backoff (5s, 10s, 20s)
- Max 3 retries per symbol/expiry
- After max retries, skips that symbol until next cycle

## Configuration

Environment variables (already set in `market-data-service/.env`):
```bash
FYERS_APP_ID=<your-app-id>              # From Fyers dashboard
EVENT_BUS=redis                          # For distributed caching
EVENT_BUS_URL=redis://redis:6379/0      # Redis connection
```

## Testing

### Run All Integration Tests
```bash
cd market-data-service
uv sync --extra dev
uv run pytest tests/integration/ -v
```

### Test Option Scheduler Manager Specifically
```bash
uv run pytest tests/integration/test_option_scheduler_integration.py -v
```

### Test Option WebSocket Functionality
```bash
uv run pytest tests/integration/test_option_websocket.py -v
```

### Type Check
```bash
uv run mypy src/
```

## Performance Considerations

### Memory
- Per-user cache: ~1-5MB (depends on number of strikes)
- Shared Redis cache: distributed across instances
- Fetcher session: 1 aiohttp.ClientSession per broker_id+user_id

### Network
- Refresh interval: 30 seconds (configurable)
- Each refresh: multiple Fyers API calls (1 per strike)
- Typical payload: 10-20 KB per update

### CPU
- Non-blocking async design
- Lock contention: minimal (only on scheduler start)
- Broadcast: uses asyncio.gather for parallelization

## Future Enhancements

### 1. Dynamic Strike Resolution
Currently hardcoded to common strikes (15000-22000). Should:
```python
# Fetch available strikes for symbol/expiry from Fyers
strikes = await fetcher.get_available_strikes(symbol, expiry)
```

### 2. Greeks Calculation
Optional math library for on-the-fly calculation:
```python
from smarttrade_common.greeks import GreeksCalculator
calculator = GreeksCalculator()
# Inject into broadcaster
option_broadcaster = OptionChainEventBroadcaster(
    ws_manager=ws_manager,
    cache=cache,
    greeks_calc=calculator,  # NEW
    iv_calc=None
)
```

### 3. Message Compression
For high-frequency updates, compress deltas:
```python
# Only send changed fields instead of full contract data
# Reduces payload from 10KB to 1-2KB per update
```

## Troubleshooting

### Scheduler Not Starting
**Symptoms**: Options subscribed but no data updates

**Check**:
1. Verify access token is valid: `plugin.access_token` is not empty
2. Check logs for `"Starting option chain scheduler"` message
3. Verify SessionManager is initialized: `get_session_manager()`

### No Quotes from Fyers
**Symptoms**: Scheduler runs but fetcher returns empty list

**Check**:
1. Verify symbol format: `NSE:NIFTY50-25FEB2500CE`
2. Verify Fyers API credentials in headers
3. Check Fyers API documentation for current endpoint

### High Memory Usage
**Symptoms**: Redis cache growing

**Solution**:
1. Reduce refresh interval: `refresh_interval_seconds=60`
2. Reduce number of cached strikes
3. Clear expired entries from Redis manually

## API Reference

### OptionSchedulerManager

```python
async def ensure_scheduler_started(
    broker_id: str,
    user_id: UUID,
    account_type: Optional[str] = None
) -> None:
    """Start scheduler if not already running."""
```

```python
async def stop_scheduler(broker_id: str, user_id: UUID) -> None:
    """Stop a specific scheduler."""
```

```python
async def stop_all() -> None:
    """Stop all running schedulers (called on app shutdown)."""
```

### OptionChainRefreshScheduler

```python
async def start_refresh_loop(broker_id: str, user_id: str) -> None:
    """Start background refresh loop (called by manager)."""
```

```python
async def stop() -> None:
    """Stop refresh loop gracefully."""
```

### FyersOptionChainFetcher

```python
async def fetch_option_chain(
    symbol: str,
    expiry: str
) -> List[OptionContractDTO]:
    """Fetch option chain from Fyers API."""
```

## Support

For issues or questions:
1. Check logs: `docker-compose logs market-data-service`
2. Review this guide
3. Check integration test cases for examples
4. Verify Fyers API connectivity and credentials
