# SmartTrade Project — Documentation Index

Design documents, architecture references, guides, and project planning for the SmartTrade trading platform.

## Quick Links

| Document | Description |
|----------|-------------|
| [Architecture](docs/ARCHITECTURE.md) | Production-grade system architecture |
| [Roadmap](docs/ROADMAP.md) | Feature roadmap Q1–Q4 2026 |
| [API Overview](docs/API_OVERVIEW.md) | All REST + WebSocket APIs |
| [Financial Correctness](docs/FINANCIAL_CORRECTNESS.md) | Non-negotiable rules for financial code |

## How-To Guides

| Guide | Description |
|-------|-------------|
| [Setup](docs/guides/SETUP.md) | Local development setup |
| [Deploy](docs/guides/DEPLOY.md) | Production deployment |
| [Add Broker](docs/guides/ADD_BROKER.md) | Integrate a new broker |
| [Add Feature](docs/guides/ADD_FEATURE.md) | Implement a new feature |
| [Testing](docs/guides/TESTING.md) | Testing patterns and best practices |

## Service Design Documents

| Service | Document |
|---------|----------|
| Authentication | [design/authentication-service/DESIGN.md](design/authentication-service/DESIGN.md) |
| Broker Adapter | [design/broker-adapter-service/Design.md](design/broker-adapter-service/Design.md) |
| → Order Lifecycle | [design/broker-adapter-service/ORDER_LIFECYCLE_DESIGN.md](design/broker-adapter-service/ORDER_LIFECYCLE_DESIGN.md) |
| → Risk Engine | [design/broker-adapter-service/RISK_ENGINE_DESIGN.md](design/broker-adapter-service/RISK_ENGINE_DESIGN.md) |
| → PIE Engine | [design/broker-adapter-service/PIE_DESIGN.md](design/broker-adapter-service/PIE_DESIGN.md) |
| → Settlement (T+1) | [design/broker-adapter-service/SETTLEMENT_DESIGN.md](design/broker-adapter-service/SETTLEMENT_DESIGN.md) |
| → Position Management | [design/broker-adapter-service/POSITION_MANAGEMENT_DESIGN.md](design/broker-adapter-service/POSITION_MANAGEMENT_DESIGN.md) |
| Market Data | [design/market-data-service/DESIGN.md](design/market-data-service/DESIGN.md) |
| Frontend | [design/frontend/DESIGN.md](design/frontend/DESIGN.md) |
| smarttrade-common | [design/smarttrade-common/DESIGN.md](design/smarttrade-common/DESIGN.md) |

## Platform Overview

SmartTrade is a broker-agnostic algorithmic trading platform:

- **5 microservices**: Auth, BAS (Broker Adapter), MDS (Market Data), Mock, Frontend
- **Broker plugins**: Fyers (live), Paper (simulation), + extensible for others
- **PIE Engine**: Position automation — auto-entry, kill switch, strategy execution
- **Risk Engine**: Config-driven rules (daily loss, position limits, per-trade risk)
- **Options**: Live chains, Black-Scholes Greeks, IV rank/percentile
- **Settlement**: T+1 settlement cycle with immutable audit trail
- **250+ tests** across all services

## Services & Ports

| Service | Port | Database |
|---------|------|----------|
| Authentication | 8001 | `smarttrade_authentication_service` |
| Mock Service | 8002 | `smarttrade_mock_service` |
| Market Data (MDS) | 8004 | `smarttrade_market_data_service` |
| Broker Adapter (BAS) | 8005 | `smarttrade_broker_adapter_service` |
| Frontend | 5173 | — |

## Legacy Docs

- [Architecture Implementation Plan](docs/ARCHITECTURE_IMPLEMENTATION_PLAN.json)
- [Fyers Integration Guide](docs/FYERS_INTEGRATION_GUIDE.md)
- [Transaction Boundaries](docs/TRANSACTION_BOUNDARIES.md)
- [Phase 5 Frontend Integration](docs/PHASE_5_FRONTEND_INTEGRATION.md)
