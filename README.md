# SmartTrade Project — Documentation Index

Design documents, architecture references, guides, and project planning for the SmartTrade trading platform.

## Quick Links

| Document | Description |
|----------|-------------|
| **[Architecture v3.4 (Current)](smarttrade-architecture-v3.4-current.md)** | **← START HERE** Current production architecture (Phase 10 active) |
| [ROADMAP.md](ROADMAP.md) | Feature roadmap Q1–Q4 2026 |
| [CLAUDE.md](CLAUDE.md) | Global development guidance and architecture principles |

## Service-Specific Documentation

**Note**: Service-specific design and implementation documentation is maintained in each service's own repository under `docs/`. The locations below are for reference only.

### Core Services
- **Broker Adapter Service (BAS)**: See [`broker-adapter-service/docs/INDEX.md`](../broker-adapter-service/docs/INDEX.md) for complete documentation
  - Includes: Order State Machine, Execution Orchestrator, Idempotency, Outbox Pattern, Risk Engine, Fyers API reference
- **Market Data Service (MDS)**: See [`market-data-service/docs/`](../market-data-service/docs/) for documentation
- **Paper Broker Service** (formerly Mock Service): See [`paper-broker-service/docs/`](../paper-broker-service/docs/) for documentation
- **Authentication Service**: See [`authentication-service/docs/`](../authentication-service/docs/) for documentation

### Testing & Deployment
- **E2E Testing Strategy**: See [`smarttrade-tests/docs/E2E_TESTING_STRATEGY.md`](../smarttrade-tests/docs/E2E_TESTING_STRATEGY.md)
- **Deployment Configuration**: See [`smarttrade-deployment/`](../smarttrade-deployment/) for docker-compose and infrastructure as code

## Platform Overview

SmartTrade is a broker-agnostic algorithmic trading platform in production:

- **Core Services**: Authentication, Broker Adapter (BAS), Market Data (MDS), Paper Broker Service (paper trading), Frontend
- **Order Handling**: Order State Machine (Phase 4 ✅), Execution Orchestrator (Phase 8 ✅), Idempotency (Phase 3 ✅)
- **Event Architecture**: Outbox Pattern (Phase 5 ✅), transactional event publishing via Redis Streams
- **Broker Adapters**: Fyers (live trading) + Paper Broker Service (integration testing)
- **Risk Management**: Position Intelligence Engine (PIE), Risk Engine with daily loss/position limits
- **Test Coverage**: 250+ unit tests, 100+ integration tests, E2E test suite
- **Phase Status**: Phase 10 (Production Hardening) — Load testing ✅, Performance tuning ✅, Chaos engineering ✅

## Services & Ports

| Service | Port | Database |
|---------|------|----------|
| Authentication Service | 8001 | `smarttrade_authentication_service` |
| Paper Broker Service | 8002 | `smarttrade_paper_broker_service` |
| Market Data Service (MDS) | 8004 | `smarttrade_market_data_service` |
| Broker Adapter Service (BAS) | 8005 | `smarttrade_broker_adapter_service` |
| Frontend | 5173 | — (served by Node.js) |

## Documentation Organization

This repository contains **cross-service** architecture and planning documents. Service-specific implementation documentation lives in each service's `docs/` directory to keep it close to the code and easy to maintain.

### When to Look Where

| What You Need | Where to Look |
|---------------|---------------|
| Overall architecture & phases | [smarttrade-architecture-v3.4-current.md](smarttrade-architecture-v3.4-current.md) (this repo) |
| BAS implementation details (Order State Machine, Orchestrator, etc.) | [`broker-adapter-service/docs/INDEX.md`](../broker-adapter-service/docs/INDEX.md) |
| MDS implementation | [`market-data-service/docs/`](../market-data-service/docs/) |
| Paper Broker Service implementation | [`paper-broker-service/docs/`](../paper-broker-service/docs/) |
| E2E test strategy | [`smarttrade-tests/docs/E2E_TESTING_STRATEGY.md`](../smarttrade-tests/docs/E2E_TESTING_STRATEGY.md) |
| Fyers API reference | [`broker-adapter-service/docs/fyers-api-reference/`](../broker-adapter-service/docs/fyers-api-reference/) |
| Deployment & infrastructure | [`smarttrade-deployment/`](../smarttrade-deployment/) |

### History

Archived documentation for completed phases (Phases 1-9) is available in each service's `docs/archive/` directory for reference. Do not implement from archived docs — use current docs in service `docs/` directories.
