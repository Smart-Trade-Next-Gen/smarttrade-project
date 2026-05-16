# SmartTrade Project — Documentation Index

Cross-service architecture, design records, and platform planning for the
SmartTrade trading platform. Service-specific design and implementation
documentation lives in each service's own repository.

## Start here

| Document | Description |
|----------|-------------|
| **[FINAL_TARGET_ARCHITECTURE_v4.0.md](FINAL_TARGET_ARCHITECTURE_v4.0.md)** | Authoritative system-level architecture (v4.0, 2026-04-20). Execution / async plane separation, 9 services, event taxonomy, extraction roadmap. |
| [ROADMAP.md](ROADMAP.md) | Cross-service roadmap. |
| [CLAUDE.md](CLAUDE.md) | Working conventions for this design repo. |

## Instrument master architecture (active)

| Document | Description |
|----------|-------------|
| [ARCHITECTURE_DECISION_RECORD_instrument_master.md](ARCHITECTURE_DECISION_RECORD_instrument_master.md) | ADR: replicated instrument master + snapshot bootstrap. |
| [DESIGN_smarttrade_common_instrument_master.md](DESIGN_smarttrade_common_instrument_master.md) | Design doc for `smarttrade_common.instrument_master`. |
| [IMPLEMENTATION_SUMMARY_instrument_master.md](IMPLEMENTATION_SUMMARY_instrument_master.md) | Implementation snapshot. |

## Cross-service designs

- [`design/cross-service/`](design/cross-service/) — multi-service feature designs.

## Service-specific documentation (in each service repo)

| Service | Location |
|---------|----------|
| Broker Adapter Service (BAS) | [`../broker-adapter-service/docs/INDEX.md`](../broker-adapter-service/docs/INDEX.md) |
| Market Data Service (MDS) | [`../market-data-service/docs/`](../market-data-service/docs/) |
| Paper Broker Service (PBS) | [`../paper-broker-service/docs/`](../paper-broker-service/docs/) |
| Journal Service | [`../journal-service/`](../journal-service/) |
| Portfolio Service | [`../portfolio-service/`](../portfolio-service/) |
| Strategy Service | [`../strategy-service/`](../strategy-service/) |
| Notification Service | [`../notification-service/`](../notification-service/) |
| Authentication Service | [`../authentication-service/`](../authentication-service/) |
| User Setting Service | [`../user-setting-service/`](../user-setting-service/) |
| AI Service | [`../ai-service/`](../ai-service/) |
| Frontend | [`../smarttrade-frontend/`](../smarttrade-frontend/) |
| Shared library | [`../smarttrade-common/docs/`](../smarttrade-common/docs/) |
| Deployment | [`../smarttrade-deployment/`](../smarttrade-deployment/) |
| E2E tests | [`../smarttrade-tests/docs/`](../smarttrade-tests/docs/) |

## Services & ports

(Host-side ports per `smarttrade-deployment/docker-compose.yml`.)

| Service | Host port | Database |
|---------|-----------|----------|
| Authentication Service | 8001 | `smarttrade_authentication_service` |
| Paper Broker Service (PBS) | 8002 | `smarttrade_paper_broker_service` |
| Market Data Service (MDS) | 8004 | `smarttrade_market_data_service` |
| Broker Adapter Service (BAS) | 8005 | `smarttrade_broker_adapter_service` |
| Strategy Service | 8006 | `smarttrade_strategy_service` |
| Journal Service | 8007 | `smarttrade_journal_service` |
| Portfolio Service | 8008 | `smarttrade_portfolio_service` |
| Notification Service | 8011 | `smarttrade_notification_service` |
| Frontend | 5173 | — |

## When to look where

| What you need | Where to look |
|---------------|---------------|
| System architecture | [FINAL_TARGET_ARCHITECTURE_v4.0.md](FINAL_TARGET_ARCHITECTURE_v4.0.md) |
| Service-internal design / LLDs | each service's `docs/` |
| Cross-service feature design | [`design/cross-service/`](design/cross-service/) |
| Event contracts | [`../contracts/`](../contracts/) |
| Deployment / infra | [`../smarttrade-deployment/`](../smarttrade-deployment/) |
| E2E test strategy | [`../smarttrade-tests/docs/`](../smarttrade-tests/docs/) |

## History

Historical phase plans, alignment requirements, and earlier architecture
versions (v3.2, v3.4) have been deleted. Git history preserves them if you
need to look them up.
