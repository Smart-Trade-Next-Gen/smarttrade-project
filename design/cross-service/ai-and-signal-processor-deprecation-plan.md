# Deprecation Plan: ai-service + signal-processor-service → amis-lab-service

## Goal
Remove both `ai-service` and `signal-processor-service` from the monorepo. Move the only actively-used features into `amis-lab-service`. Net result: 11 → 9 services.

## Guiding Principle
Keep only what's used. The live analysis pipeline (polling, runtime context, signals API, events) has zero consumers — it gets deleted. The deterministic engines and Trade Intelligence get moved.

## Architecture After Migration

| Service | Role |
|---------|------|
| **amis-lab-service** | Research + dataset generation + replay + walk-forward validation + **Trade Intelligence assessment** + **deterministic market analysis engines** |
| **amis-core-service** | Artifact registry + promotion + lineage (unchanged) |

## Part A — Trade Intelligence → amis-lab

Trade Intelligence (`POST /api/v1/trade-intelligence/assess`) is the **only actively-used SPS feature** (frontend AMIS panel calls it).

1. Move `engines/trade_intelligence/` (SetupRankingEngine, HorizonEngine, QualityEngine, RiskEngine, TradeIntelligenceService) into `amis_lab/engines/trade_intelligence/`.
2. **Fix 3 critical TI bugs during the move:**
   - **Import path**: `routes_trade_intelligence.py:18` imports from wrong module. Change to `from signal_processor_service.engines import trade_intelligence_service`.
   - **Response validation**: `routes_trade_intelligence.py:121-122` converts `datetime` → ISO string, but `TradeIntelligenceResponse.timestamp` is typed as `datetime`. Remove the manual `.isoformat()` conversion — Pydantic v2 serializes `datetime` automatically.
   - **Logic regression**: `trade_intelligence_service.py:62-75` collapsed 6-branch `_determine_overall_recommendation` into 3 branches, losing NEUTRAL+HIGH→FAVORABLE and NEUTRAL+LOW→CAUTION cases. Restore full original logic from ai-service.
3. Add `amis_lab/api/routes_trade_intelligence.py` with `POST /api/v1/trade-intelligence/assess`; register in `main.py`.
4. Add `trade-intelligence` resource to amis-lab `rbac_policies.yaml`.

## Part B — Deterministic Engines → amis-lab (batch library)

These 10 engines are pure Python with no DB or external dependencies. They are needed to finally implement the replay pipeline's candle analysis.

5. Move engines into `amis_lab/engines/`:
   - `swing_detection.py`
   - `market_structure.py`
   - `market_regime.py`
   - `liquidity.py`
   - `zone.py`
   - `candlestick_patterns.py`
   - `breakout_retest.py`
   - `multi_timeframe.py`
   - `setup_registry.py`
   - `setup_scoring.py`
   - `feature_generation.py`
6. Refactor into a **batch-mode orchestrator** that operates on a per-call local context (not the SPS shared in-memory global store). Strip polling, event-publishing, DB-snapshot concerns.
7. **Implement `_fetch_candles()` in `replay_pipeline_worker.py`** using `amis_lab.data.candle_fetcher`, then run the batch orchestrator to generate `ReplayEvent`/`ReplaySignal`/structure/zones data.

## Part C — Drop the Dormant Pipeline

The following SPS features have **zero consumers** and are deleted:

8. Do **not** port:
   - `polling_service.py` (60s MDS polling loop)
   - `runtime_context` global store + `state_persistence`
   - `routes_analysis.py` (analysis query endpoints)
   - `routes_instruments.py` (user watchlist CRUD)
   - `routes_signals.py` (BUY/SELL signals)
   - `event_publisher.py` (market.analysis.completed, trade.setup.snapshot)
   - DB tables: `analysis_snapshots`, `user_instruments`, `market_structure_state`, `runtime_context_state`
9. Drop events from catalog: `market.analysis.completed`, `trade.setup.snapshot`.
10. Drop the entire `smarttrade_signal_processor` database (no data migration — watchlists only existed for the deleted polling pipeline).

## Part D — Frontend

11. Move `assessSetup` from `signalProcessorClient.ts` → `amisLabClient.ts`; change base URL from `SIGNAL_PROCESSOR_API` → `AMIS_LAB_URL`.
12. Remove from `apiConfig.ts`: all `SIGNAL_PROCESSOR_API` endpoints (analysis, instruments, signals, setups, market-structure) + the `signalProcessor` service URL.
13. Remove from `App.tsx`: commented-out `/signal-processor` route.
14. Remove from `Sidebar.tsx`: the `/signal-processor` nav link.
15. Remove from `vite.config.ts`: the `/signal-processor` proxy block.
16. Resolve `AMISDashboard.tsx`: it references non-existent `dashboardMetrics` store properties. Delete the component since there is no backend for it.

## Part E — Event Catalog (smarttrade-common)

17. Remove `market.analysis.completed` and `trade.setup.snapshot` events (no consumers).
18. Reassign `ai.*` events currently owned by `signal-processor-service` to `amis-lab-service`.
19. Fix duplicate entries: `ai.setup.scored` and `ai.outcome.evaluated` each appear twice.

## Part F — Decommission signal-processor-service

20. Delete `signal-processor-service/` directory.
21. Remove SPS from:
   - `CLAUDE.md` service table
   - `docker-compose.yml` env vars (`VITE_SIGNAL_PROCESSOR_API`, etc.)
   - `smarttrade-project/design/cross-service/ai-service-deprecation-plan.md` (update status)
   - `PLAYWRIGHT_TESTS_README.md`
22. Delete/relocate:
   - `smarttrade-tests/e2e/integration/signal_processor/`
   - `smarttrade-tests/e2e/integration/ai-service/`

## Part G — Finish ai-service Removal

23. Delete `ai-service/` directory.
24. Delete `smarttrade-frontend/src/api/amisClient.ts` (fully orphaned, nothing imports it).
25. Delete `smarttrade_tests/e2e/integration/ai-service/` test files.

## Verification Checklist

- [x] `amis-lab`: `uv run ruff check` passes
- [x] `amis-lab`: imports resolve (engines + route)
- [x] `frontend`: `npm run build` passes
- [x] Grep sweep: no `signal-processor` / `ai-service` / `8012` / `8013` / `VITE_SIGNAL_PROCESSOR_API` in active code
- [ ] `amis-lab`: `uv run pytest` passes
- [ ] `amis-lab`: `uv run alembic upgrade head` succeeds
- [ ] Manual: AMIS panel `/assess` works against amis-lab
- [ ] Manual: Replay Dashboard shows real candles (not empty)

## Risks

| Risk | Mitigation |
|------|------------|
| Batch orchestrator refactor (engines assume shared context store) | Isolate per-call context; add unit tests for each engine with local state |
| Replay correctness once `_fetch_candles` is real | Verify on known instrument/date range; compare signal counts |
| Trade Intelligence logic regression (already broken) | Restore full 6-branch logic from ai-service source; add unit tests |
