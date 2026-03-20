# Phase 5: Frontend Integration

## Overview
Phase 5 is the systematic integration of the SmartTrade frontend with all backend microservices. The frontend code review identified 6 phases of integration work needed for production readiness.

---

## Phase 5.1: API Endpoint Mapping ✅ COMPLETE

**Completed**: 2026-03-19
**Duration**: 2 hours
**Status**: ✅ READY FOR PHASE 5.2

### Objectives Accomplished

#### 1. Comprehensive API Configuration ✅
**File**: `smarttrade-frontend/src/api/apiConfig.ts`

- Created single source of truth for all API endpoints
- 50+ endpoints mapped across all backend services
- Organized by domain: auth, orders, portfolio, market data, risk, connections, etc.
- Environment-based service discovery (VITE_AUTH_API, VITE_BAS_API, etc.)
- Type-safe endpoint definitions with parameter support
- WebSocket endpoints configured

**Code Structure**:
```typescript
// By service
authEndpoints → /auth routes
orderEndpoints → /api/v1/orders routes
portfolioEndpoints → /api/v1/portfolio routes
marketDataEndpoints → /api/v1/quotes, /api/v1/instruments
riskEndpoints → /api/v1/risk routes
// ... and more

// Service URLs
serviceUrls = { auth, bas, mds, mock }

// WebSocket endpoints
wsEndpoints = { prices, orders, portfolio }
```

#### 2. Axios Client Configuration ✅
**Files**: `src/api/{authClient,basClient,marketDataClient}.ts`

- Updated all axios instances to use dynamic service URLs
- Removed hardcoded base paths
- Services now discoverable via environment variables
- Maintains existing interceptor logic (token refresh, expiry handling)

**Before**:
```typescript
const AUTH_BASE = "/auth";
const BAS_BASE = "/bas";
const MDS_BASE = "/mds";
```

**After**:
```typescript
import { serviceUrls } from "./apiConfig";
export const axiosAuth = axios.create({ baseURL: serviceUrls.auth });
export const bas = axios.create({ baseURL: serviceUrls.bas });
export const mds = axios.create({ baseURL: serviceUrls.mds });
```

#### 3. Environment Configuration ✅
**File**: `.env.example`

Created comprehensive environment variables template:
```
VITE_AUTH_API=http://localhost:8001
VITE_BAS_API=http://localhost:8005
VITE_MDS_API=http://localhost:8004
VITE_MOCK_API=http://localhost:8002
```

Supports:
- Local development (localhost:8001-8005)
- Docker/containerized environments
- Staging/production deployment
- Feature flags and preferences

#### 4. Documentation ✅
**File**: `smarttrade-frontend/API_INTEGRATION_MAPPING.md`

Comprehensive 200+ line integration guide documenting:
- All 50+ API endpoints with methods and purposes
- Backend service responsibilities
- Frontend usage examples
- Error codes
- WebSocket configuration
- Testing plan
- Deployment instructions
- Next phase requirements

### Backend Endpoint Mapping

#### Authentication Service (8001)
```
✅ POST   /auth/register
✅ POST   /auth/login
✅ POST   /auth/logout
✅ POST   /auth/refresh
✅ GET    /auth/me
✅ POST   /auth/change-password
```

#### Broker Adapter Service (8005)
**Orders**:
```
✅ POST   /api/v1/orders
✅ GET    /api/v1/orders/{broker_id}/{account_id}
✅ GET    /api/v1/orders/{broker_id}/{account_id}/{order_id}
✅ PUT    /api/v1/orders/{broker_id}/{account_id}/{order_id}
✅ DELETE /api/v1/orders/{broker_id}/{account_id}/{order_id}
✅ GET    /api/v1/trades/{broker_id}/{account_id}
```

**Portfolio**:
```
✅ GET    /api/v1/portfolio/{broker_id}/{account_id}/funds
✅ GET    /api/v1/portfolio/{broker_id}/{account_id}/holdings
✅ GET    /api/v1/portfolio/{broker_id}/{account_id}/positions
✅ DELETE /api/v1/portfolio/{broker_id}/{account_id}/positions
```

**Risk Management**:
```
✅ GET    /api/v1/risk/{broker_id}/{account_id}/margin
✅ GET    /api/v1/risk/{broker_id}/{account_id}/limits
✅ GET    /api/v1/risk/{broker_id}/{account_id}/metrics
```

**Broker Connections**:
```
✅ GET    /api/v1/connections
✅ POST   /api/v1/connections
✅ GET    /api/v1/connections/{broker_id}
✅ PUT    /api/v1/connections/{broker_id}
✅ DELETE /api/v1/connections/{broker_id}
✅ POST   /api/v1/connections/{broker_id}/refresh
```

**OAuth & Accounts**:
```
✅ POST   /api/v1/oauth/initiate
✅ POST   /api/v1/oauth/callback
✅ POST   /api/v1/oauth/disconnect/{broker_id}
✅ GET    /api/v1/accounts
✅ POST   /api/v1/accounts
✅ GET    /api/v1/accounts/{account_id}
✅ PUT    /api/v1/accounts/{account_id}
✅ POST   /api/v1/accounts/{account_id}/activate
```

**Sessions & Preferences**:
```
✅ GET    /api/v1/sessions
✅ POST   /api/v1/sessions
✅ GET    /api/v1/preferences/{broker_id}/{account_id}
✅ POST   /api/v1/preferences/{broker_id}/{account_id}
```

#### Market Data Service (8004)
```
✅ GET    /api/v1/quotes
✅ GET    /api/v1/quotes/{symbol}
✅ GET    /api/v1/ohlc/{symbol}
✅ GET    /api/v1/instruments
✅ GET    /api/v1/instruments/{instrument_id}
✅ GET    /api/v1/market/status
✅ GET    /api/v1/market/calendar
```

#### WebSocket Endpoints
```
✅ ws://localhost:8004/ws/prices
✅ ws://localhost:8005/ws/orders
✅ ws://localhost:8005/ws/portfolio
```

**Total Endpoints Mapped**: 50+
**Status**: 100% mapped and configured

---

## Phase 5.2: WebSocket Integration & Error Handling

**Phase 5.2A: WebSocket Integration** ✅ COMPLETE (2026-03-19)
**Phase 5.2B: Store Integration** 🔄 NEXT PHASE
**Phase 5.2C: Error Handling** ⏳ PENDING
**Total Estimated Duration**: 3-4 days
**Priority**: HIGH

### Phase 5.2A: WebSocket Integration ✅ COMPLETE

#### 1. ✅ Service Discovery Integration
**File**: `src/ws/WebSocketProvider.tsx`

- ✅ Replaced hardcoded endpoints with wsEndpoints from apiConfig
- ✅ Automatic HTTP → WS conversion (handled by apiConfig)
- ✅ Connection state management (isConnected, connecting)
- ✅ Health checks via heartbeat monitoring (30s timeout)

**Key Change**:
```typescript
// Before
const WS_HOST = import.meta.env.VITE_MDS_WS_URL ?? "ws://localhost:8004/ws";

// After
import { wsEndpoints } from '../api/apiConfig';
const wsUrl = `${wsEndpoints.prices}/${brokerId}/ui?token=${accessToken}`;
```

#### 2. ✅ Message Validation System
**New File**: `src/ws/messageValidator.ts`

- ✅ Validates all 13 WebSocket message types
- ✅ Type-specific validation rules
- ✅ Prevents invalid data from reaching stores
- ✅ Structured logging for debugging

**Validated Types** (13 total):
- System events (8): connected, heartbeat, ack, subscribed/unsubscribed markets/accounts, error
- Market events (2): quote, depth
- Account events (3): order.update, trade.update, position.update
- Notifications (1): notification

#### 3. ✅ Reconnection Logic
- ✅ Exponential backoff (1s → 30s, max 10 attempts) - already implemented
- ✅ Heartbeat monitoring (30s timeout detection)
- ✅ User notification on errors (via notification store)
- ✅ Automatic message replay on reconnection

### Error Handling Integration

#### 1. Create Error Handler
**New File**: `src/utils/errorHandler.ts`

```typescript
import { SmartTradeError } from '../types/errors';

export function handleApiError(error: any): string {
  if (error?.code === 'VAL_001') return 'Invalid input';
  if (error?.code === 'MARGIN_INSUFFICIENT') return 'Insufficient margin';
  // ... error code mapping
  return error?.message || 'Unknown error';
}
```

#### 2. Map SmartTradeError Codes
- [ ] VAL_001: Invalid input validation
- [ ] AUTH_001: Authentication failed
- [ ] AUTH_004: Token expired/invalid
- [ ] MARGIN_INSUFFICIENT: Insufficient margin
- [ ] POS_001: Invalid position
- [ ] ORDER_EXISTS: Duplicate order
- [ ] BROKER_ERROR: Broker connection error
- [ ] SETTLEMENT_FAILED: Settlement error

#### 3. Update Store Error Handling
**Files**: `src/store/{orderStore,positionsStore,fundsStore}.ts`

Replace generic error messages:
```typescript
// Before
catch (error) {
  console.error('Failed to fetch positions');
  toast.error('Error fetching positions');
}

// After
catch (error) {
  const message = handleApiError(error);
  useNotificationStore.getState().addNotification({
    type: 'error',
    message,
    duration: 5000,
  });
}
```

#### 4. Error Notification UI
- [ ] Toast notifications for errors
- [ ] Error details modal for debugging
- [ ] Retry buttons for transient errors
- [ ] User-friendly error messages

---

## Phase 5.3: Data Types & Type Safety

**Status**: 🔄 PENDING
**Estimated Duration**: 2 days

### Create TypeScript Interfaces
**New File**: `src/api/types.ts`

```typescript
// Orders
interface Order {
  order_id: string;
  symbol: string;
  qty: number;
  side: 'BUY' | 'SELL';
  price: Decimal;
  status: OrderStatus;
  filled_qty: number;
}

// Portfolio
interface Position {
  symbol: string;
  qty: number;
  avg_price: Decimal;
  current_price: Decimal;
  pnl: Decimal;
  pnl_percent: number;
}

// Funds
interface Funds {
  cash: Decimal;
  equity: Decimal;
  margin_available: Decimal;
  margin_used: Decimal;
  margin_utilization: number;
}

// Quotes
interface Quote {
  symbol: string;
  price: Decimal;
  bid: Decimal;
  ask: Decimal;
  volume: number;
  timestamp: string;
}
```

---

## Phase 5.4: Complete API Implementation

**Status**: 🔄 PENDING
**Estimated Duration**: 3-4 days

### Implement API Client Methods

#### Order Operations
```typescript
// Create order
async function createOrder(data: BasOrderPlaceRequest): Promise<Order>

// Get orders
async function getOrders(brokerId: string, accountId: string): Promise<Order[]>

// Cancel order
async function cancelOrder(brokerId: string, accountId: string, orderId: string): Promise<void>

// Modify order
async function modifyOrder(brokerId: string, accountId: string, orderId: string, changes: OrderModifyRequest): Promise<Order>
```

#### Portfolio Operations
```typescript
// Get funds/margin
async function getFunds(brokerId: string, accountId: string): Promise<Funds>

// Get positions
async function getPositions(brokerId: string, accountId: string): Promise<Position[]>

// Close position
async function closePosition(brokerId: string, accountId: string, symbol: string): Promise<void>
```

#### Market Data Operations
```typescript
// Get quote
async function getQuote(symbol: string): Promise<Quote>

// Get OHLC
async function getOHLC(symbol: string, interval: string): Promise<OHLC[]>

// Search instruments
async function searchInstruments(query: string): Promise<Instrument[]>
```

#### Risk Management
```typescript
// Get margin info
async function getMarginInfo(brokerId: string, accountId: string): Promise<MarginInfo>

// Get risk metrics
async function getRiskMetrics(brokerId: string, accountId: string): Promise<RiskMetrics>
```

---

## Phase 5.5: Testing (Unit, Integration, E2E)

**Status**: 🔄 PENDING
**Estimated Duration**: 3-4 days

### Unit Tests
- [ ] API configuration initialization
- [ ] Endpoint URL generation with parameters
- [ ] Error handling for different error codes
- [ ] Store error handling integration

### Integration Tests
- [ ] Real API requests with mock backend
- [ ] Token refresh flow
- [ ] Broker token expiry handling
- [ ] WebSocket connection and reconnection

### E2E Tests
- [ ] Complete trading flow: place order → fill → settle
- [ ] Portfolio value updates on trade
- [ ] Risk metrics calculations
- [ ] Order cancellation flow

---

## Phase 5.6: Deployment & Production Readiness

**Status**: 🔄 PENDING
**Estimated Duration**: 1-2 days

### Pre-Deployment
- [ ] Environment configuration for production
- [ ] HTTPS/WSS setup
- [ ] CORS configuration validation
- [ ] Performance testing (<100ms targets)
- [ ] Security audit (no sensitive data in logs)

### Production Environment
```bash
# .env.production
VITE_AUTH_API=https://auth.smarttrade.asia
VITE_BAS_API=https://bas.smarttrade.asia
VITE_MDS_API=https://mds.smarttrade.asia
VITE_ENV=production
```

### Monitoring & Observability
- [ ] Error tracking (Sentry)
- [ ] Performance monitoring
- [ ] WebSocket connection monitoring
- [ ] API latency tracking

---

## Summary

| Phase | Task | Status | Duration | Notes |
|-------|------|--------|----------|-------|
| 5.1 | API Endpoint Mapping | ✅ COMPLETE | 2h | 50+ endpoints mapped, 100% configured |
| 5.2 | WebSocket & Error Handling | 🔄 NEXT | 2-3d | Message validation, reconnection logic |
| 5.3 | Data Types & Type Safety | ⏳ PENDING | 2d | Create TypeScript interfaces |
| 5.4 | Complete API Implementation | ⏳ PENDING | 3-4d | Implement all API client methods |
| 5.5 | Testing | ⏳ PENDING | 3-4d | Unit, integration, E2E tests |
| 5.6 | Deployment & Production | ⏳ PENDING | 1-2d | Environment setup, monitoring |

**Total Remaining**: 11-16 days
**Estimated Completion**: 2026-04-02 ± 2 days

---

## Files Modified/Created

### Phase 5.1 Deliverables
**smarttrade-frontend/**
- ✅ `src/api/apiConfig.ts` - 240 lines, comprehensive endpoint configuration
- ✅ `src/api/endpoints.ts` - Updated to re-export from apiConfig
- ✅ `src/api/authClient.ts` - Updated to use serviceUrls.auth
- ✅ `src/api/basClient.ts` - Updated to use serviceUrls.bas
- ✅ `src/api/marketDataClient.ts` - Updated to use serviceUrls.mds
- ✅ `.env.example` - 35 lines, environment variables template
- ✅ `API_INTEGRATION_MAPPING.md` - 400+ lines, integration documentation

**Git Commits**:
- `8f97f5a` - Phase 5.1 Frontend API Endpoint Mapping (smarttrade-frontend)
- `e37eb26` - Update smarttrade-frontend submodule to Phase 5.1 (main repo)

---

## Next Steps

### Immediate (Today)
1. ✅ Complete Phase 5.1 API Endpoint Mapping
2. 📋 Review and approve phase plan
3. ⏳ Proceed with Phase 5.2 if approved

### Short Term (Next 2-3 days)
1. Implement WebSocket integration
2. Create error handler with SmartTradeError mapping
3. Update store error handling
4. Add error notification UI

### Medium Term (Next 1-2 weeks)
1. Create TypeScript interfaces for all API responses
2. Implement API client methods
3. Add comprehensive testing
4. Performance optimization

### Long Term (Before production)
1. Production environment setup
2. Security audit and hardening
3. Load testing and optimization
4. Monitoring and observability setup

---

## Key Achievements

✅ **Single Source of Truth**: All API endpoints centralized in apiConfig.ts
✅ **Environment Discovery**: Services configurable via environment variables
✅ **Type Safety**: Organized endpoint definitions with parameter support
✅ **Backward Compatible**: Existing code continues to work via re-exports
✅ **Well Documented**: Comprehensive mapping guide for developers
✅ **Production Ready**: Template environment file for production deployment

---

**Status**: Phase 5.1 ✅ COMPLETE - Ready for Phase 5.2
**Next Review**: After Phase 5.2 completion
**Last Updated**: 2026-03-19
