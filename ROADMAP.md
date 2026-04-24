# SmartTrade Platform — Product Roadmap

**Last Updated:** 2026-04-18 (MDS Phase 0-4 alignment added)

---

## Current Status

| Service | Status | Coverage |
|---------|--------|---------|
| Authentication Service | Production Ready ✅ | Full test suite |
| Broker Adapter Service | Production Ready ✅ | 250+ tests |
| Market Data Service (Real-time) | Production Ready ✅ | Quotes, instruments, WebSocket |
| Market Data Service (OHLC/Backtest) | Phase 0-4 In Progress 🔄 | PostgreSQL schema, aggregation, determinism |
| Mock Service | Production Ready ✅ | Mirrors BAS |
| Frontend | In Progress 🔄 | Phase 5: API integration |
| PIE Engine | Production Ready ✅ | Full lifecycle tested |
| Position Management | Production Ready ✅ | Group rules + exit |
| Options (Greeks/IV) | Production Ready ✅ | BS calculator + vol surface |
| Risk Engine | Production Ready ✅ | 3 rule types, YAML-driven |
| Settlement (T+1) | Production Ready ✅ | Full lifecycle |

---

## Q1 2026 (Jan–Mar) — Foundation Complete ✅

### Completed
- [x] Core microservices architecture (Auth, BAS, MDS, Mock)
- [x] `smarttrade-common` shared library (auth, events, DB, resilience, observability)
- [x] Fyers broker plugin (REST + WebSocket)
- [x] Paper trading plugin (Mock Service)
- [x] Order lifecycle (placement → fill → settlement T+1)
- [x] Risk Engine (daily loss, position limits, per-trade risk)
- [x] PIE Engine (auto-entry, kill switch, strategy execution, action orchestrator)
- [x] Position Management (groups, rules, aggregated P&L)
- [x] Options (chain, Greeks calculator, IV metrics, volatility surface)
- [x] Frontend Phase 1–4 (dashboard, chart, orders, positions panels)
- [x] Frontend Phase 5.1 (API endpoint mapping, axios clients)
- [x] 250+ tests passing across all services

---

## Q2 2026 (Apr–Jun) — Frontend Integration & Production Hardening

### Phase 5: Frontend → Backend Integration (Apr 2026)

**5.2 WebSocket Integration**
- [ ] Real-time order updates via BAS WebSocket
- [ ] Real-time position updates from BAS events
- [ ] PIE status feed (strategy state, kill switch alerts)
- [ ] Risk alert notifications via WS
- [ ] Reconnection logic with exponential backoff

**5.3 Type Safety & API Contract Alignment**
- [ ] Align TypeScript types with Pydantic schemas
- [ ] Auto-generate TypeScript types from OpenAPI spec
- [ ] Runtime response validation with Zod

**5.4 Complete API Implementation**
- [ ] Connect all panel components to real APIs
- [ ] Order panel: live order book + order submission
- [ ] Positions panel: real P&L from BAS
- [ ] Risk panel: live snapshot from RiskEngine
- [ ] PIE dashboard: real strategy state + auto-entry controls
- [ ] Options chain: live data from MDS WebSocket

**5.5 Frontend Testing**
- [ ] Vitest unit tests for stores and hooks
- [ ] Playwright E2E: login → order placement → position update
- [ ] Playwright E2E: PIE strategy activation → kill switch
- [ ] WebSocket reconnection tests

### MDS Phase 0-4 Implementation (Parallel: Apr–Jun 2026)

**Phase 0: Foundation (Weeks 1-2, Apr)**
- [ ] PostgreSQL schema (partitioned historical_candles table with idempotency keys, TTL indexes)
- [ ] Trading calendar seeding (market hours, holidays, expected_candles_per_day)
- [ ] Bucket-scoped tick buffer (per-(symbol, bucket_start) isolation, no cross-contamination)
- [ ] Time-driven finalization scheduler (runs every 60s, independent of tick arrival)
- [ ] Memory management service (TTL cleanup for finalized buckets, stale symbol pruning)
- [ ] Determinism test suite (20+ tests: same input → identical output, verified 10 times)
- [ ] Memory leak tests (verify TTL cleanup prevents unbounded growth)

**Phase 1: Historical Data Feed (Weeks 3-4, Apr–May)**
- [ ] Broker daily OHLC backfill (Fyers, Paper Broker)
- [ ] Gap detection & logging (missing candles, broker outages)
- [ ] Data quality metrics & validation (outliers, volume spikes)
- [ ] Integration tests (backfill scenarios, gap detection)

**Phase 2: Multi-Interval Derivation (Weeks 5-6, May)**
- [ ] 5m candle derivation from 1m (exact boundary conditions)
- [ ] 15m/1h/1d derivation (watermark-based finalization)
- [ ] Idempotent derived candle inserts (SHA256 idempotency keys)
- [ ] Interval boundary tests (verify completeness before derivation)

**Phase 3: IV & Greeks Enhancement (Weeks 7-8, May–Jun)**
- [ ] Config-driven IV calculation (no hardcoded parameters)
- [ ] Real-time IV surface updates (on quote refresh)
- [ ] Multi-leg Greeks support (option spreads: bull call, iron condor, etc.)
- [ ] Volatility skew/smile detection
- [ ] Greeks accuracy tests (Black-Scholes edge cases)

**Phase 4: Backtest Data Feed (Week 9, Jun)**
- [ ] Backtest API: `/api/v1/data/ohlc?symbol=SBIN-EQ&interval=5m&from=2025-01-01`
- [ ] Replay cursor abstraction (seek, peek_next, progress, is_complete)
- [ ] Corporate action application (dividends, splits, bonus adjustments)
- [ ] Gap rejection for incomplete data sets
- [ ] Replay cursor tests (deterministic seek, progress tracking)

**Total Effort**: ~400 hours (8 weeks parallel to Frontend Phase 5)  
**Blocking Dependencies**: None (parallel track)  
**Go-Live Gate**: Must complete before Q3 production trading launch

### Production Hardening (May–Jun 2026)

**Infrastructure**
- [ ] nginx reverse proxy config (SSL termination, WebSocket proxy)
- [ ] Docker Compose production config (resource limits, health checks)
- [ ] PostgreSQL connection pooling (PgBouncer)
- [ ] Redis Sentinel for HA

**Observability**
- [ ] Grafana dashboard templates (latency, order flow, risk)
- [ ] Prometheus alerting rules (circuit breaker open, P99 > 500ms)
- [ ] Error rate alerts (Sentry integration)
- [ ] PagerDuty integration for critical risk alerts
- [ ] MDS memory metrics (mds_tick_buffers_count, mds_finalized_buckets_count, mds_active_symbols_count)
- [ ] MDS latency tracking (bucket finalization duration, late tick arrival rates)
- [ ] Circuit breaker state monitoring (tick ingestion protection)

**MDS Infrastructure**
- [ ] PostgreSQL partitioning strategy (historical_candles partitioned by month)
- [ ] Redis Streams consumer group setup (backtest data subscription)
- [ ] Scheduler resource allocation (time-driven finalization loop, 60s interval)
- [ ] Load testing: 10K ticks/sec with circuit breaker active
- [ ] Memory profiling: verify TTL cleanup prevents unbounded growth over 24h continuous run

**Security Hardening**
- [ ] API key rotation procedure
- [ ] Secret scanning in CI (truffleHog)
- [ ] Penetration test (OWASP Top 10 coverage)
- [ ] Credential rotation automation for broker API keys

---

## Q3 2026 (Jul–Sep) — New Broker + Advanced Features

### Zerodha Broker Integration
- [ ] Zerodha plugin (implements `BrokerPlugin` interface)
- [ ] Kite Connect REST API mapper (order DTOs ↔ Kite)
- [ ] Zerodha WebSocket adapter (KiteTicker)
- [ ] Zerodha OAuth flow (routes_oauth extension)
- [ ] Instrument mapping for NSE/BSE/NFO via Kite instruments CSV
- [ ] E2E tests with Kite sandbox

**NOTE**: Zerodha integration will follow MDS v2.1 production-grade patterns established in Q2:
- Broker-specific tick ingestion plugin with bucket-scoped buffering
- Deterministic idempotency keys (SHA256-based)
- Same late tick handling policy (discard/log_only)
- Reuse Phase 0 determinism test suite for Zerodha plugin validation

### Advanced Risk Features
- [ ] Intraday drawdown circuit breaker (% drawdown from daily high)
- [ ] Sector exposure limits (cap exposure to any one sector)
- [ ] Greeks-based risk: total delta limit, total vega limit
- [ ] Portfolio heat map (visual risk concentration)
- [ ] Risk rule CRUD via API (dynamic rule management, not just YAML)

### Advanced PIE Features
- [ ] Multi-leg strategy support (enter/exit multiple instruments atomically)
- [ ] Time-based triggers (enter at market open, exit before close)
- [ ] P&L target exit (auto-exit on profit target or stop-loss)
- [ ] Strategy templates library (common option strategies)
- [ ] PIE backtesting mode (simulate strategy on historical data)

### Options Enhancements
- [ ] Options strategy payoff diagram (real-time)
- [ ] IV percentile charts (historical IV vs current)
- [ ] Put/call ratio tracking
- [ ] Open interest analytics
- [ ] Options scanner (unusual activity detection)

---

## Q4 2026 (Oct–Dec) — Intelligence & Scale

### AI/ML Trading Assistance
- [ ] Trade setup validator (pattern recognition on chart)
- [ ] Support/resistance auto-detection (price action analysis)
- [ ] Trend forecast (LSTM model, 1-5 candle ahead)
- [ ] Market regime detection (trending vs ranging)
- [ ] Trade notes AI summarization
- [ ] Pre-trade checklist scoring

### Analytics & Journaling
- [ ] Trade journal (screenshot capture + annotate)
- [ ] P&L heatmap (day × hour)
- [ ] Win rate by setup type
- [ ] MAE/MFE analysis per trade
- [ ] Execution quality analysis (slippage, fill vs limit)
- [ ] Monthly performance report (PDF export)

### Scalability
- [ ] Horizontal scaling: BAS stateless with shared Redis session
- [ ] Event sourcing for order state (append-only event log)
- [ ] Read replicas for portfolio/analytics queries
- [ ] Message queue migration: Redis → Kafka (for high-throughput replay)
- [ ] Multi-region deployment design

### Platform
- [ ] Multi-user support (admin panel, user management)
- [ ] Subscription tiers (feature flags per tier)
- [ ] Mobile PWA (responsive layout, offline mode)
- [ ] API SDK (Python client library for SmartTrade API)
- [ ] Webhook support (notify external systems on order events)

---

## Backlog (Future)

### Broker Integrations
- [ ] Interactive Brokers (TWS API)
- [ ] Angel Broking (Smart API)
- [ ] Dhan
- [ ] Upstox

### Infrastructure
- [ ] Kubernetes deployment (Helm charts)
- [ ] Blue/green deployment pipeline
- [ ] Database migration automation (Alembic in CI/CD)
- [ ] Secrets management (HashiCorp Vault)

### Features
- [ ] Algo marketplace (share/sell strategies)
- [ ] Copy trading
- [ ] Multi-currency support
- [ ] Fixed income / bonds support
- [ ] Mutual fund integration (via RTA APIs)

---

## Feature Flags

Features in progress use feature flags managed via preferences or environment variables to enable gradual rollout:

| Flag | Default | Description |
|------|---------|-------------|
| `ENABLE_PIE` | `true` | PIE strategy automation |
| `ENABLE_OPTIONS` | `true` | Options chain + Greeks |
| `ENABLE_PAPER_TRADING` | `true` | Paper trading mode |
| `ENABLE_KILL_SWITCH` | `true` | Emergency kill switch |
| `ENABLE_SENTRY` | `false` (local) | Error tracking |
| `ENABLE_PROMETHEUS` | `true` | Metrics export |

---

## Release Cadence

| Release Type | Frequency | Scope |
|-------------|-----------|-------|
| Patch | Weekly | Bug fixes, minor improvements |
| Minor | Monthly | New features, non-breaking changes |
| Major | Quarterly | Breaking changes, new services |

All releases require:
1. All tests passing (unit + integration)
2. PR reviewed and approved
3. Staging deployment + smoke test
4. Design doc updated (for Major/Minor)
