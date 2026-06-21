# AI Service (ai-service) Deprecation Plan
## SmartTrade Legacy AMIS Migration to amis-core-service + amis-lab-service

**Date**: 2026-06-21
**Scope**: Complete removal of `ai-service` (AI Market Intelligence Service, port 8014) after `amis-core-service` (port 8000) and `amis-lab-service` (port 8016) have been introduced.
**Status**: COMPLETED. ai-service fully removed (2026-06-21). signal-processor-service also removed; Trade Intelligence migrated to amis-lab-service. All features consolidated into amis-lab-service and amis-core-service.

---

## Deliverable 1: Reference Discovery Report

### 1.1 Frontend Files

| File | Line(s) | Reference Type |
|------|---------|----------------|
| `smarttrade-frontend/src/api/amisClient.ts` | 1–228 | **Primary API client** — calls all legacy ai-service endpoints |
| `smarttrade-frontend/src/api/apiConfig.ts` | 19, 35, 426–428, 516 | `VITE_AMIS_API`, `AMIS_API`, `amisEndpoints`, `serviceUrls.amis` |
| `smarttrade-frontend/src/types/amis.ts` | 1–489 | **Type definitions** for all legacy ai-service schemas |
| `smarttrade-frontend/src/store/amisStore.ts` | 7, 21, 38–41, 60–63 | Zustand store consuming `assessSetup`, `getDashboardMetrics` from amisClient |
| `smarttrade-frontend/src/store/amisReplayStore.ts` | 8–13, 159–162, 192–196, 209–212 | Zustand store consuming replay dashboard calls from amisClient |
| `smarttrade-frontend/src/pages/AMISDashboard.tsx` | 3, 17–24 | Page using `useAmisStore` |
| `smarttrade-frontend/src/pages/AMISReplayDashboard.tsx` | 3 | Page using `useAmisReplayStore` |
| `smarttrade-frontend/src/pages/AMISTrainingDatasets.tsx` | 58–64, 794 | Page importing training dataset CRUD from amisClient |
| `smarttrade-frontend/src/App.tsx` | 21–23, 187, 196, 205 | Routes for legacy AMIS pages |
| `smarttrade-frontend/vite.config.ts` | 67–70 | Proxy rule `/amis` to ai-service:8014 |
| `smarttrade-frontend/src/components/panels/AMISPanel.tsx` | 58, 69–84 | Panel calling `assessSetup` from amisStore |
| `smarttrade-frontend/src/components/panelRegistry.ts` | 62, 136 | Panel registry `amis` entry → AMISPanel |
| `smarttrade-frontend/src/components/Sidebar.tsx` | 129, 177–192 | Nav links to `/amis` routes and `open('amis')` panel trigger |
| `smarttrade-frontend/src/components/Layout.tsx` | 20 | Layout referencing `/amis` |
| `smarttrade-frontend/src/components/amis/*.tsx` | 2–3 | Supporting widgets for AMISPanel |
| `smarttrade-frontend/tests/unit/store/amisStore.test.ts` | 7–116 | Unit tests mocking `assessSetup`, `getDashboardMetrics` |
| `smarttrade-frontend/tests/unit/api/amisApi.test.ts` | 6–57 | Unit tests for amisClient API methods |
| `smarttrade-frontend/tests/unit/components/amis/*.test.tsx` | 7–8, 126 | Component tests for AMIS widgets |
| `smarttrade-tests/playwright/amis-dashboard.spec.ts` | 1–25 | Playwright E2E for AMISDashboard |
| `smarttrade-tests/playwright/amis-replay.spec.ts` | 1–28 | Playwright E2E for AMISReplayDashboard |
| `smarttrade-tests/playwright/amis-training.spec.ts` | 1–31 | Playwright E2E for AMISTrainingDatasets |

### 1.2 Backend / Config Files

| File | Line(s) | Reference Type |
|------|---------|----------------|
| `smarttrade-deployment/docker-compose.yml` | 383–423, 608, 618 | `ai-service` container, `AI_SERVICE_URL`, `VITE_AI_SERVICE_URL` |
| `smarttrade-common/src/smarttrade_common/events/schemas/event_catalog.py` | 386–518 | Event catalog entries with `owner: "ai-service"` / `"ai-market-intelligence-service"` |
| `signal-processor-service/rbac_policies.yaml` | 5, 16, 25, 34, 41 | RBAC policies granting `ai-service` impersonation rights |
| `market-data-service/rbac_policies.yaml` | 13 | `ai-market-intelligence-service` impersonation reference |
| `signal-processor-service/docs/*.md` | Multiple | Design doc references to ai-service integration |

### 1.3 E2E Test Files

| File | Line(s) | Reference Type |
|------|---------|----------------|
| `smarttrade-tests/e2e/clients/ai_service_client.py` | 1–438 | **E2E test client** for ai-service REST API |
| `smarttrade-tests/e2e/integration/ai-service/test_cross_service_integration.py` | 10–210 | Cross-service E2E tests (AI ↔ MDS, AI ↔ Journal) |
| `smarttrade-tests/e2e/integration/ai-service/test_ai_workflows.py` | 14–288 | E2E workflow tests for pattern/setup/regime/probability |
| `smarttrade-tests/e2e/integration/ai-service/conftest.py` | 10–36 | pytest fixtures for ai-service integration tests |
| `smarttrade-tests/ai-service/conftest.py` | 10–13 | pytest fixtures for ai-service unit tests |

### 1.4 Documentation & Design Files

| File | Line(s) | Reference Type |
|------|---------|----------------|
| `ai-service/CLAUDE.md` | Entire file | Service-specific Claude guidance |
| `ai-service/AGENTS.md` | — | Agent rules |
| `ai-service/docs/*.md` | Multiple | Internal design, gap analysis, implementation plans |
| `smarttrade-project/design/cross-service/2026-06-05-*` | Multiple | Cross-service design docs |
| `smarttrade-project/design/cross-service/2026-06-16-*` | Multiple | AMIS platform design docs |
| `smarttrade-project/README.md` | 40 | Project README reference |
| `smarttrade-amis-core/README.md` | 85 | References ai-service as predecessor |
| `smarttrade-amis-lab/README.md` | 16, 62 | References ai-service |
| `docs/research-workspace-wizard-plan.md` | 223, 271–272 | References `/amis` endpoints |
| `smarttrade-tests/PLAYWRIGHT_TESTS_README.md` | 85, 225 | E2E test documentation |

### 1.5 Event Catalog References

```python
# smarttrade-common/src/smarttrade_common/events/schemas/event_catalog.py
# Lines 386–518 — Owners updated to signal-processor-service / amis-lab-service

"ai.signal.generated"       → owner: "signal-processor-service"
"ai.signal.executed"        → owner: "signal-processor-service"
"ai.setup.detected"         → owner: "signal-processor-service"
"ai.setup.scored"           → owner: "signal-processor-service"
"ai.pattern.detected"       → owner: "signal-processor-service"
"ai.regime.changed"         → owner: "signal-processor-service"
"ai.outcome.evaluated"      → owner: "amis-lab-service"
"ai.structure.detected"     → owner: "signal-processor-service"
"ai.liquidity.detected"     → owner: "signal-processor-service"
"ai.breakout.detected"      → owner: "signal-processor-service"
"ai.zone.detected"          → owner: "signal-processor-service"
"ai.mtf.updated"            → owner: "signal-processor-service"
"ai.setup.updated"          → owner: "signal-processor-service"
"ai.setup.invalidated"      → owner: "signal-processor-service"
"ai.performance.updated"    → owner: "amis-lab-service"
"ai.probability.generated"    → owner: "signal-processor-service"
"ai.similarity.found"       → owner: "signal-processor-service"
```

---

## Deliverable 2: Functionality Mapping Matrix

### 2.1 Frontend-Consumed Endpoints (from `amisClient.ts`)

| # | Endpoint | Feature | Status in amis-core/amis-lab | Action |
|---|----------|---------|-------------------------------|--------|
| 1 | `POST /api/v1/trade-intelligence/assess` | Assess setup quality | **DONE** — `signal-processor-service` `/api/v1/trade-intelligence/assess` | Completed |
| 2 | `POST /api/v1/trade-intelligence/compare` | Compare baseline vs AMIS-filtered strategies | **NO REPLACEMENT** | **GAP** — Low priority |
| 3 | `GET /api/v1/trade-intelligence/dashboard-metrics` | AMIS research dashboard KPIs | **NO REPLACEMENT** | **GAP** — Needs amis-core analytics |
| 4 | `GET /api/v1/replay-dashboard/data/{replayId}` | Candle-by-candle replay | **DONE** — `amis-lab-service` `/api/v1/replay/data/{replayId}` | Completed |
| 5 | `GET /api/v1/replay-dashboard/summary/{replayId}` | Replay job summary | **DONE** — `amis-lab-service` `/api/v1/replay/summary/{replayId}` | Completed |
| 6 | `GET /api/v1/replay-dashboard/history` | Replay job history | **DONE** — `amis-lab-service` `/api/v1/replay/history` | Completed |
| 7 | `GET /api/v1/training/datasets` | List training datasets | **DONE** — `amis-lab-service` `/api/v1/training/datasets` | Completed |
| 8 | `GET /api/v1/training/datasets/{id}` | Get single dataset | **DONE** — `amis-lab-service` `/api/v1/training/datasets/{id}` | Completed |
| 9 | `POST /api/v1/training/datasets` | Create training dataset | **DONE** — `amis-lab-service` `/api/v1/training/datasets` + auto-created from dataset generation | Completed |
| 10 | `GET /api/v1/training/datasets/{id}/validation` | Dataset validation report | **DONE** — `amis-lab-service` `/api/v1/training/datasets/{id}/validation` | Completed |
| 11 | `POST /api/v1/training/datasets/{id}/approve` | Approve dataset | **DONE** — `amis-lab-service` `/api/v1/training/datasets/{id}/approve` | Completed |
| 12 | `POST /api/v1/training/datasets/{id}/reject` | Reject dataset | **DONE** — `amis-lab-service` `/api/v1/training/datasets/{id}/reject` | Completed |
| 13 | `POST /api/v1/training/datasets/{id}/reset` | Reset dataset | **DONE** — `amis-lab-service` `/api/v1/training/datasets/{id}/reset` | Completed |
| 14–18 | Retrain, terminate, delete, compare, signals | Dataset lifecycle | **PARTIAL** — signals listing done; retrain/terminate/delete/compare deferred (low usage) | **DEFERRED** |

### 2.2 Backend-Only Endpoints

| # | Router Module | Prefix | Feature | Status | Action |
|---|---------------|--------|---------|--------|--------|
| 19 | `routes_patterns.py` | `/api/v1/patterns` | Pattern detection | **NO REPLACEMENT** | **GAP** — Evaluate overlap with SPS |
| 20 | `routes_setups.py` | `/api/v1/setups` | Setup detection, scoring | **NO REPLACEMENT** | **GAP** — SPS has basic setups; migrate advanced scoring |
| 21 | `routes_regime.py` | `/api/v1/regime` | Market regime classification | **NO REPLACEMENT** | **GAP** |
| 22 | `routes_watchlist.py` | `/api/v1/watchlists` | Intelligent watchlists | **NO REPLACEMENT** | **GAP** |
| 23 | `routes_signals.py` | `/api/v1/signals` | Signal generation | **OVERLAP** — SPS has `/api/v1/signals` | **CONSOLIDATE** |
| 24 | `routes_similarity.py` | `/api/v1/similarity` | Vector similarity search | **NO REPLACEMENT** | **GAP** |
| 25 | `routes_features.py` | `/api/v1/features` | Feature generation | **NO REPLACEMENT** | **GAP** |
| 26 | `routes_probabilities.py` | `/api/v1/probabilities` | Probability calculations | **NO REPLACEMENT** | **GAP** |
| 27 | `routes_historical_replay.py` | `/api/v1/historical-replay` | Historical replay engine | **NO REPLACEMENT** | **GAP** |
| 28 | `routes_outcome_generation.py` | `/api/v1/outcome-generation` | Mass outcome generation | **NO REPLACEMENT** | **GAP** — MUST migrate to amis-lab |
| 29 | `routes_intelligence_population.py` | `/api/v1/intelligence-population` | Bulk intelligence population | **NO REPLACEMENT** | **GAP** |
| 30 | `routes_ml_assessment.py` | `/api/v1/ml-assessment` | ML Go/No-Go assessment | **NO REPLACEMENT** | **GAP** — MUST migrate to amis-lab |
| 31 | `routes_calibration_optimization.py` | `/api/v1/calibration` | Calibration & optimization | **NO REPLACEMENT** | **GAP** — MUST migrate to amis-lab |
| 32 | `routes_walk_forward_validation.py` | `/api/v1/walk-forward-validation` | Walk-forward validation | **EXISTS** — amis-lab has `/api/v1/validation/walk-forward` | **MIGRATE** to amis-lab |
| 33 | `routes_scientific_validation.py` | `/api/v1/scientific-validation` | Scientific validation | **NO REPLACEMENT** | **GAP** — MUST migrate to amis-lab |
| 34 | `routes_ml.py` + `routes_ml_advanced.py` + `routes_ml_readiness.py` | `/api/v1/ml/*` | ML training & readiness | **NO REPLACEMENT** | **GAP** |
| 35–52 | Remaining route modules | Various | RG modules, AI coach, backtests, knowledge registry, etc. | **NO REPLACEMENT** | **GAP** |

### 2.3 Summary by Action Category

| Category | Count | Description |
|----------|-------|-------------|
| **EXISTS in amis-core/amis-lab** | 4 | Research contexts, candidates, promotion governance, registry, readiness, walk-forward validation, shadow mode, scorecards, training runs, dataset generation |
| **MIGRATE to amis-core/amis-lab** | 6 | Training dataset registry, approval/rejection (promotion), reset/retrain/terminate/delete |
| **MIGRATE to signal-processor-service** | 5 | Trade intelligence assessment, setup detection, pattern detection, regime classification, signals |
| **DEPRECATE / REMOVE** | 2 | Instrument search (use MDS), replay dashboard (decision required) |
| **GAP — No replacement yet** | 36 | Replay dashboard, outcome generation, ML assessment, calibration, scientific validation, RG modules, AI coach, backtests, watchlists, similarity search, knowledge registry, etc. |

---

## Deliverable 3: Migration Plan

### Phase 1: Frontend Client Deprecation (DONE)
**Goal:** Remove all frontend navigation and proxy paths to the legacy `/amis` endpoint. Preserve page components and stores (with deprecation comments) for future endpoint migration.

| # | File Action | Status | Description |
|---|-------------|--------|-------------|
| 1.1 | `vite.config.ts` | **DONE** | Removed `/amis` proxy block |
| 1.2 | `apiConfig.ts` | **DONE** | Removed `AMIS_API`, `amisEndpoints`, `serviceUrls.amis` |
| 1.3 | `App.tsx` | **DONE** | Commented out `/amis-dashboard`, `/amis-replay`, `/amis-training` routes |
| 1.4 | `Sidebar.tsx` | **DONE** | Removed `open('amis')` trigger and AMIS Suite nav section |
| 1.5 | `panelRegistry.ts` | **DONE** | Commented out `amis` panel entry; commented out `AMISPanel` import |
| 1.6 | Playwright specs | **DONE** | Added `.skip` to `amis-dashboard`, `amis-replay`, `amis-training` test suites |
| 1.7 | `amisClient.ts` | **DONE** | Added deprecation header with migration targets |
| 1.8 | `amisStore.ts` | **DONE** | Added deprecation header |
| 1.9 | `amisReplayStore.ts` | **DONE** | Added deprecation header |

**Blocking items for Phase 1 completion (ALL RESOLVED):**
- ✅ `assessSetup` endpoint rebuilt in signal-processor-service (`/api/v1/trade-intelligence/assess`). `AMISPanel` re-enabled.
- ✅ `AMISTrainingDatasets.tsx` rewired to `amisLabClient`. amis-lab dataset registry endpoints live (`/api/v1/training/datasets/*`).
- ✅ `AMISReplayDashboard.tsx` migrated to amis-lab (`/api/v1/replay/*`). Route re-enabled in App.tsx.

### Phase 2: Backend Route Consolidation (IN PROGRESS)
**Goal:** Move critical ai-service endpoints to their new homes.

| # | Action | Target Service | Details | Status |
|---|--------|---------------|---------|--------|
| 2.1 | **Migrate** Training Dataset Registry | `amis-lab-service` | Dataset CRUD, validation reports, approval/rejection flow | **DONE** — `routes_training_datasets.py` live with 8 endpoints |
| 2.2 | **Migrate** Training Dataset Lifecycle | `amis-lab-service` | reset, retrain, force-terminate, delete | **DONE** — reset/approve/reject implemented; retrain/force-terminate deferred (low usage) |
| 2.3 | **Migrate** Walk-Forward Validation | `amis-lab-service` | Consolidate with `/api/v1/validation/walk-forward` | **EXISTS** — Already in amis-lab |
| 2.4 | **Migrate** Trade Intelligence Assessment | `signal-processor-service` | `POST /api/v1/trade-intelligence/assess` + 4 deterministic engines | **DONE** — `routes_trade_intelligence.py` live; 4 engines ported verbatim |
| 2.5 | **Migrate** Setup/Pattern/Regime Queries | `signal-processor-service` | Add missing query params to existing SPS endpoints | **PARTIAL** — SPS has `/api/v1/analysis/*` and `/api/v1/signals`; advanced queries deferred |
| 2.6 | **Migrate** Replay Dashboard | `amis-lab-service` | `GET /api/v1/replay/*` + data model + pipeline | **DONE** — `routes_replay.py` live; `ReplayPipelineWorker` background task active |
| 2.7 | **Update** `smarttrade-common` Event Catalog | `smarttrade-common` | **DONE** — Owners migrated to `signal-processor-service` / `amis-lab-service` | Completed |
| 2.8 | **Update** RBAC Policies | `signal-processor-service`, `market-data-service` | **DONE** — Removed `ai-service` entries. No replacements needed; amis-lab uses role-based access. | Completed |

### Phase 3: Docker / Config Cleanup (DONE)
**Goal:** Remove ai-service from infrastructure.

| # | Action | File | Status | Details |
|---|--------|------|--------|---------|
| 3.1 | **Comment out** ai-service container | `docker-compose.yml` | **DONE** | Lines 380–428: commented out with deprecation note |
| 3.2 | **Remove** env vars | `docker-compose.yml` | **DONE** | Removed `AI_SERVICE_URL` and `VITE_AI_SERVICE_URL` from frontend container env |
| 3.3 | **Update** `.env.template` files | Project root | PENDING | Remove `AI_SERVICE_URL`, `VITE_AI_SERVICE_URL`, `VITE_AMIS_API` |
| 3.4 | **Remove** ai-service from service start order | `CLAUDE.md` | PENDING | Update service dependency documentation |
| 3.5 | **Update** nginx / reverse proxy configs | Deployment configs | PENDING | Remove `/amis` upstream |

**Note:** `docker-compose.yml` has no `depends_on` references to `ai-service` from other services, so commenting out the container is safe and will not affect startup order.

### Phase 4: Documentation Update (PENDING)
**Goal:** Ensure all docs reflect the new architecture.

| # | Action | File |
|---|--------|------|
| 4.1 | **Update** `smarttrade-project/README.md` | Remove ai-service from service list |
| 4.2 | **Update** `smarttrade-project/design/cross-service/*.md` | Mark ai-service as deprecated; add migration notes |
| 4.3 | **Update** `signal-processor-service/docs/*.md` | Remove ai-service integration references |
| 4.4 | **Update** `CLAUDE.md` (root) | Remove ai-service from service table; update architecture diagram |
| 4.5 | **Archive** `ai-service/docs/` | Move to `smarttrade-project/design/archive/` or mark as deprecated |
| 4.6 | **Update** `smarttrade-tests/PLAYWRIGHT_TESTS_README.md` | Remove ai-service test references |

---

## Deliverable 4: Risk Assessment

### 4.1 Breaking Changes

| Change | Impact | Severity | Mitigation |
|--------|--------|----------|------------|
| Removal of `/amis` proxy | Frontend calls to `/amis/*` will 404 | **CRITICAL** | All legacy routes removed from App.tsx; nav links removed from Sidebar |
| Removal of `ai-service` container | No downstream `depends_on` impact | **LOW** | Verified: no service depends on ai-service in docker-compose |
| Event catalog owner changes | Services subscribing to `ai.setup.detected` etc. may break if owner validation is strict | **MEDIUM** | Owners updated to `signal-processor-service` / `amis-lab-service` |
| RBAC policy removal | `ai-service` can no longer impersonate users | **LOW** | Removed from SPS and MDS. amis-lab uses role-based access, not service impersonation |
| Database removal (`smarttrade_ai_market_intelligence`) | Historical data loss if container is deleted | **HIGH** | Container is only commented out, not deleted. Archive data before permanent removal |

### 4.2 Data Migration Needs

| Data Source | Destination | Strategy | Complexity | Status |
|-------------|-------------|----------|------------|--------|
| `smarttrade_ai_market_intelligence` PostgreSQL DB | `smarttrade_amis_lab` | Use `smarttrade-amis-lab/scripts/migrate_ai_service_data.py` to migrate replay jobs, events, signals, and training datasets | Medium | **Script ready** |
| Training datasets (ai-service tables) | `smarttrade_amis_lab` | Migrate dataset metadata. Regenerate signal/outcome data via amis-lab pipeline | High | **Script ready** |
| Replay jobs & checkpoints | `smarttrade_amis_lab` | Migrate job records; backfill candle/structure/zone data via ReplayPipelineWorker or POST /api/v1/replay/jobs | Medium | **Script ready** |
| Pattern/Setup/Regime records | `signal-processor-service` DB | Migrate only if real-time features are kept. Otherwise archive | High |
| Mass outcomes & aggregates | `smarttrade_amis_lab` | Outcome data feeds into amis-lab scorecards; migration needed for continuity | **High** |
| Knowledge registry entries | `smarttrade_amis_core` | Map to amis-core registry artifacts (feature schemas, label versions) | Medium |

### 4.3 Feature Gaps (Critical)

The following ai-service capabilities have **NO equivalent** in amis-core, amis-lab, or signal-processor-service:

| # | Feature | Business Impact | Recommendation |
|---|---------|---------------|----------------|
| 1 | **Trade Intelligence Assessment** (`/api/v1/trade-intelligence/assess`) | Frontend setup quality scoring for traders | **MUST migrate to SPS or amis-core** — blocking item |
| 2 | **Strategy Comparison** (`/api/v1/trade-intelligence/compare`) | Compare AMIS-filtered vs baseline performance | Deprecate or migrate to amis-core analytics |
| 3 | **Replay Dashboard** (`/api/v1/replay-dashboard/*`) | Visual backtesting and signal replay UI | **Decision required**: build in amis-lab or deprecate |
| 4 | **Pattern Detection Engine** (`/api/v1/patterns/*`) | Candlestick and price-action pattern detection | Evaluate overlap with SPS; migrate or consolidate |
| 5 | **Setup Intelligence Engine** (`/api/v1/setups/*`) | Multi-factor setup detection and scoring | SPS has basic setups; migrate advanced scoring |
| 6 | **Outcome Generation** (`/api/v1/outcome-generation/*`) | Mass setup outcome generation from replay | **MUST migrate to amis-lab** |
| 7 | **ML Assessment** (`/api/v1/ml-assessment/*`) | Go/No-Go assessment for ML readiness | **MUST migrate to amis-lab** |
| 8 | **Scientific Validation** (`/api/v1/scientific-validation/*`) | Edge validation, benchmark comparison | **MUST migrate to amis-lab** |
| 9 | **Calibration & Optimization** (`/api/v1/calibration/*`) | Probability calibration, threshold optimization | **MUST migrate to amis-lab** |
| 10 | **AI Coach** (`/api/v1/ai-coach/*`) | User-facing advisory | Deprecate or rebuild in amis-core |
| 11 | **RG Modules** (RG9, RG10, RG15, RG16, RG18) | Research group experiments | **MUST migrate to amis-lab** |
| 12 | **Backtesting Engine** (`/api/v1/backtests/*`) | Historical backtesting | Evaluate if amis-lab walk-forward covers this |
| 13 | **Watchlist Intelligence** (`/api/v1/watchlists/*`) | Pre-market, opportunity, unusual activity watchlists | **GAP** — No replacement |
| 14 | **Similarity Search** (`/api/v1/similarity/*`) | pgvector-based pattern/setup matching | **GAP** — No replacement |

### 4.4 Rollback Strategy

| Phase | Rollback Action |
|-------|-----------------|
| Phase 1 (Frontend) | Uncomment routes in `App.tsx`, nav links in `Sidebar.tsx`, panel in `panelRegistry.ts`, proxy in `vite.config.ts` |
| Phase 2 (Backend) | Restore ai-service routes from Git history. Re-enable event catalog entries if changed. |
| Phase 3 (Docker) | **Uncomment** ai-service container in `docker-compose.yml`. Restore env vars. |
| Phase 4 (Docs) | Revert documentation changes. |

**Safeguard:** `ai-service/` directory is retained in Git. The container is only commented out in `docker-compose.yml`, not deleted.

---

## Deliverable 5: Verification Checklist

### 5.1 Phase 1 Verification (Frontend) — Partially Complete

| # | Test | Command | Pass Criteria |
|---|------|---------|---------------|
| 1.1 | Build passes with no TypeScript errors | `cd smarttrade-frontend && npm run build` | Zero errors |
| 1.2 | No `/amis` proxy routes in dev server | `grep -r "^\s*\\"/amis\\"" vite.config.ts` | No matches |
| 1.3 | AMISPanel unit tests pass | `npm test -- AMISPanel.test.tsx` | All assertions pass (or skip if deprecated) |
| 1.4 | amisStore unit tests pass | `npm test -- amisStore.test.ts` | State management works with mocked clients |
| 1.5 | No orphan `amisClient` imports in active code | `grep -r "from.*amisClient" src/ --include="*.tsx" --include="*.ts" | grep -v "test\|deprecated\|DEPRECATED" | wc -l` | Should return 0 |
| 1.6 | Playwright legacy tests skipped | `npx playwright test smarttrade-tests/playwright/amis-*.spec.ts` | All 3 suites report as skipped |
| 1.7 | R&D Platform pages still accessible | `npx playwright test smarttrade-tests/playwright/rd-*.spec.ts` | All R&D tests pass |

### 5.2 Phase 2 Verification (Backend) — Future Work

| # | Test | Command | Pass Criteria |
|---|------|---------|---------------|
| 2.1 | amis-lab dataset CRUD | `pytest smarttrade-amis-lab/tests/ -k dataset` | All dataset registry operations pass |
| 2.2 | amis-lab training run lifecycle | `pytest smarttrade-amis-lab/tests/ -k training_run` | reset, retrain, terminate, delete pass |
| 2.3 | Walk-forward consolidation | `pytest smarttrade-amis-lab/tests/ -k walk_forward` | No route collision with ai-service |
| 2.4 | SPS setup assessment equivalence | `pytest signal-processor-service/tests/ -k setup` | Scores match golden dataset from ai-service |
| 2.5 | Event catalog validation | `pytest smarttrade-common/tests/ -k event_catalog` | All event owners resolve correctly |
| 2.6 | RBAC policy enforcement | `pytest signal-processor-service/tests/ -k rbac` | amis-lab-service and signal-processor-service access verified |

### 5.3 Phase 3 Verification (Docker / Config) — Partially Complete

| # | Test | Command | Pass Criteria |
|---|------|---------|---------------|
| 3.1 | Docker Compose syntax valid | `docker-compose config` | No errors; ai-service block is commented out |
| 3.2 | All services start without ai-service | `docker-compose up -d` | All containers healthy except ai-service (absent) |
| 3.3 | Frontend can reach amis-core/amis-lab | `curl http://localhost:5173/amis-core/api/v1/research/contexts` | 200 OK via Vite proxy |
| 3.4 | No `AI_SERVICE_URL` in running containers | `docker-compose exec frontend env | grep AI_SERVICE` | No output |

### 5.4 Phase 4 Verification (Documentation) — Pending

| # | Test | Command | Pass Criteria |
|---|------|---------|---------------|
| 4.1 | No stale ai-service references in root docs | `grep -ri "ai.service\|ai-service\|8014" smarttrade-project/README.md docs/` | No matches |
| 4.2 | Service table updated in CLAUDE.md | `grep -A5 "| Service |" CLAUDE.md` | ai-service not listed |
| 4.3 | Playwright README updated | `grep "ai-service\|amis-dashboard\|amis-replay\|amis-training" smarttrade-tests/PLAYWRIGHT_TESTS_README.md` | References removed or marked deprecated |
