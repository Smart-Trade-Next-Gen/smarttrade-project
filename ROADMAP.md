# SmartTrade Platform — Roadmap

Cross-service roadmap. Service-internal roadmaps (if any) live in each
service's own `docs/`.

For the target architecture see
[`FINAL_TARGET_ARCHITECTURE_v4.0.md`](FINAL_TARGET_ARCHITECTURE_v4.0.md).

---

## Current service status (snapshot)

| Service | Status |
|---------|--------|
| Authentication Service | Live |
| Broker Adapter Service (BAS) | Live — stateless execution kernel (post-refactor: no local order/position/risk persistence) |
| Market Data Service (MDS) | Live — Redis Streams + KV quote distribution, instrument master, option chain & Greeks |
| Paper Broker Service (PBS) | Live — execution engine consumes real MDS prices via `market.quote.v1` |
| Strategy Service | Live — consumes market + portfolio state, publishes `strategy.decision.v1` |
| Journal Service | Live — read-only event consumer; trades, orders, actions, FIFO open lots, journal entries |
| Portfolio Service | Live — read-only event consumer; positions, holdings, portfolio summary with live valuation |
| Notification Service | Live — user notification delivery |
| User Setting Service | Live |
| AI Service | In development |
| Frontend | Live — dual-WebSocket model (MDS for market data, BAS for account events) |

---

## Architecture inflection (completed)

These platform-level changes shape current work and replace older roadmap
items. They are listed here so contributors don't re-propose superseded
plans.

- **BAS stateless refactor** — removed local order / position / trade /
  execution-state tables, the Order State Machine, Outbox Processor,
  Execution Orchestrator, Risk Engine, and Settings/Symbol/PIE in-BAS
  components. Broker is the source of truth.
- **Read-side extraction** — order/trade history moved to Journal Service;
  positions / holdings / portfolio summary moved to Portfolio Service.
- **Strategy / Risk decoupling** — strategy logic and risk validation now
  live downstream of BAS.
- **Quote distribution** — MDS publishes durable `market.quote.v1` Redis
  Stream + KV snapshots; BAS, PBS, Portfolio, Strategy consume via consumer
  groups instead of WebSocket-direct.
- **WebSocket separation** — MDS WS carries market data only; BAS WS
  carries account events. UI must not relay account events via MDS.
- **Replicated instrument master** — MDS owns the master; other services
  replicate locally and consume `market.instrument.v1` for updates. See
  [`ARCHITECTURE_DECISION_RECORD_instrument_master.md`](ARCHITECTURE_DECISION_RECORD_instrument_master.md).

---

## Near-term focus

- **Production hardening across the new services** — Journal, Portfolio,
  Strategy, Notification: load testing, observability dashboards, alert
  rules.
- **Frontend dual-WebSocket completion** — fully migrate UI off any
  remaining MDS-relay path for account events.
- **Instrument master replication rollout** — finish wiring all consumers
  off runtime MDS REST calls.
- **OHLC / backtest data feed in MDS** — historical candles, multi-interval
  derivation, replay cursor.

## Medium-term

- **New broker plugin** (Zerodha first) reusing current Fyers plugin shape
  and MDS production patterns.
- **Advanced risk** — drawdown circuit breaker, sector / Greeks-based limits,
  rule CRUD via API.
- **Options enhancements** — payoff diagrams, IV percentile, OI analytics,
  unusual-activity scanner.
- **Multi-leg strategy execution** in Strategy Service.

## Longer-term

- **AI/ML assistance** — pattern recognition, support/resistance
  auto-detection, regime detection, trade summarization, pre-trade
  checklists (AI Service).
- **Analytics & journaling** — P&L heatmap, win rate by setup, MAE/MFE,
  execution-quality analysis, monthly report exports.
- **Scalability** — horizontal scaling per service, event sourcing where
  it pays off, read replicas for analytics, possible Kafka migration for
  high-throughput replay.
- **Platform** — multi-user / admin tools, subscription tiers, mobile PWA,
  external API SDK, webhooks.

## Backlog

- Additional brokers (Interactive Brokers, Angel, Dhan, Upstox).
- Kubernetes / Helm deployment, blue-green pipeline, secret-rotation
  automation.
- Algo marketplace, copy trading, multi-currency, fixed income, mutual funds.

---

## Release process

| Type | Cadence | Scope |
|------|---------|-------|
| Patch | Weekly | Bug fixes, minor improvements |
| Minor | Monthly | Backwards-compatible features |
| Major | Quarterly | Breaking changes, new services |

Every release requires passing tests, code review, staging smoke test, and
updated docs for any user-facing or contract change.
