# Price Action Patterns + Candidate Permutation Engine (v1)

**Date:** 2026-06-21
**Status:** Approved for implementation
**Services affected:** AMIS Lab (`smarttrade-amis-lab`), AMIS Core (`smarttrade-amis-core`), Frontend (`smarttrade-frontend`)

---

## Executive Summary

**What:** Extend the AMIS research pipeline with (1) proven price-action and
candlestick pattern features, and (2) a bounded engine to systematically
generate research candidates across a constrained search space — so we can
discover the most robust trading configuration.

**Why:** Today the pipeline supports only two feature sets — V1 (technical
indicators) and V2 (V1 + RG16 structural features). There are no candlestick
or price-action pattern features. Researchers also create candidates one at a
time by hand. To find the best/robust configuration we need (a) richer
pattern signals and (b) a way to fan out candidates over combinations of
training-time dimensions.

**Which services:**
- **AMIS Lab** — new pattern detection module + V3 feature set.
- **AMIS Core** — new batch candidate creation endpoint.
- **Frontend** — permutation wizard + lightweight leaderboard.

**Success criteria:**
- V3 (patterns) is directly comparable to V2 in walk-forward results, so we
  can measure whether patterns add signal.
- A researcher can generate a *bounded* set of candidates (≤ cap) varying
  only training-time dimensions, then compare them on a leaderboard.
- All pattern features are strictly causal — no look-ahead bias.

**Explicit non-goals:**
- No fully automated unattended validation of hundreds of candidates
  (resource-infeasible; deferred).
- No new ML model architectures (reuse RandomForest / XGBoost).
- No external pattern libraries (e.g. TA-Lib) — keep dependencies minimal.

---

## Design Principles

1. **Shape-based, not context-variant patterns.** Detect ~12 geometric candle
   shapes; let the existing RG16 trend features supply directional context.
   This avoids redundancy (e.g. Hammer vs Hanging Man are the *same* shape and
   differ only by prior trend, which RG16 already encodes).
2. **Strictly causal detection.** Use only the current bar and prior bars. Do
   **not** copy the existing `center=True` look-ahead pattern found in
   `detect_swing_points()` in `rg16_features.py` (that is a pre-existing bug,
   flagged separately below).
3. **Separate training-time vs evaluation-time dimensions.** Only
   training-time dimensions create candidates. Evaluation-time dimensions
   (probability threshold, setup-score minimum, regime/structure filters) are
   swept on a single trained model via the existing operating envelope.
4. **Bounded permutation.** Hard server-side cap on generated candidates, with
   a live counter in the UI.
5. **Incremental delivery.** Validate that patterns actually help (Phase 1+2)
   before building the permutation engine (Phase 3+4).

---

## Architecture / Data Flow

```
                       ┌──────────────────────────────────────────────┐
                       │                 Frontend                      │
                       │  ResearchPage  ─►  GenerateCandidatesModal     │
                       │       │                    │                  │
                       │       │ (leaderboard)      │ batch create     │
                       └───────┼────────────────────┼──────────────────┘
                               │                    │
                  GET candidates + jobs     POST /candidates/batch
                               │                    │
                       ┌───────▼────────────────────▼──────────────────┐
                       │                 AMIS Core                      │
                       │  routes_research  ─►  CandidateArtifact (DRAFT)│
                       │                       ResearchContext (idempo.)│
                       │  config stored in candidate.extra_metadata     │
                       └───────┬────────────────────────────────────────┘
                               │  (user triggers per candidate)
                 Generate Dataset / Run Expanding Window
                               │
                       ┌───────▼────────────────────────────────────────┐
                       │                 AMIS Lab                        │
                       │  dataset_generation_service                     │
                       │     V1 / V2 / V3 feature builders               │
                       │        └─ pattern_features.extract_*()  (NEW)   │
                       │  validation_service (expanding window +         │
                       │     operating envelope sweep — already built)   │
                       └─────────────────────────────────────────────────┘
```

No new event streams. No cross-service DB access. The only new HTTP surface is
the AMIS Core batch endpoint; everything else reuses existing endpoints.

---

## Phase 1 — Pattern Detection Module (AMIS Lab)

**New file:** `smarttrade-amis-lab/src/amis_lab/data/pattern_features.py`

Detects **12 binary pattern columns**, each a pure pandas/numpy geometric
definition computed vectorially (no row loops).

| Column | Bars | Definition (causal) |
|--------|------|---------------------|
| `pat_doji` | 1 | body ≤ 10% of range |
| `pat_hammer_shape` | 1 | small body near top; lower wick ≥ 2× body |
| `pat_inv_hammer_shape` | 1 | small body near bottom; upper wick ≥ 2× body |
| `pat_marubozu` | 1 | body ≥ 90% of range (negligible wicks) |
| `pat_spinning_top` | 1 | small body; both wicks present and comparable |
| `pat_long_body` | 1 | body ≥ 1.5× rolling-20 mean body |
| `pat_bull_engulfing` | 2 | current bullish body fully engulfs prior bearish body |
| `pat_bear_engulfing` | 2 | current bearish body fully engulfs prior bullish body |
| `pat_inside_bar` | 2 | current high < prior high AND current low > prior low |
| `pat_outside_bar` | 2 | current high > prior high AND current low < prior low |
| `pat_morning_star` | 3 | down candle → small-body middle → up candle closing into 1st body |
| `pat_evening_star` | 3 | up candle → small-body middle → down candle closing into 1st body |

**Public API:**
```python
def extract_pattern_features(candles: pd.DataFrame) -> pd.DataFrame:
    """Returns DataFrame with ['timestamp'] + 12 pattern binary columns.

    All detection is causal: uses only the current bar and prior bars.
    Multi-bar patterns evaluate at the close of the final bar.
    """
```

**Design notes:**
- Body / wick thresholds are module-level constants for tunability.
- No `center=True` rolling. Multi-bar patterns use backward shifts only.
- Output is left-merged onto the feature frame on `timestamp`.

**Tests:** `tests/test_pattern_features.py` — each pattern verified on
hand-built synthetic candles (positive + negative cases), plus a
no-look-ahead assertion (output at row *i* is unchanged when future rows are
mutated).

---

## Phase 2 — V3 Feature Set Integration (AMIS Lab)

**Files modified:**
- `src/amis_lab/services/dataset_generation_service.py`
  - Add `feature_set.upper() == "V3"` branch → `_build_v3_dataset()`.
  - `_build_v3_dataset(candles, labels, htf_candles)` = V2 features +
    pattern features merged on `timestamp`.
  - Add `_generate_feature_schema_v3()` registering 33 feature definitions
    (21 from V2 + 12 patterns) with descriptions.

**Feature count:** V3 = 21 (V2) + 12 (patterns) = **33 features**.

**No DB migration needed** — `feature_set` is a free string; pattern columns
are stored in the parquet dataset.

**Validation flow:** unchanged. `runExpandingWindowValidation` already accepts
`feature_set`, so V3 flows through automatically, and the operating-envelope
filtering already implemented works identically.

### Decision Gate (mandatory)

After Phase 2, manually create one **V2** and one **V3** candidate on
NIFTY 1H, run expanding-window validation, and compare:
- Walk-forward Expected-R stability
- Filtered (operating-envelope) win % and net P&L (R)
- Per-window consistency

**If V3 does not beat V2, STOP** — the permutation engine is not worth
building. This gate prevents investing in a search engine over features that
add no signal.

---

## Phase 3 — Permutation Engine Backend (AMIS Core)

**New endpoint:** `POST /api/v1/research/candidates/batch`

**Request schema:**
```json
{
  "research_group": "string",
  "base_name": "string",
  "dimensions": {
    "instruments": ["NSE:CASH:INDEX:NIFTY", "NSE:CASH:INDEX:BANKNIFTY"],
    "primary_timeframe": ["1h"],
    "higher_timeframe": ["4h"],
    "feature_sets": ["V2", "V3"],
    "label_pairs": [
      {"horizon_bars": 16, "target_multiplier": 1.8},
      {"horizon_bars": 20, "target_multiplier": 2.0}
    ],
    "model_types": ["XGBOOST"]
  },
  "max_candidates": 50
}
```

**Response schema:**
```json
{
  "created": [
    {"id": "uuid", "candidate_name": "NIFTY_1H_V3_H16_T18_XGB",
     "research_context_id": "uuid"}
  ],
  "skipped": [
    {"candidate_name": "NIFTY_1H_V2_H20_T20_XGB", "reason": "ALREADY_EXISTS"}
  ],
  "total_requested": 8,
  "total_created": 7
}
```

**Behavior:**
1. Compute the Cartesian product of training-time dimensions.
2. **Reject with HTTP 400** if the product exceeds `max_candidates`
   (server-side guardrail; default cap 50).
3. For each combination:
   - Idempotently create / reuse a `ResearchContext` (keyed by instrument +
     timeframes; `context_hash` dedupes — see
     `ContextFingerprintService`).
   - Create a `CandidateArtifact` in `DRAFT` status with a deterministic
     name, e.g. `NIFTY_1H_V3_H16_T18_XGB`.
   - Store the full permutation config in the candidate's `extra_metadata`:
     ```json
     {
       "permutation": {
         "feature_set": "V3",
         "label_config": {"horizon_bars": 16, "target_multiplier": 1.8},
         "model_type": "XGBOOST",
         "hyperparameters": {"n_estimators": 300, "max_depth": 4}
       }
     }
     ```
4. Return created + skipped lists.

**Why metadata, not new columns:** `feature_set` / `horizon` / `model_type`
are dataset/training parameters, not context parameters. `CandidateArtifact`
already has an `extra_metadata` JSON field — no migration required.

**Naming collisions:** rely on the existing composite unique constraint
`(research_group, candidate_name)`. Duplicates are reported under `skipped`.

**RBAC:** reuse `@require_policy("research", "write")`.

**Files modified:**
- `src/amis_core/api/routes_research.py` — new `CreateCandidatesBatchRequest`
  model + endpoint.
- Reuse `ResearchContextRepository`, `CandidateArtifactRepository`,
  `ContextFingerprintService`, `CandidateStateMachine`.

---

## Phase 4 — Permutation Engine Frontend

**New component:** `src/components/GenerateCandidatesModal.tsx`
(opened from a button on the Research page Candidates tab).

**UI:**
- Multi-select chips per dimension: instruments, feature sets, label pairs,
  model types.
- **Live candidate counter**: "This will create **N** candidates." Turns red
  and disables submit above the cap (50).
- Research group + base name inputs.
- On submit → `POST /candidates/batch` → toast with created count → refresh
  list.

**Per-candidate flow:** each generated candidate flows through the *existing*
pipeline UI — user clicks "Generate Dataset" → "Run Expanding Window" on each
card, with parameters pre-filled from `candidate.metadata.permutation`. **No
auto-validation** (deliberate scope choice).

**Lightweight leaderboard:** extend the Candidates list with sortable columns
sourced from each candidate's latest completed validation job —
filtered win %, net P&L (R), risk:reward, windows passed — so candidates can
be ranked without a separate page.

**Files modified / added:**
- `src/modules/amis/pages/ResearchPage.tsx` — "Generate Candidates" button +
  leaderboard columns.
- `src/components/GenerateCandidatesModal.tsx` — new.
- `src/api/amisCoreClient.ts` — add `createCandidatesBatch()`.
- `src/components/CandidatePipelineCard.tsx` — read config from
  `candidate.metadata.permutation` to pre-fill dataset / validation params.

---

## Detailed Specification: Dimension Classification

| Dimension | Class | In permutation grid? | Mechanism |
|-----------|-------|----------------------|-----------|
| Instrument | Training | ✅ | New context + candidate |
| Primary / Higher TF | Training | ✅ | New context + candidate |
| Feature set (V1/V2/V3) | Training | ✅ | candidate metadata |
| `horizon_bars` | Training | ✅ | `label_config` |
| `target_multiplier` | Training | ✅ | `label_config` |
| Stop constraints | Training | ⚠️ optional | `label_config` |
| Model type / hyperparameters | Training | ✅ | candidate metadata |
| `probability_threshold` | **Evaluation** | ❌ | operating envelope (swept on one model) |
| `setup_score_min` | **Evaluation** | ❌ | operating envelope |
| Regime / structure filters | **Evaluation** | ❌ | operating envelope |

**Rationale:** evaluation-time dimensions do not require retraining. The
operating-envelope filtering (already implemented in `validation_service`)
sweeps them on a single trained model, so including them in the permutation
grid would multiply candidate counts pointlessly.

---

## Dependencies & Sequencing

```
Phase 1 (pattern module)
   └─► Phase 2 (V3 feature set) ─► DECISION GATE: do patterns help?
                                       │ (only if yes)
                                       ▼
                                  Phase 3 (batch endpoint)
                                       └─► Phase 4 (frontend wizard + leaderboard)
```

- Phases 1+2 are one reviewable unit (AMIS Lab only).
- Phase 3 is independent (AMIS Core only).
- Phase 4 depends on Phase 3 (frontend consumes the batch endpoint).
- The decision gate after Phase 2 is a hard checkpoint.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Pattern sparsity → overfitting on small early windows (V3 = 33 features; 2018-19 train ≈ 2k candles) | Keep to 12 patterns; rely on tree models' feature selection; monitor V2-vs-V3 stability at the Phase 2 gate |
| Combinatorial explosion | Server + UI hard cap (50); only training-time dims in grid |
| Look-ahead bias | Causal-only detection; explicit no-look-ahead unit test; do not reuse `center=True` |
| Patterns add no value | Phase 2 decision gate before any permutation work |
| Pre-existing swing-point look-ahead bug (`detect_swing_points`, `center=True`) | Flagged here; fix is **out of scope** for this design but recommended as a separate change |

---

## Testing Strategy

**Unit (AMIS Lab):**
- `tests/test_pattern_features.py` — per-pattern positive/negative cases on
  synthetic candles; no-look-ahead assertion.
- Extend dataset-generation tests to assert V3 produces 33 feature columns
  and pattern columns are present.

**Unit (AMIS Core):**
- Batch endpoint: cap enforcement (400 over limit), idempotent context reuse,
  duplicate-name skipping, metadata persistence.

**Frontend:**
- `npm run build` + `npx eslint` on touched files.
- Modal: live counter correctness; cap disables submit.

**Manual E2E:**
- Create V2 + V3 candidate → generate dataset → run expanding window →
  compare in `ValidationResultsModal` (Phase 2 gate evidence).
- Batch-create a small grid → verify candidates appear with correct metadata
  → run one through the pipeline.

**Verification commands:**
```bash
# AMIS Lab
cd smarttrade-amis-lab && uv run ruff check src/ && uv run pytest tests/test_pattern_features.py

# AMIS Core
cd smarttrade-amis-core && uv run ruff check src/ && uv run pytest

# Frontend
cd smarttrade-frontend && npm run build && npx eslint src/components/GenerateCandidatesModal.tsx
```

---

## Open Items / Future Work

- Auto-validation pipeline (background job that runs dataset generation +
  expanding-window validation across the whole grid and produces a ranked
  leaderboard) — deferred; resource-intensive.
- Custom user-defined patterns via JSON config — deferred.
- Fix the pre-existing `center=True` look-ahead in `detect_swing_points()`.
- Promote validated V3 pattern findings as `ResearchAsset`
  (`FEATURE_DISCOVERY`) records once the Assets UI exists.
