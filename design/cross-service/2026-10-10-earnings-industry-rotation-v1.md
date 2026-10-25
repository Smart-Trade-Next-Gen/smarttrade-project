# Earnings Calendar, Post-Earnings Research, Industry Reports & Sector Rotation (HLD)

**Version**: 1.0
**Date**: 2026-10-10
**Status**: Draft — pending approval
**Services Affected**: ai-scoring-service (new earnings/RS/industry modules), market-data-service (read-only client usage — no code change), smarttrade-frontend (new Research tabs)

---

## Executive Summary

### What
An earnings-driven analytics layer inside ai-scoring-service:

1. **Earnings calendar** — a rotating screener-announcements sweep over the *full imported universe* produces a persistent per-company calendar (board meetings, declared results, estimated dates).
2. **Post-earnings research** — a newly detected results declaration for a *monitored* company (research-alert watchlist item or monitored holding) automatically enqueues a recheck research run with `from_run_id` provenance — mirroring the alert-triggered recheck pattern.
3. **Quarterly P&L in reports** — the screener dataset's `quarterly_results` table is normalized into the research report and surfaced in the UI.
4. **RS ratings** — a daily engine computes an IBD-style weighted relative-strength score per universe instrument vs NIFTY, percentile-ranked to a 0-100 rating.
5. **Industry reports** — deterministic per-industry reports: member tracker (price, RS, 52wk-high distance, latest-quarter growth) plus an aggregate quarterly earnings trend ("X/Y companies reported").
6. **Sector rotation** — daily industry/sector aggregate snapshots give RS-based rankings and rank trajectories.

### Why
- Results are the single biggest fundamental catalyst — today nothing tracks when monitored companies report, and nothing re-researches them on fresh numbers.
- Research reports contain quarterly-growth *metrics* but not the quarterly *table* investors actually read.
- There is no relative-strength or sector/industry view anywhere in the platform — discovery ranks candidates in isolation.

### Which Services
- **ai-scoring-service**: everything. New `research/earnings/`, `research/relative_strength/`, `research/industry/` modules, new tables, new REST routes, watcher/engine loops under the worker runtime role.
- **market-data-service**: consumed read-only via the existing fundamentals universe + history/download-pool APIs. No changes.
- **smarttrade-frontend**: Earnings/Industries/Sectors tabs on the Research page, quarterly P&L panel + RS badge in the report card.

### Success Criteria
1. `GET /api/v1/earnings/calendar` lists upcoming board-meeting dates (basis=ANNOUNCED), estimated dates (basis=ESTIMATED), and recently declared results for the universe.
2. A detected `RESULTS_DECLARED` filing for a monitored company enqueues one `source=EARNINGS` run per monitoring user/scope — with scope-toggle, cooldown, and in-flight dedupe guards identical in spirit to `enqueue_alert_recheck`.
3. Research reports carry a normalized `quarterly_results` block; the report card renders it.
4. `rs_ratings` covers every universe instrument with ≥63 sessions of history; ratings are percentiles, never fabricated.
5. `GET /api/v1/industries/{name}` returns a member tracker + aggregate quarterly earnings trend with an honest reported/total coverage counter.
6. `GET /api/v1/sectors` returns rotation rankings with rank deltas from `breadth_snapshots` history.

---

## Architecture Diagram

```
ai-scoring-service
┌──────────────────────────────────────────────────────────────────┐
│ worker role (RESEARCH_RUNTIME_ROLE ∈ {worker, standalone})       │
│                                                                   │
│  EarningsWatcher ── sequential screener polls, paced by           │
│   earnings_poll_state (monitored tier 12h; universe tier 14d)     │
│     │ results declared ──► earnings_calendar (DECLARED)           │
│     │                    ► get_company_dataset → company_quarterly│
│     │                    ► EarningsService.maybe_enqueue_recheck  │
│     │ board meeting    ──► earnings_calendar (ANNOUNCED)          │
│     │ forward_outlook  ──► earnings_calendar (ESTIMATED)          │
│                                                                   │
│  RsEngine ── daily: MDS download pool → get_history 1d (~260s)    │
│     weighted rel. return vs NSE:CASH:INDEX:NIFTY → percentile     │
│     → rs_ratings (+ rs_rating_history)                            │
│     → breadth_snapshots (daily industry+sector aggregates)    │
│                                                                   │
│ api role                                                          │
│  /api/v1/earnings/*  /api/v1/industries/*  /api/v1/sectors/*      │
│  /api/v1/rs/*                                                     │
└──────────────────────────────────────────────────────────────────┘
          ▲ read-only                                ▲ read-only
   screener-mcp /api/* (announcements, datasets,     market-data-service
    forward-outlook)                                  fundamentals universe +
                                                      history/download pool
```

No new cross-service events in v1: post-earnings recheck is intra-service. A future `scoring.earnings.detected` domain event can feed notification-service ("INFY reported") once the calendar proves itself.

---

## Data Model (ai-scoring-service, migration 020)

### `earnings_poll_state`
| Column | Type | Notes |
|---|---|---|
| company_id | str PK | canonical MDS company UUID (or screener symbol when unmapped) |
| instrument_id | str | primary mapped instrument |
| screener_symbol | str, nullable | |
| tier | str | MONITORED \| UNIVERSE |
| last_polled_at | datetime | |
| next_poll_at | datetime, indexed | sweep ordering |
| error_count | int | exponential backoff driver |

### `earnings_calendar`
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| company_id | str, indexed | |
| instrument_id | str, indexed | |
| screener_symbol | str, nullable | |
| company_name | str | |
| event_type | str | RESULTS_DECLARED \| BOARD_MEETING \| RESULTS_EXPECTED \| DIVIDEND \| AGM \| OTHER_MATERIAL |
| fiscal_period | str, nullable | e.g. "Q1-FY27" |
| results_date | date, nullable | scheduled or declared date (parsed from headline) |
| results_date_text | str, nullable | raw date text when unparseable |
| announced_on | date, nullable | filing date |
| basis | str | ANNOUNCED \| ESTIMATED \| DECLARED |
| headline | str | source filing headline |
| url | str, nullable | |
| detected_at | datetime | |

Unique: `(company_id, event_type, fiscal_period)` — partial on `fiscal_period IS NOT NULL`.

### `company_quarterly`
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| company_id | str | + fiscal_period unique |
| fiscal_period | str | "Q1-FY27" |
| period_label | str | e.g. "Jun 2026" |
| sales / operating_profit / net_profit | Numeric | crore, Decimal |
| opm_pct / eps | Numeric | |
| extra | JSONB | other_income, interest, depreciation, tax… |
| source | str | "screener" |
| updated_at | timestamp | |

### `rs_ratings`
`instrument_id` PK, `rs_score` (Decimal), `rs_rating` (int 0-100), `window_returns` JSONB ({r63,r126,r189,r252, benchmark}), `computed_at`.

### `rs_rating_history`
`(instrument_id, computed_at)` PK, `rs_rating` — powers trend/"days" columns.

### `breadth_snapshots`
`(granularity, name, computed_at)` unique, `member_count`, `avg_rs`, `pct_above_50`, `agg` JSONB (median PE, total mcap, top names). `granularity` = `industry` | `industry_group` — one table serves both the industry detail page and sector-rotation history.

---

## Module Design

### `research/earnings/`

- **`classifier.py`** — pure functions over announcement payloads:
  - `classify_event(announcement) -> (event_type, fiscal_period, results_date, results_date_text)`
  - Reuses `_EVENT_CATEGORIES`/`_EVENT_DATE_RE` logic from `analysis/news.py`; adds real multi-format Indian filing-date parsing (`25 Oct 2025`, `25-Oct-2025`, `25/10/2025`, `25th October, 2025`) and quarter inference (`quarter ended June 30, 2026` → `Q1-FY27`; `Q1`/`first quarter` tokens).
- **`watcher.py`** — `EarningsWatcher` asyncio loop, started where `research_jobs` runs:
  - Per tick: `SELECT … WHERE next_poll_at <= now ORDER BY tier, next_poll_at LIMIT EARNINGS_POLL_BATCH` → sequential polls with `EARNINGS_POLL_DELAY_SECONDS` between calls.
  - Tier resolution: company has an enabled watchlist item or monitored holding → MONITORED (`EARNINGS_MONITOR_POLL_HOURS` cadence); else UNIVERSE (`EARNINGS_UNIVERSE_SWEEP_DAYS` spread).
  - `get_announcements(symbol, days=EARNINGS_ANNOUNCEMENT_LOOKBACK_DAYS)` → classifier → upsert `earnings_calendar`; new RESULTS_DECLARED → `get_company_dataset` quarterly harvest → `company_quarterly` upsert → `EarningsService.maybe_enqueue_recheck`.
  - `get_forward_outlook` on the slower `EARNINGS_FORWARD_OUTLOOK_REFRESH_DAYS` cadence → RESULTS_EXPECTED rows (ESTIMATED basis).
  - Error path: `error_count++`, `next_poll_at = now + min(2^n × base, EARNINGS_RECHECK_BACKOFF_CAP_DAYS)`; never aborts the sweep.
- **`service.py`** — `EarningsService`:
  - `calendar_query(from, to, scope)` / `company_timeline(instrument_id)` / `quarterly(company_id)` read paths for routes.
  - `maybe_enqueue_recheck(company)` — for each monitoring (user, scope) pair: watchlist (enabled item + `monitor_configs.watchlist_enabled`) or holdings (`holdings_enabled` + live holding via `_fetch_monitor_holding`, not excluded):
    - cooldown: skip if an EARNINGS-sourced run for the company within `EARNINGS_RECHECK_MIN_MINUTES`;
    - `enqueue_monitor_run(source=EARNINGS, from_run_id=<latest completed run>, same profile, monitor_context)` — built-in active-run dedupe applies.
  - Sold-holding parity with the alert path: no live holding → skip, publish no research (alert rules stay as-is; the price-alert path owns clears).

### `research/relative_strength/engine.py`

Daily loop (worker):
1. Universe = primary instrument mappings → `request_price_download` (1d) batch enqueues (pool paces broker limits); `get_history` after warm-up.
2. Benchmark: `NSE:CASH:INDEX:NIFTY` (`RS_BENCHMARK_INSTRUMENT`).
3. `rs_score = (0.4·r63 + 0.2·r126 + 0.2·r189 + 0.2·r252) − same(benchmark)`; percentile-rank across scored instruments → `rs_rating` 0-100.
4. Upsert `rs_ratings` + append `rs_rating_history`; write `breadth_snapshots` aggregates (both granularities).
5. Instruments with < `RS_MIN_HISTORY_SESSIONS` (63) candles → no row; coverage reported in API metadata — never a fabricated rating.

### `research/industry/service.py`

Read-time aggregation (no extra jobs — RS engine already writes daily snapshot rows):
- `GET /api/v1/industries` — industry list: member_count, avg_rs, median P/E, total mcap, coverage.
- `GET /api/v1/industries/{name}` — **IndustryReport**:
  - `members[]`: name, instrument_id, price, rs_rating, `dist_52w_high_pct` (snapshot `high_price`), `yoy_quarterly_sales_growth` / `…profit_growth`, mcap, P/E.
  - `earnings_trend[]`: per fiscal period — Σ sales / Σ op_profit / Σ PAT, median growth, `reported_count/member_count`.
  - `stats`: aggregates.
- `GET /api/v1/sectors` — industry_groups ranked by avg_rs + rank delta vs `breadth_snapshots` N days back; `GET /api/v1/sectors/{name}/history` — trajectory.

---

## Research Source & Report Contract

- `ResearchSource.EARNINGS` added — provenance only; RunsPanel badges it like other sources. Recheck runs keep the origin's `research_profile`/`monitor_context` and carry `from_run_id`.
- `FundamentalAnalysis.quarterly_results: list[QuarterlyEarningRow]` — `{period_label, sales, operating_profit, opm_pct, pat, eps, sales_yoy_pct, pat_yoy_pct, opm_yoy_pp}` — normalized in `analysis/fundamentals.py` from `dataset.quarterly_results` (already collected today; extraction only).
- Company list rows gain `rs_rating` when a ratings row exists (LEFT JOIN semantics — absent, never 0).

## REST API

| Endpoint | Purpose | RBAC |
|---|---|---|
| `GET /api/v1/earnings/calendar?from=&to=&scope=&limit=` | upcoming + declared events | research.read |
| `GET /api/v1/earnings/company/{instrument_id}` | timeline + quarterly table | research.read |
| `POST /api/v1/earnings/refresh/{instrument_id}` | force repoll | research.compute |
| `POST /api/v1/earnings/refresh` | kick sweep | research.compute |
| `GET /api/v1/rs/{instrument_id}` | current rating + windows | research.read |
| `GET /api/v1/rs?industry=&min=` | ratings list | research.read |
| `POST /api/v1/rs/refresh` | kick engine | research.compute |
| `GET /api/v1/industries` | industry list + aggregates | research.read |
| `GET /api/v1/industries/{name}` | industry report | research.read |
| `GET /api/v1/sectors` | rotation ranking | research.read |
| `GET /api/v1/sectors/{name}/history` | rank trajectory | research.read |

## Config (new settings)

`EARNINGS_WATCH_ENABLED` (True), `EARNINGS_POLL_DELAY_SECONDS` (2.0), `EARNINGS_POLL_BATCH` (200), `EARNINGS_MONITOR_POLL_HOURS` (12), `EARNINGS_UNIVERSE_SWEEP_DAYS` (14), `EARNINGS_ANNOUNCEMENT_LOOKBACK_DAYS` (90), `EARNINGS_FORWARD_OUTLOOK_REFRESH_DAYS` (7), `EARNINGS_RECHECK_ENABLED` (True), `EARNINGS_RECHECK_MIN_MINUTES` (720), `EARNINGS_RECHECK_BACKOFF_CAP_DAYS` (7),
`RS_ENGINE_ENABLED` (True), `RS_COMPUTE_INTERVAL_HOURS` (24), `RS_BENCHMARK_INSTRUMENT` (`NSE:CASH:INDEX:NIFTY`), `RS_BROKER_ID` (str, optional), `RS_ACTING_USER_ID` (str, optional), `RS_SESSIONS` (260), `RS_MIN_HISTORY_SESSIONS` (63).

## Error Codes

- `EARN_001` — earnings/RS/industry resource not found
- `EARN_002` — invalid query params (scope, date range)

## Frontend

- `api/researchApi.ts` + `types`: earnings calendar, industry report, sector ranking, rs rating.
- `pages/Research.tsx`: new tabs — **Earnings**, **Industries**, **Sectors**.
- `components/research/`:
  - `EarningsCalendarPanel.tsx` — upcoming table (date, company, basis badge) + just-reported feed + "Re-research now" + watchlist badge.
  - `IndustriesPanel.tsx` → `IndustryDetailPanel.tsx` — tracker table + earnings-trend bars (`lightweight-charts`).
  - `SectorsPanel.tsx` — ranking + Δ-rank + sparkline.
  - `InvestorReportCard.tsx` — Quarterly P&L table + RS badge.
  - `CompaniesPanel.tsx` — RS column.

## Dependencies & Sequencing

1. This design doc → approval → commit (+ service-repo symlinks optional).
2. Migration 020 + models + contracts + config.
3. **Phase 1**: earnings classifier + watcher + service + routes + Earnings tab.
4. **Phase 2**: quarterly-results extraction + RS engine + rs routes + report UI/RS columns.
5. **Phase 3**: industry/sector aggregation + routes + Industries/Sectors tabs.
6. Docs sync + verification.

## Testing Strategy

- Unit: classifier (date formats, quarter inference, category mapping), quarterly normalization (dataset → `company_quarterly` rows), RS percentile math (synthetic candles, boundary windows), industry aggregation (Σ + coverage), recheck guard matrix (scope off / sold holding / cooldown / dedupe / disabled master switch), poll pacing & error backoff.
- Routes: calendar/industry/sector GETs, refresh RBAC, scope filters.
- `uv run pytest`, `uv run ruff check`, `uv run mypy`; migration upgrade/downgrade on SQLite; single head.
- Frontend: `npm run lint`, `npm run build`.
- E2E (docker): seed a monitored company → watcher poll → fabricated RESULTS_DECLARED → `EARNINGS` run enqueued; calendar reflects it.

## Risks / Constraints

- **Screener rate limits**: announcements are the only earnings-date source; sweep budget (~universe/14 per day + monitored 12h + 2s pacing) keeps upstream load light. Backoff prevents thundering herds.
- **Filing-date parsing is fuzzy**: `basis` (ANNOUNCED/ESTIMATED/DECLARED) keeps provenance honest; unparseable dates stored as text, never guessed.
- **RS is broker-bound**: `RS_BROKER_ID`/`RS_ACTING_USER_ID` required for MDS history; missing history ⇒ no row (coverage surfaced).
- **Industry earnings trend accrues organically**: `company_quarterly` fills as results are declared; coverage counter shows `X/Y reported` — a follow-up backfill sweep is noted but out of v1 scope.
- **Multi-user rechecks**: each monitoring user gets their own recheck (runs are per-acting-user, consistent with monitor runs).
