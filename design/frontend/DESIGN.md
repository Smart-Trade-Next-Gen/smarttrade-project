# SmartTrade Frontend — Design Document

**Version:** 1.0
**Framework:** React 18 + TypeScript + Vite
**Port:** 5173 (dev), 80 (production via nginx)

---

## 1. Architecture Overview

```
smarttrade-frontend/src/
    App.tsx                 # Router + auth guard (RequireAuth)
    main.tsx               # React 18 root, WebSocketProvider mount

    api/                   # Axios clients per service
    ws/                    # WebSocket connection layer
    store/                 # Zustand domain stores
    hooks/                 # React hooks (data fetching + store access)
    components/            # UI components
    pages/                 # Route-level page components
    types/                 # TypeScript type definitions
    utils/                 # Pure utility functions
```

**Design Principle:** Each layer has a single responsibility. Components never call APIs directly — they use hooks. Hooks use stores or API functions. Stores hold normalized state.

---

## 2. State Management (Zustand)

Each domain has its own Zustand store. No shared mutable state between stores.

### Store Inventory

| Store | Holds | Updated by |
|-------|-------|-----------|
| `authStore` | User identity, JWT tokens | `useAuthService` hook |
| `accountStore` | Trading accounts, broker connections | `useAccountsService` |
| `brokerAuthStore` | Broker OAuth state | OAuth flow |
| `orderStore` | Open/completed orders | `useLoadOrders` + WS |
| `positionsStore` | Current positions | `useLoadPositions` + WS |
| `portfolioStore` | Holdings, portfolio summary | `useLoadPositions` |
| `marketDataStore` | Live LTP per symbol | WebSocket fanout |
| `tradeStore` | Executed trades | `useLoadTrades` |
| `fundsStore` | Available margin, used margin | `useLoadFunds` |
| `pieStore` | Strategy state, kill switch, actions | `usePIEStatus` + WS |
| `positionGroupsStore` | Position groups + members | REST API |
| `notificationStore` | Toast + alert queue | Various event handlers |
| `sessionStore` | Broker session state | `useStartTradingSession` |
| `panelStore` | Dashboard panel layout | User preferences |
| `searchStore` | Instrument search results | `useInstrumentSearch` |
| `selectedInstrumentStore` | Currently focused instrument | User interaction |
| `chartResolutionStore` | Chart timeframe selection | Timeframe selector |
| `holdingStore` | Long-term holdings | `useLoadPositions` |

---

## 3. API Client Layer

Each backend service has its own Axios client configured with the service base URL:

```typescript
// api/authClient.ts
const authClient = axios.create({ baseURL: import.meta.env.VITE_AUTH_BASE });

// api/basClient.ts
const basClient = axios.create({ baseURL: import.meta.env.VITE_BAS_BASE });
basClient.interceptors.request.use(attachJWT);
basClient.interceptors.response.use(identity, handleAuthError);

// api/marketDataClient.ts
const mdsClient = axios.create({ baseURL: import.meta.env.VITE_MDS_BASE });
```

API functions are plain async functions grouped by domain:

```typescript
// api/orderApi.ts
export const placeOrder = (request: PlaceOrderRequest) =>
    basClient.post<OrderResponse>('/api/v1/orders', request);

export const getOrders = (params: OrderFilter) =>
    basClient.get<PaginatedResponse<OrderResponse>>('/api/v1/orders', { params });
```

---

## 4. WebSocket Architecture

### WebSocketProvider (ws/WebSocketProvider.tsx)

Top-level provider that manages one WebSocket connection to each active broker session. Mounted at the app root, available to all components.

```typescript
const WebSocketProvider = ({ children }) => {
    const { brokerId, userId } = useResolvedBrokerAndAccount();

    useEffect(() => {
        const ws = new WebSocketClient(`${WAS_BASE}/ws/${brokerId}/${userId}`);
        ws.onMessage(websocketDataFanout);
        return () => ws.close();
    }, [brokerId, userId]);

    return children;
};
```

### websocketDataFanout.ts

Routes incoming WebSocket messages to the correct store:

```typescript
export function websocketDataFanout(message: WsMessage) {
    switch (message.type) {
        case "tick":
            marketDataStore.getState().updateLTP(message.symbol, message.ltp);
            break;
        case "order_update":
            orderStore.getState().updateOrder(message.order);
            break;
        case "position_update":
            positionsStore.getState().updatePosition(message.position);
            break;
        case "pie_status":
            pieStore.getState().updateStatus(message.status);
            break;
        case "risk_alert":
            notificationStore.getState().addAlert(message);
            break;
    }
}
```

### Subscription Registry (ws/subscriptionRegistry.ts)

Tracks which symbols are subscribed. Components register/unregister subscriptions; the registry sends batch subscribe/unsubscribe messages to the WebSocket.

```typescript
subscriptionRegistry.subscribe(["NSE:RELIANCE-EQ", "NSE:NIFTY50-IDX"]);
subscriptionRegistry.unsubscribe(["NSE:RELIANCE-EQ"]);
```

---

## 5. Chart System

### EnhancedChart (components/Chart/EnhancedChart.tsx)

The main charting component. Built on `lightweight-charts`. Supports:

- Multiple pane layout (price + indicators)
- Realtime candle building from tick data
- Historical candle loading (via `useChartHistory`)
- Drawing tools (trend lines, S/R levels)
- Chart overlays (trades, entries, exits)
- Replay mode (historical market replay)
- Multi-chart crosshair sync
- Timeframe sync across charts

### Internal Architecture

```
EnhancedChart
    ├── useChartInstance       Creates/destroys lightweight-charts instance
    ├── useChartHistory        Fetches + loads historical OHLCV
    ├── useChartRealtime       Subscribes to WS ticks, builds live candles
    │       └── RealtimeCandleBuilder    Aggregates ticks into OHLCV
    ├── useChartIndicators     Computes + renders EMA, VWAP, BB, etc.
    ├── useChartOverlays       Renders trade markers on chart
    ├── useChartReplay         Historical replay mode controller
    ├── useCrosshairSync       Syncs crosshair position across charts
    │       └── crosshairSyncBus        Broadcast bus (custom EventEmitter)
    ├── useTimeRangeSync       Syncs visible time range across charts
    └── useLinkedTimeframe     Shares timeframe state across charts
```

### Chart Registry (components/Chart/internal/chartRegistry.ts)

Singleton that tracks all mounted chart instances. Enables crosshair sync and time range sync across multiple charts in a grid layout.

---

## 6. Dashboard Layout System

The dashboard uses `react-grid-layout` for a drag-and-drop panel grid.

### Panel System

```typescript
// components/PanelRenderer.tsx
const PANEL_MAP: Record<string, React.FC> = {
    OrdersPanel,
    PositionsPanel,
    PortfolioAnalytics,
    OptionsChain,
    RiskManagementPanel,
    // ... 60+ panel components
};

// PanelRenderer renders the correct component for each grid item
const PanelRenderer = ({ panelType }: { panelType: string }) => {
    const Component = PANEL_MAP[panelType];
    return <Component />;
};
```

### Workspace Manager

`WorkspaceManager.tsx` handles saving/loading dashboard layouts. Layouts are persisted to user preferences via `preferences.ts` API.

---

## 7. PIE Frontend

The PIE (Position Intelligence Engine) interface provides:

| Component | Purpose |
|-----------|---------|
| `PIEDashboard.tsx` | Main PIE page with status overview |
| `StrategyPanel.tsx` | List, create, activate, stop strategies |
| `AutoEntryPanel.tsx` | Auto-entry rule management |
| `KillSwitchButton.tsx` | Emergency stop button with confirm dialog |
| `MonitorLog.tsx` | Real-time action log feed |

**State:** `pieStore.ts` holds kill switch state, active strategies, recent actions.

**Real-time updates:** `usePIESubscription.ts` subscribes to `pie.*` WebSocket events.

---

## 8. Position Management Frontend

| Component | Purpose |
|-----------|---------|
| `PositionGroupsPanel.tsx` | List all groups, P&L per group |
| `GroupDetailsPanel.tsx` | Members, rules, individual P&L |
| `GroupRulesPanel.tsx` | Edit exit rules for a group |
| `AggregatedPnLPanel.tsx` | Real-time group P&L (updates on tick) |
| `AddToGroupModal.tsx` | Add open position to a group |
| `ExitConfirmModal.tsx` | Confirm group exit with summary |
| `PositionsTable.tsx` | Positions with group membership |

**P&L computation:** Client-side using `positionGroupSelectors.ts`. Selectors derive group P&L from `positionsStore` + `marketDataStore` without extra API calls.

---

## 9. Authentication Flow

```
User visits / → RequireAuth → no token → redirect to /login
/login → LoginPage → authApi.login() → JWT stored in authStore (memory + secure cookie)
→ redirect to /dashboard

Token refresh: authStore.refreshToken() called 1 minute before expiry
JWT attached to all BAS/MDS requests via Axios interceptor

Logout: authApi.logout() → clear authStore → redirect to /login
```

---

## 10. Key Hooks

| Hook | Purpose |
|------|---------|
| `useLoadOrders` | Fetch orders, set orderStore |
| `useLoadPositions` | Fetch positions, set positionsStore + portfolioStore |
| `useLoadFunds` | Fetch available margin |
| `useOptionChain` | Fetch option chain + subscribe to WS ticks |
| `useInstrumentSearch` | Debounced instrument search |
| `useRiskSnapshot` | Poll current risk state |
| `usePIEStatus` | Poll PIE status + handle WS updates |
| `useStartTradingSession` | Initiate broker session |
| `useOrderPanel` | Order placement form state |
| `useStrategies` | PIE strategy management |

---

## 11. Type Safety

All API response types defined in `types/`:

```typescript
// types/api.ts
export interface OrderResponse {
    id: string;
    symbol: string;
    side: 'BUY' | 'SELL';
    qty: number;
    price: string;          // Decimal as string (never number)
    status: OrderStatus;
    filledQty: number;
    avgFillPrice: string | null;
}
```

**Note:** All monetary values received from the API are strings (Decimal serialized). Client converts to `parseFloat()` only for display; never for computation. This mirrors the backend's Decimal-only rule.

---

## 12. Environment Variables

```bash
VITE_AUTH_BASE=http://localhost:8001    # Authentication Service
VITE_BAS_BASE=http://localhost:8005     # Broker Adapter Service
VITE_MDS_BASE=http://localhost:8004     # Market Data Service
VITE_WAS_BASE=ws://localhost:8004       # WebSocket (MDS)
VITE_MOCK_BASE=http://localhost:8002    # Mock Service
```

---

## 13. Build + Lint

```bash
npm run dev       # Vite HMR dev server
npm run build     # Production bundle (tree-shaken, chunked)
npm run lint      # ESLint (TypeScript strict)
npm run preview   # Preview production build locally
```

Production build produces:
- `index.html` (single entry point)
- Vendor chunk (React, Zustand, Axios)
- Chart chunk (lightweight-charts, large — lazy loaded)
- Feature chunks (PIE, Position Management — lazy loaded by route)
