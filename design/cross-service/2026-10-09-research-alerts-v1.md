# Research-Driven Alerts Engine (HLD)

**Version**: 1.0
**Date**: 2026-10-09
**Status**: Approved — implemented
**Services Affected**: portfolio-service (alerts engine — new module), ai-scoring-service (alert-intent producer), smarttrade-common (event schemas), notification-service (stream/category classification), smarttrade-frontend (alerts UI + research-table upgrades)

---

## Executive Summary

### What
A real alerting subsystem: user-managed alert **rules** (manual + research-sourced) evaluated against live market quotes and research lifecycle events, firing `alerts.triggered` domain events that notification-service relays to users. Completed research runs on watchlist/holdings names **auto-sync** a desired alert set — entry, exit, and rebalance alerts are added, updated, or deactivated as new research lands.

### Why
- Research currently produces advice (`position_advice`, `rebalance`) that goes stale in a report — nothing acts on it.
- Users need actionable alerts: "tell me when XYZ reaches the entry zone the research identified", "warn me when a holding's advice flips to exit", "alert me when a rebalance is suggested".
- No alert engine exists today: `AlertManager` is a hardcoded demo, Smart Exit only covers exits on open positions (not watchlist names), notification-service is a pure relay.

### Which Services
- **portfolio-service**: new `position_management/alerts` module — rule store, evaluators, sync reconciliation, quote evaluation, REST API.
- **ai-scoring-service**: derives the desired alert set per completed watchlist/holdings run and publishes `alerts.sync` events; emits clears on unwatch/unmonitor/master-toggle-off.
- **smarttrade-common**: `alerts.sync` + `alerts.triggered` event schemas in the centralized catalog.
- **notification-service**: `alerts.*` stream/category classification entries.
- **smarttrade-frontend**: "Alerts" tab on the Research page, real `AlertManager` panel, plus per-column filters and holdings bulk operations on the research tables.

### Success Criteria
1. Users can create/pause/delete manual price alerts (above/below, pct move) via API and UI.
2. Completed WATCHLIST/HOLDINGS research runs auto-reconcile research-sourced alert rules (add/update/deactivate) with per-rule provenance (run id + reason).
3. Price rules evaluate on `market.quote` ticks; advice/rebalance rules evaluate on `alerts.sync` payloads.
4. Every trigger produces an `alerts.triggered` event reaching the user's notification WebSocket.
5. Alert evaluation is idempotent; rules have cooldown/one-shot semantics; user-deleted research rules are never resurrected by sync.
6. Instruments with active price alerts are upstream-subscribed via the MDS lease mechanism when a broker context is known.

---

## Architecture Diagram

```
ai-scoring-service                              portfolio-service
┌──────────────────────────┐                   ┌────────────────────────────────────┐
│ ResearchService          │                   │ position_management/alerts/        │
│  submit_analysis() ──┐   │                   │                                    │
│  (run → COMPLETED)   │   │                   │  ┌──────────┐   ┌───────────────┐  │
│  watchlist delete    │   │  alerts.sync      │  │ AlertRule │   │ AlertTrigger  │  │
│  holding exclusion   ├───┼─────────────────► │  │  table    │   │  history      │  │
│  monitor toggle off  │   │  (desired set)    │  └──────────┘   └───────────────┘  │
│  alert_intents.py    │   │                   │       ▲                ▲           │
└──────────────────────────┘                   │  sync_service    evaluation_service│
                                               │       ▲                ▲           │
                                               │  alert_consumer    on_quote()      │
                                               │  (EventBus)            │           │
MDS                                            │  subscription_service  │           │
 market.subscription.request ◄──────────────── │  (lease-renewed subs)  │           │
 market.quote ────────────────────────────────►│  PortfolioMarketDataConsumer        │
                                               └──────────┬─────────────────────────┘
                                                          │ alerts.triggered
                                        ┌─────────────────┴──────────────────┐
                                        ▼                                    ▼
                             notification-service              ai-scoring-service
                             (@subscribe('*')                  (alert_triggered_consumer
                             → classified → user               — research price alerts
                              notification WS)                  enqueue a recheck run)
```

---

## Event Schemas (smarttrade-common)

New `events/schemas/domain/alerts/alerts_events.py`; registered in `event_catalog.py`.

### `alerts.sync` — owner: ai-scoring-service

Published when a WATCHLIST- or HOLDINGS-sourced run completes, or when a monitored company leaves monitoring scope. Carries the full **desired** research alert set for one (user, instrument) — reconciliation is the consumer's job.

```python
class AlertIntentItem(BaseModel):
    key: str          # stable per (instrument, purpose): advice_watch | rebalance_watch
                      # | entry_zone | breakout_entry | exit_target | exit_stop | risk_floor
    kind: str         # PRICE_ABOVE | PRICE_BELOW | PCT_MOVE | ADVICE_CHANGE | REBALANCE_SIGNAL
    params: dict      # e.g. {"level": "412.50"} — Decimal as string
    reason: str       # human-readable evidence, shown in UI

class AlertsSyncV1(DomainEventBase):
    event_name: str = "alerts.sync"
    category: str = "ALERTS"

    research_run_id: Optional[str]      # null on clear events
    instrument_id: str
    screener_symbol: Optional[str]
    company_name: Optional[str]
    run_source: Optional[str]           # WATCHLIST | HOLDINGS | None on clear
    research_profile: Optional[str]
    broker_id: Optional[str]            # monitor-config broker (for quote leases)
    account_id: Optional[str]
    position_advice: Optional[dict]     # {action, conviction, horizon}
    rebalance_suggested: bool = False
    desired_alerts: List[AlertIntentItem] = []
    clear: bool = False                 # true → deactivate all research rules for (user, instrument)
    event_at: str                       # ISO8601
```

### `alerts.triggered` — owner: portfolio-service

```python
class AlertsTriggeredV1(DomainEventBase):
    event_name: str = "alerts.triggered"
    category: str = "ALERTS"

    rule_id: str
    instrument_id: str
    kind: str                           # PRICE_ABOVE | ... | REBALANCE_SIGNAL
    source: str                         # MANUAL | RESEARCH
    trigger_value: str                  # value that crossed (Decimal as string)
    context: dict                       # {ltp, level} | {pct, baseline} | {advice_from, advice_to} | {rebalance}
    reason: Optional[str]               # research intent reason or manual note
    source_run_id: Optional[str]        # research provenance
    triggered_at: str
```

Idempotency: publish key `alerts.triggered:{rule_id}:{minute_bucket}` for quote rules; `alerts.triggered:{rule_id}:{run_id}` for event rules.

---

## Data Model (portfolio-service)

### `alert_rules`

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | UUIDMixin |
| user_id | UUID, indexed | multi-tenancy |
| instrument_id | str, indexed | MDS canonical id |
| kind | str | PRICE_ABOVE/PRICE_BELOW/PCT_MOVE/ADVICE_CHANGE/REBALANCE_SIGNAL |
| params | JSONB | {level} or {pct} etc.; Decimals as strings |
| source | str | MANUAL | RESEARCH |
| source_key | str, nullable | dedupe key from AlertIntentItem.key (research) |
| source_run_id | str, nullable | producing research run |
| reason | str, nullable | intent reason / user note |
| broker_id | str, nullable | for quote lease requests |
| account_id | str, nullable | for quote lease requests |
| state | str | ACTIVE | PAUSED | TRIGGERED | REMOVED_BY_USER (tombstone) |
| baseline_price | str, nullable | PCT_MOVE baseline; re-set per trigger |
| eval_state | JSONB | e.g. {last_advice: "hold"} for event rules |
| cooldown_seconds | int, default 300 | min gap between triggers (recurring kinds) |
| trigger_count | int | |
| last_triggered_at | datetime, nullable | |
| created_at/updated_at | timestamps | TimestampMixin |

Unique: `(user_id, instrument_id, source, source_key)` — partial on `source_key IS NOT NULL`.

### `alert_triggers`

`id, rule_id (FK, indexed), user_id, instrument_id, kind, trigger_value (str), context JSONB, created_at (indexed)` — append-only; retention cleanup mirrors `run_smart_exit_cleanup` (default 30d).

### Rule semantics

- **PRICE_ABOVE / PRICE_BELOW**: one-shot. `ltp >= level` / `ltp <= level` → trigger, state → TRIGGERED. `POST /alerts/{id}/reset` re-arms. Sync param changes reset TRIGGERED → ACTIVE.
- **PCT_MOVE**: recurring. Baseline = first quote seen after activation; `|ltp − baseline|/baseline ≥ pct` → trigger and re-baseline to trigger ltp. `market.quote` carries no prev_close.
- **ADVICE_CHANGE**: evaluated on `alerts.sync`; fires when `position_advice.action` differs from `eval_state.last_advice` (first seen seeds, no fire).
- **REBALANCE_SIGNAL**: evaluated on `alerts.sync`; fires when `rebalance_suggested` transitions false → true.

---

## Sync Reconciliation (portfolio-service `sync_service`)

On `alerts.sync` for (user_id, instrument_id):

1. `clear=true` → set all RESEARCH rules for the pair to PAUSED (not deleted — history preserved; re-activated if the company is re-monitored and intents re-derive them).
2. For each `desired_alerts` item: upsert by `source_key` — create if absent (unless a `REMOVED_BY_USER` tombstone exists for that key), update `params`/`reason`/`source_run_id`/`broker_id`/`account_id`; if `params` changed and state == TRIGGERED → back to ACTIVE.
3. RESEARCH rules whose `source_key` is absent from `desired_alerts` → PAUSED (sync-managed deactivation).
4. Update `eval_state` and evaluate ADVICE_CHANGE/REBALANCE_SIGNAL rules → produce triggers.
5. Reconcile the quote-subscription set for the instrument.

Manual (source=MANUAL) rules are never touched by sync. User `DELETE` on a RESEARCH rule → tombstone `REMOVED_BY_USER` (blocks recreation); `DELETE` on MANUAL → hard delete.

---

## Quote Evaluation (portfolio-service)

`PortfolioMarketDataConsumer.process_message` calls `alert_evaluation_service.on_quote(instrument_id, ltp)` after `market_data_cache.update_quote`. The evaluator keeps an in-memory `instrument_id → [rules]` index, invalidated on any rule write (CRUD/sync) and lazily rebuilt. Cooldown: skip if `last_triggered_at` within `cooldown_seconds`.

### Quote coverage

`AlertSubscriptionService` (modelled on BAS `SubscriptionPublisher`) publishes `market.subscription.request` with `service_id="portfolio-alerts"`, `lease_ttl_seconds=120`, refreshed on a loop, for instruments having ≥1 active price-based rule and a known `broker_id` (account_id optional). Reconciled on rule-set changes. Without a lease, alerts still evaluate on whatever quotes flow on the shared stream.

---

## REST API (portfolio-service `/api/v1/alerts`)

| Endpoint | Purpose |
|---|---|
| `GET /alerts?source=&kind=&state=&instrument_id=&limit=&offset=` | List caller's rules |
| `POST /alerts` | Create manual rule `{instrument_id, kind, params, broker_id?, account_id?, cooldown_seconds?, reason?}` |
| `GET /alerts/{id}` | Get one rule |
| `PUT /alerts/{id}` | Update params / is_active (PAUSED↔ACTIVE) / cooldown |
| `DELETE /alerts/{id}` | MANUAL → hard delete; RESEARCH → tombstone |
| `POST /alerts/{id}/reset` | TRIGGERED → ACTIVE (re-arm one-shot) |
| `GET /alerts/triggers?instrument_id=&rule_id=&limit=&offset=` | Trigger history |

RBAC: `alerts: {read.self, write.self}` in `rbac_policies.yaml`.

## Error Codes

- `ALERT_001` — rule not found / not owned by caller
- `ALERT_002` — invalid params for kind (e.g. missing `level` on PRICE_ABOVE)
- `ALERT_003` — invalid kind
- `ALERT_004` — rule is TRIGGERED; reset before editing params (PUT semantics — decided: param edits auto-reset instead; error unused)

## ai-scoring-service changes

- `research/alert_intents.py` — pure `derive(run, report) -> AlertsSyncV1 | None`:
  - Always: `advice_watch` (ADVICE_CHANGE). HOLDINGS source also `rebalance_watch`.
  - WATCHLIST + action `enter|accumulate`: `entry_zone` PRICE_BELOW at nearest support ≤ price (fallbacks: scenario base-range low → pattern reference); `breakout_entry` PRICE_ABOVE at pattern reference level / nearest resistance when a pattern is forming/triggered.
  - HOLDINGS + action `reduce|exit`: `exit_target` PRICE_ABOVE at nearest resistance / base-range mid; `exit_stop` PRICE_BELOW at invalidation/nearest support.
  - HOLDINGS + action `hold|accumulate`: `risk_floor` PRICE_BELOW at nearest support.
  - `wait|avoid` → watches only.
- Publish in `orchestrator.submit_analysis` after `analysis_submitted` (system_context user_id = `run.acting_user_id`). Scope: `run_source ∈ {WATCHLIST, HOLDINGS}` only.
- Clears (`alerts.sync{clear:true}`): watchlist item delete, `set_holding_monitoring(false)`, monitor master-toggle off for the scope.
- `events/alert_triggered_consumer.py` — consumes `alerts.triggered`. Only `source=RESEARCH` + `kind ∈ {PRICE_ABOVE, PRICE_BELOW}` enqueues a recheck (`ResearchService.enqueue_alert_recheck`): resolves the origin run via `source_run_id`, checks the scope guards (monitor toggle on, watchlist item still enabled, holding still held/not excluded), applies a per-instrument cooldown (`RESEARCH_ALERT_RECHECK_MIN_MINUTES`, default 240) + the existing per-(company, profile) dedupe, then enqueues under the origin's same source/profile with `from_run_id` provenance — the completed run republishes `alerts.sync` and re-arms the rules. Manual alerts and `ADVICE_CHANGE`/`REBALANCE_SIGNAL` never re-research (they originate from research completion and would self-loop). There is no periodic re-research scheduler — alerts drive it.
- **Edge (resolved)**: sold holdings — if a research alert fires on an instrument no longer in the configured account, the recheck skips and publishes an `alerts.sync{clear:true}` so stale rules are deactivated.

## notification-service changes

- `stream_classification.py`: `alerts.*` → `StreamType.NOTIFICATION` (explicit entry).
- `category_classification.py`: `"alerts."` prefix → `NotificationCategory.RISK`… decided: add explicit `alerts.` → new mapping `TRADING`? — `alerts.` maps to `RISK`-adjacent `TRADING`? Decision: map `alerts.` → `TRADING` (entry/exit activity). Revisit if a dedicated ALERTS category is wanted later.

## Frontend changes

- `api/alertApi.ts` + `types/alerts.ts` (portfolio client).
- `components/research/AlertsPanel.tsx` — research-sourced rules + trigger feed; new "Alerts" tab in `Research.tsx`.
- `components/panels/AlertManager.tsx` — rewrite against the real API (manual CRUD; research rules shown read-only with source badge).
- Per-column filters on Companies/Holdings/Runs tables (shared `useColumnFilters` + header popovers).
- Holdings tab bulk ops (run/monitor/cancel/retry-failed) via client-side fan-out on existing endpoints.

## Dependencies & Sequencing

1. smarttrade-common event schemas (blocks 2+3).
2. portfolio-service alerts module (+migration).
3. ai-scoring intents + publish/clears.
4. notification-service classification.
5. Frontend (alerts UI; filters; bulk ops — independent, can interleave).

## Testing Strategy

- portfolio-service unit: evaluators (price crossing incl. boundary, pct re-baseline, advice transition, rebalance flip), sync reconcile (create/update/deactivate/tombstone/clear), cooldown.
- portfolio-service routes: CRUD + ownership + validation; consumer test: `alerts.sync` idempotent reconcile.
- ai-scoring-service unit: intent derivation on fixture reports (each action × source matrix; level fallbacks).
- notification-service: classification unit tests.
- Frontend: `npm run lint`, `npm run build`.
- E2E (manual): create manual alert → quote ticks → `alerts.triggered` on WS; complete HOLDINGS run → rules reconciled + trigger feed.
