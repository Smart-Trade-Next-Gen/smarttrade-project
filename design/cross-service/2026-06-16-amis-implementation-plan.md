> **DEPRECATED**: This document describes the legacy `ai-service` (port 8014), which has been deprecated. Its functionality has been migrated to `amis-core-service` (port 8000), `amis-lab-service` (port 8016), and `signal-processor-service` (port 8012). See `ai-service-deprecation-plan.md` for details.

# AMIS Implementation Plan

| Field | Value |
|-------|-------|
| **Document Version** | 1.0 |
| **Date** | 2026-06-16 |
| **Status** | Ready for Implementation |
| **Related Documents** | [AMIS Platform Architecture v2.3](./2026-06-16-amis-platform-v1.md), [AMIS HLD v1.3](./2026-06-16-amis-hld-v1.md) |
| **Audience** | Engineering Team |

---

## Executive Summary

This plan assumes the existing AMIS Core and AMIS Lab codebases are the starting point. Both services have substantial working code. The plan is **incremental** -- build new artifacts on top of what exists, reuse patterns from `smarttrade-common`, and do not rewrite working code.

### What Exists Today

| Component | AMIS Core | AMIS Lab |
|-----------|-----------|----------|
| Models | 15/26 built | 7/7 built |
| Registry API | 50+ endpoints | Stubs (501) |
| Promotion API | 15+ endpoints | N/A |
| Services | PromotionService complete | DependencyValidationService complete |
| Repositories | 12 built | Operations repos built |
| Migrations | 4 done | 2 done |
| Data Pipeline | N/A | Complete (candles, features, labels) |
| Research Scripts | N/A | 40+ complete |
| Regime Analytics | N/A | Complete (scorecard, signature) |
| Baselines | N/A | 4 complete |

### smarttrade-common Patterns Available

| Pattern | How We Use It |
|---------|---------------|
| `UUIDMixin`, `TimestampMixin` | All AMIS models inherit these |
| `BaseRepository` | Generic CRUD -- inherit for simple repos, override for complex queries |
| `@transactional` / `@transactional(read_only=True)` | Service layer methods |
| `@subscribe` + `EventBus` | Event-driven integration between Core and Lab |
| `BaseServiceClient` | Core-Client, Lab-Client for inter-service HTTP calls |
| `CommonSettings` | `AMISCoreSettings`, `AMISLabSettings` extend this |
| `SmartTradeError` | All AMIS errors use this with `error_code` |
| `RequestContextMiddleware` | Trace ID injection |
| `system_context()` | Background task context |
| `IdempotencyService` | Deduplicate critical operations |
| `OutboxService` | Transactional event publishing |
| `Health checks` | Service-specific checks |

---

## Phase 1A: Core Registry (Weeks 1-3)

**Goal**: All governance models exist in Core with REST APIs. No business logic yet -- just registration, retrieval, and lineage.

### 1A.1 New Models (AMIS Core)

Add these models to `src/amis_core/models.py`. **Reuse existing patterns** -- look at `FeatureSchema` and `ModelArtifact` for reference.

| Model | Pattern Source | Key Fields |
|-------|----------------|------------|
| `ResearchContext` | `FeatureSchema` (immutable, hash-based) | `context_hash` UK, `instrument_ids` JSON, `primary_timeframe`, `higher_timeframe`, `vix_required`, `vix_min/max`, `volatility_regimes` JSON, `market_structures` JSON |
| `RegimeDefinition` | `LabelVersion` (versioned, opaque) | `definition_name` UK, `semantic_version`, `regime_dimensions_json` JSON (Lab owns schema), `defining_lab` |
| `ExecutionProfile` | `OperatingEnvelopeVersion` (opaque JSON) | `profile_name` UK, `semantic_version`, `execution_params_json` JSON, `defining_lab` |
| `ResearchAsset` | `ModelArtifact` (immutable, status enum) | `asset_type`, `name` UK, `status`, `parent_artifact_ids` JSON, `evidence_references` JSON, `semantic_version` |
| `CandidateArtifact` | `ModelArtifact` (with state machine) | `candidate_name` UK, `research_group`, `candidate_type`, `status` (state machine), `research_context_id` FK |
| `ExperimentArtifact` | `ModelArtifact` (opaque definition) | `experiment_name` UK, `experiment_type`, `status`, `performing_lab`, `experiment_definition_json` JSON, `results_json` JSON |
| `ReportArtifact` | `DatasetQualityReport` (content-addressable) | `report_type`, `report_name`, `report_hash` UK, `storage_path`, `storage_size_bytes`, `storage_checksum` |
| `ResearchProgram` | `ModelArtifact` (simple entity) | `program_name` UK, `description`, `status`, `lead_researcher` |
| `ResearchTrack` | `ValidationWindow` (belongs to parent) | `program_id` FK, `track_name`, `description`, `status` |
| `ResearchMilestone` | `ValidationWindow` (belongs to parent) | `track_id` FK, `milestone_name`, `description`, `status`, `linked_artifact_ids` JSON, `target_date`, `completed_date` |

### 1A.2 Model Changes to Existing Entities

Update these existing models in `src/amis_core/models.py`:

| Model | Change | Rationale |
|-------|--------|-----------|
| `FeatureSchema` | Add `research_context_id: UUID = Field(foreign_key="research_contexts.id", nullable=False)` | Mandatory context linkage |
| `Dataset` | Add `research_context_id: UUID = Field(foreign_key="research_contexts.id", nullable=False)` | Mandatory context linkage |
| `TrainingRun` | Add `research_context_id: UUID = Field(foreign_key="research_contexts.id", nullable=False)` | Mandatory context linkage |
| `ModelArtifact` | Add `research_context_id: UUID = Field(foreign_key="research_contexts.id", nullable=False)` | Mandatory context linkage |
| `OperatingEnvelopeVersion` | Add `research_context_id: UUID = Field(foreign_key="research_contexts.id", nullable=False)` | Mandatory context linkage. Add `envelope_hash: str` UK for immutability. |

### 1A.3 Repositories

Create `src/amis_core/repositories/research_repositories.py`.

**Pattern**: Inherit `BaseRepository` from `smarttrade_common` for simple CRUD. Only override for complex queries (e.g., `list_by_program`, `list_by_track`).

| Repository | Base Class | Special Methods |
|------------|-----------|-----------------|
| `ResearchContextRepository` | `BaseRepository[ResearchContext]` | `get_by_context_hash()`, `list_by_instrument()` |
| `RegimeDefinitionRepository` | `BaseRepository[RegimeDefinition]` | `get_by_definition_name()`, `list_by_status()` |
| `ExecutionProfileRepository` | `BaseRepository[ExecutionProfile]` | `get_by_profile_name()`, `list_by_status()` |
| `ResearchAssetRepository` | `BaseRepository[ResearchAsset]` | `get_by_name()`, `list_by_asset_type()` |
| `CandidateArtifactRepository` | `BaseRepository[CandidateArtifact]` | `get_by_candidate_name()`, `list_by_research_group()` |
| `ExperimentArtifactRepository` | `BaseRepository[ExperimentArtifact]` | `get_by_experiment_name()`, `list_by_performing_lab()` |
| `ReportArtifactRepository` | `BaseRepository[ReportArtifact]` | `get_by_report_hash()`, `list_by_report_type()` |
| `ResearchProgramRepository` | `BaseRepository[ResearchProgram]` | `get_by_program_name()` |
| `ResearchTrackRepository` | `BaseRepository[ResearchTrack]` | `list_by_program_id()` |
| `ResearchMilestoneRepository` | `BaseRepository[ResearchMilestone]` | `list_by_track_id()`, `list_by_status()` |

**Existing repositories to update**:
- `DatasetRepository`: Add `list_by_research_context()`
- `TrainingRunRepository`: Add `list_by_research_context()`
- `ModelArtifactRepository`: Add `list_by_research_context()`
- `OperatingEnvelopeVersionRepository`: Add `get_by_envelope_hash()`

### 1A.4 Database Migration

Create `migrations/versions/005_add_research_tier.py`.

Use Alembic autogenerate, then manually review:
- All new tables
- All new `research_context_id` FK columns on existing tables
- Indexes on all new FKs and UKs
- Check constraints for `CandidateArtifact.status` (valid states)

### 1A.5 API Routes

Create `src/amis_core/api/routes_research.py`.

**Pattern**: Follow `routes_registry.py` exactly. Same error handling, same response models, same pagination.

| Endpoint | Method | Status | Request/Response |
|----------|--------|--------|-----------------|
| `/api/v1/research/contexts` | POST | Create | `CreateResearchContextRequest` -> `ResearchContextResponse` |
| `/api/v1/research/contexts` | GET | List | Query params: instrument, timeframe, status |
| `/api/v1/research/contexts/{id}` | GET | Get by ID | `ResearchContextResponse` |
| `/api/v1/research/contexts/{id}/artifacts` | GET | List linked artifacts | All artifacts with this context |
| `/api/v1/research/regime-definitions` | POST | Create | `CreateRegimeDefinitionRequest` -> `RegimeDefinitionResponse` |
| `/api/v1/research/regime-definitions` | GET | List | Query params: status |
| `/api/v1/research/regime-definitions/{id}` | GET | Get by ID | `RegimeDefinitionResponse` |
| `/api/v1/research/execution-profiles` | POST | Create | `CreateExecutionProfileRequest` -> `ExecutionProfileResponse` |
| `/api/v1/research/execution-profiles` | GET | List | Query params: status |
| `/api/v1/research/execution-profiles/{id}` | GET | Get by ID | `ExecutionProfileResponse` |
| `/api/v1/research/assets` | POST | Create | `CreateResearchAssetRequest` -> `ResearchAssetResponse` |
| `/api/v1/research/assets` | GET | List | Query params: asset_type, status |
| `/api/v1/research/assets/{id}` | GET | Get by ID | `ResearchAssetResponse` |
| `/api/v1/research/candidates` | POST | Create | `CreateCandidateRequest` -> `CandidateResponse` |
| `/api/v1/research/candidates` | GET | List | Query params: research_group, status |
| `/api/v1/research/candidates/{id}` | GET | Get by ID | `CandidateResponse` |
| `/api/v1/research/candidates/{id}/transition` | POST | State transition | `TransitionRequest` -> validates and transitions state |
| `/api/v1/research/experiments` | POST | Create | `CreateExperimentRequest` -> `ExperimentResponse` |
| `/api/v1/research/experiments` | GET | List | Query params: experiment_type, status |
| `/api/v1/research/experiments/{id}` | GET | Get by ID | `ExperimentResponse` |
| `/api/v1/research/reports` | POST | Create | `CreateReportRequest` -> `ReportResponse` |
| `/api/v1/research/reports` | GET | List | Query params: report_type |
| `/api/v1/research/reports/{id}` | GET | Get by ID | `ReportResponse` |
| `/api/v1/programs` | POST | Create program | `CreateProgramRequest` -> `ProgramResponse` |
| `/api/v1/programs` | GET | List programs | Query params: status |
| `/api/v1/programs/{id}` | GET | Get program | `ProgramResponse` with tracks |
| `/api/v1/programs/{id}/tracks` | POST | Add track | `CreateTrackRequest` -> `TrackResponse` |
| `/api/v1/tracks/{id}/milestones` | POST | Add milestone | `CreateMilestoneRequest` -> `MilestoneResponse` |
| `/api/v1/programs/{id}/status` | GET | Program status overview | Aggregated status of all tracks/milestones |


### 1A.6 State Machine Service

Create `src/amis_core/services/candidate_service.py`.

**Pattern**: Simple validation class, no database logic. Receives `current_status` and `target_status`, returns `(bool, str)` for (allowed, reason).

```python
class CandidateStateMachine:
    VALID_TRANSITIONS = {
        "DRAFT": ["RESEARCHING", "REJECTED"],
        "RESEARCHING": ["VALIDATING", "REJECTED"],
        "VALIDATING": ["SHADOW", "REJECTED"],
        "SHADOW": ["PRODUCTION_CANDIDATE", "REJECTED"],
        "PRODUCTION_CANDIDATE": ["PRODUCTION", "REJECTED"],
        "PRODUCTION": ["REJECTED"],
        "REJECTED": [],  # Terminal
    }

    @classmethod
    def can_transition(cls, from_status: str, to_status: str) -> tuple[bool, str | None]:
        ...
```

Use `@transactional` decorator on service methods that update the database.

### 1A.7 Acceptance Criteria (Phase 1A)

- [ ] All 11 new models exist in `models.py`
- [ ] All existing models have `research_context_id` FK (non-nullable)
- [ ] Database migration creates all tables and FKs
- [ ] All new repositories inherit `BaseRepository`
- [ ] All new API routes return correct responses (test with curl)
- [ ] `POST /api/v1/research/candidates` rejects invalid state transitions
- [ ] `POST /api/v1/research/contexts` computes `context_hash` deterministically
- [ ] `GET /api/v1/research/contexts/{id}/artifacts` returns linked artifacts
- [ ] `GET /api/v1/programs/{id}/status` returns aggregated status

---

## Phase 1B: Lab Research Framework (Weeks 4-6)

**Goal**: Wire Lab's existing research infrastructure into AMIS Core's registry.

### 1B.1 Lab API Routes -- Replace Stubs

All endpoints in `routes_research.py` and `routes_validation.py` currently return `501 NOT_IMPLEMENTED`. Wire them to existing services.

| Endpoint | Existing Service/Script to Call | New/Existing |
|----------|--------------------------------|------------|
| `POST /api/v1/research/experiments` | Create `Experiment` model, register in Core via `ExperimentArtifact` | New service method |
| `GET /api/v1/research/experiments/{id}` | Read from Lab DB | Existing model |
| `POST /api/v1/research/training-runs` | Create `TrainingRun` in Lab, register in Core | New service method |
| `POST /api/v1/validation/walk-forward` | Call `validation_service.run_walk_forward()` | Wire stub |
| `POST /api/v1/validation/shadow-mode` | Call `validation_service.run_shadow_mode()` | Wire stub |
| `GET /api/v1/validation/jobs/{id}` | Read from Lab DB | Existing model |
| `POST /api/v1/validation/scorecards/{id}/submit` | Generate scorecard, submit to Core promotion API | Wire stub |

### 1B.2 Validation Service -- Implement Stubs

File: `src/amis_lab/services/validation_service.py`

The existing methods raise `NotImplementedError`. Implement them using existing scripts as reference:

**`run_walk_forward()`**:
- Reference: `scripts/candidate_001_walk_forward.py`, `scripts/walk_forward_runner.py`
- Steps:
  1. Fetch candles via `candle_fetcher.fetch_historical_candles()`
  2. Extract features via `feature_extractor.extract_features()` or `rg16_features.extract_rg16_features()`
  3. Generate labels via `feature_extractor.generate_labels()` or `rg16_features.generate_rg16_labels()`
  4. Run walk-forward windows (reference: `walk_forward_runner.py`)
  5. Register each artifact in Core: FeatureSchema, Dataset, TrainingRun, ModelArtifact
  6. Create `ValidationJob` in Lab DB
  7. Return `ValidationJobResponse`

**`run_shadow_mode()`**:
- Reference: `scripts/rg18_05_shadow_validation.py`
- Steps:
  1. Deploy model in non-trading mode (paper/simulation)
  2. Collect predictions over target duration
  3. Register shadow validation run in Core: `ShadowValidationRun`
  4. Create daily snapshots: `ShadowValidationSnapshot`
  5. Return shadow run results

**`generate_scorecard()`**:
- Reference: `research/regime_analytics/scorecard.py` (`RegimeScorecard`)
- Steps:
  1. Load model predictions
  2. Compute baseline comparison (`baselines/`)
  3. Compute regime segmentation (`RegimeScorecard`)
  4. Compute gate metrics (walk-forward windows, Expected R lift, calibration)
  5. Submit to Core: `POST /api/v1/promotion/scorecards`
  6. Return scorecard

### 1B.3 Registry Clients -- Implement Stubs

Files: `src/amis_lab/registry/feature_registry.py`, `dataset_registry.py`, `label_registry.py`

These have hash computation working but registration is stubbed. Implement using `BaseServiceClient`:

```python
from smarttrade_common.http_client.service_client import BaseServiceClient

class CoreRegistryClient(BaseServiceClient):
    service_name = "amis-core"
    base_url = "http://amis-core:8015"  # From architecture doc

    async def register_feature_schema(self, schema_data: dict) -> dict:
        return await self.post("/api/v1/registry/features", json=schema_data)

    async def register_dataset(self, dataset_data: dict) -> dict:
        return await self.post("/api/v1/registry/datasets", json=dataset_data)

    async def register_label_version(self, label_data: dict) -> dict:
        return await self.post("/api/v1/registry/labels", json=label_data)
```

Replace individual registry clients with this unified client, or implement the stub methods in existing clients.

### 1B.4 Candidate Lifecycle Integration

Create `src/amis_lab/services/candidate_lifecycle_service.py`.

Orchestrate the candidate state machine in Core:

```python
class CandidateLifecycleService:
    # When candidate transitions to RESEARCHING:
    #   1. Call Core: POST /api/v1/research/candidates/{id}/transition (RESEARCHING)
    #   2. Start feature engineering
    #   3. Register FeatureSchema in Core

    # When candidate transitions to VALIDATING:
    #   1. Call Core: POST /api/v1/research/candidates/{id}/transition (VALIDATING)
    #   2. Call validation_service.run_walk_forward()
    #   3. Register artifacts in Core

    # When candidate transitions to SHADOW:
    #   1. Call Core: POST /api/v1/research/candidates/{id}/transition (SHADOW)
    #   2. Call validation_service.run_shadow_mode()
```

### 1B.5 Acceptance Criteria (Phase 1B)

- [ ] All Lab API endpoints return valid responses (no more 501)
- [ ] `POST /api/v1/validation/walk-forward` runs walk-forward and registers artifacts in Core
- [ ] `POST /api/v1/validation/scorecards/{id}/submit` generates scorecard and submits to Core promotion API
- [ ] Registry clients successfully register FeatureSchema, Dataset, LabelVersion in Core
- [ ] Candidate lifecycle: DRAFT -> RESEARCHING -> VALIDATING works end-to-end
- [ ] Regime analytics (`RegimeScorecard`) integrated into scorecard generation
- [ ] Baseline comparison (`AlwaysTargetBaseline`, `BuyAndHoldBaseline`) included in scorecard


---

## Phase 1C: Operational Lineage (Weeks 7-9)

**Goal**: Dependency lineage flows into Core. Gate 0 enforcement works.

### 1C.1 What Already Works

- `DependencyValidationService` in Lab is fully implemented (353 lines)
- `IndiaVIXValidator` is fully implemented (457 lines)
- Operations repositories are complete
- 2 database migrations exist

### 1C.2 Gate 0 Service

Create `src/amis_core/services/gate0_service.py`.

**Pattern**: Query Lab for dependency health, evaluate Gate 0 criteria, return pass/fail.

```python
class Gate0Service:
    async def evaluate(self, artifact_id: UUID) -> Gate0Result:
        # 1. Call Lab: GET /api/v1/operations/dependencies/health
        # 2. Check: All CRITICAL dependencies HEALTHY
        # 3. Check: No more than 1 HIGH dependency DEGRADED
        # 4. Check: No OPEN CRITICAL or HIGH incidents
        # 5. Check: Data reliability within thresholds
        # 6. Return Gate0Result(passed, details, timestamp)
```

Use `BaseServiceClient` to call Lab's operations API.

### 1C.3 Operations API in Lab

Create `src/amis_lab/api/routes_operations.py`.

Expose dependency health as REST API:

| Endpoint | Method | Service Method |
|----------|--------|---------------|
| `/api/v1/operations/dependencies` | GET | `DependencyValidationService.query_dependency_health()` |
| `/api/v1/operations/dependencies/{id}/health` | GET | `DependencyValidationService.query_dependency_health()` |
| `/api/v1/operations/dependencies/{id}/validate` | POST | `DependencyValidationService.create_validation_run()` |
| `/api/v1/operations/incidents` | GET | `DependencyValidationService.get_incident_history()` |
| `/api/v1/operations/incidents` | POST | `DependencyValidationService.create_incident()` |
| `/api/v1/operations/incidents/{id}/resolve` | POST | `DependencyValidationService.resolve_incident()` |

### 1C.4 Audit Service in Core

Create `src/amis_core/services/audit_service.py`.

**Pattern**: Use `smarttrade_common.database.audit_models` for the base audit model.

```python
class AuditService:
    async def log_action(self, action: str, resource: str, user_id: str,
                         before_state: dict | None, after_state: dict | None) -> None:
        # Uses AuditLog model from smarttrade_common
        # Stores: WHO, WHAT, WHEN, BEFORE, AFTER
```

Wire into:
- Promotion decisions (all approve/reject/rollback)
- Candidate state transitions
- Research context creation
- Gate evaluations

### 1C.5 Acceptance Criteria (Phase 1C)

- [ ] `GET /api/v1/operations/dependencies/health` returns current health status
- [ ] Gate 0 evaluation blocks promotion when CRITICAL dependency is unhealthy
- [ ] Gate 0 evaluation passes when all CRITICAL dependencies are healthy
- [ ] Audit log records every promotion decision with before/after state
- [ ] Incident creation and resolution workflow works end-to-end
- [ ] Dependency lineage: Definition -> ValidationRun -> Snapshot -> Incident queryable

---

## Phase 2: Promotion Engine and Gate Enforcement (Weeks 10-13)

**Goal**: Context Compatibility Gate (Gate 1) works. Promotion workflow is end-to-end.

### 2.1 What Already Works

- `PromotionService` in Core is fully implemented (461 lines)
- Multi-window validation, gate evaluation, approve/reject/rollback all work
- `PromotionDecision` model exists but lacks `evidence_hash`

### 2.2 Add `evidence_hash` to PromotionDecision

Update `src/amis_core/models.py`:

```python
class PromotionDecision(SQLModel, table=True):
    # ... existing fields ...
    evidence_hash: str = Field(nullable=False)  # NEW
    evidence_artifact_ids: list = Field(sa_column=Column(JSON), default_factory=list)  # NEW
```

Create migration: `006_add_evidence_hash_to_promotion_decisions.py`.

### 2.3 Context Compatibility Service (Gate 1)

Create `src/amis_core/services/context_compatibility_service.py`.

```python
class ContextCompatibilityService:
    async def evaluate(self, artifact_id: UUID, deployment_context: dict) -> Gate1Result:
        # 1. Load artifact's ResearchContext from Core
        # 2. Compare each field:
        #    - instrument_ids: deployment instrument must be in validated set
        #    - primary_timeframe: exact match
        #    - higher_timeframe: exact match
        #    - vix_required: must match
        #    - vix_min/max: deployment within validated range
        #    - volatility_regimes: deployment subset of validated
        #    - market_structures: deployment subset of validated
        # 3. Return Gate1Result(passed, mismatches[], timestamp)
```

### 2.4 Update PromotionService

Update `src/amis_core/services/promotion_service.py`:

1. Add `evaluate_gate0()` call at start of `submit_for_review()`
2. Add `evaluate_gate1()` call after Gate 0
3. If Gate 0 fails: immediate rejection, no overrides
4. If Gate 1 fails: immediate rejection, no overrides
5. Gate 2 (Research Validation): existing logic
6. Gate 3 (Shadow Validation): existing logic
7. Gate 4 (Human Approval): existing logic

### 2.5 Wire OperatingEnvelope to ResearchContext

Update `OperatingEnvelopeVersion` creation in Lab:

```python
# When Lab creates an operating envelope:
# 1. Define envelope in Lab (VIX thresholds, etc.)
# 2. Hash the envelope definition
# 3. POST to Core: /api/v1/registry/operating-envelopes
#    - Include research_context_id from the model's context
#    - Include envelope_hash
#    - Include envelope_definition_json (opaque to Core)
```

### 2.6 Acceptance Criteria (Phase 2)

- [ ] `POST /api/v1/promotion/artifacts/{id}/approve` checks Gate 0 first
- [ ] Gate 0 blocks if CRITICAL dependency unhealthy
- [ ] Gate 1 blocks if deployment context mismatches research context
- [ ] Gate 1 shows specific mismatched fields in rejection reason
- [ ] PromotionDecision includes `evidence_hash` computed from all evidence artifacts
- [ ] PromotionDecision is reproducible (evidence hash matches evidence artifacts)
- [ ] Manual override requires justification and is audited


---

## Phase 3: UI Control Tower and Deployment Governance (Weeks 14-17)

**Goal**: SmartTrade UI AMIS module displays programs, tracks, milestones, and context compatibility.

### 3.1 UI Module Structure

Already defined in architecture. Implement in `smarttrade-ui/src/modules/amis/`.

| Module | Files | Data Source |
|--------|-------|-------------|
| `programs/` | ProgramsDashboard, ProgramCard, TrackList, MilestoneList | Core: `/api/v1/programs` |
| `dashboard/` | AMISDashboard, ResearchContextCard | Core: `/api/v1/research/contexts`, Lab: `/api/v1/operations/dependencies/health` |
| `research/` | CandidateList, CandidateDetail, ExperimentList | Core: `/api/v1/research/candidates`, `/api/v1/research/experiments` |
| `governance/` | PromotionQueue, GateEvaluator, DecisionHistory | Core: `/api/v1/promotion/artifacts` |
| `deployments/` | DeploymentMonitor, ContextMatchPanel | Core: `/api/v1/registry/production-deployments` |
| `lineage/` | LineageGraph, ArtifactNode | Core: `/api/v1/registry/lineage/upstream/{id}`, `/api/v1/registry/lineage/downstream/{id}` |

### 3.2 Context Match Panel (Critical UI Component)

Display on every artifact detail page:

```
Research Context (validated)
  Instrument:     NIFTY 50      [MATCH]
  Primary TF:     1H            [MATCH]
  Higher TF:      4H            [MATCH]
  VIX Range:      0-15          [MATCH]
  Volatility:     LOW           [MATCH]
  Market Structure: RANGING   [MATCH]

Deployment Target
  Instrument:     BANKNIFTY     [MISMATCH]
  Primary TF:     15m           [MISMATCH]

Gate 1 Status:    BLOCKED
```

### 3.3 Deployment Page

```
Deploy Model: RG18-Candidate-004B

Research Context: RG18-NIFTY-1H-VIX15
Operating Envelope: RG18-Envelope-v1.2
Execution Profile: Paper-Conservative-10L

Context Compatibility: [CHECKING...]
  Instrument: NIFTY 50 -> NIFTY 50 [MATCH]
  Timeframe: 1H -> 1H [MATCH]
  VIX: <15 -> <15 [MATCH]

All gates passed. Deploy? [Deploy to Paper]
```

### 3.4 Acceptance Criteria (Phase 3)

- [ ] Programs Dashboard shows RG16, RG18 programs with tracks and milestones
- [ ] Blocked milestones show reason (e.g., "DEP-001: India VIX stale")
- [ ] Artifact detail page shows Research Context and context match status
- [ ] Deployment page shows context compatibility check before allowing deploy
- [ ] Deployment blocked if context mismatch detected
- [ ] Lineage explorer shows parent-child graph for any artifact
- [ ] Promotion queue shows pending artifacts with gate status

---

## Phase 4: RG18 Live Validation (Weeks 18-21)

**Goal**: RG18 completes shadow validation and becomes production candidate.

### 4.1 RG18 Context Setup

Create ResearchContext for RG18:

```python
ResearchContext(
    instrument_ids=["NSE:CASH:INDEX:NIFTY50"],
    primary_timeframe="1H",
    higher_timeframe="4H",
    vix_required=True,
    vix_min=0,
    vix_max=15,
    volatility_regimes=["LOW"],
    market_structures=["RANGING", "COMPRESSION"],
)
```

### 4.2 RG18 Candidate Lifecycle

```text
DRAFT (RG18-Candidate-004B)
  ->
RESEARCHING
  -> Feature engineering (rg16_features.py)
  -> Register FeatureSchema in Core
  ->
VALIDATING
  -> Walk-forward validation (walk_forward_runner.py)
  -> Regime analysis (RegimeScorecard)
  -> Register Dataset, TrainingRun, ModelArtifact in Core
  -> Submit scorecard to Core promotion API
  ->
SHADOW
  -> Shadow validation (rg18_05_shadow_validation.py)
  -> Register ShadowValidationRun in Core
  -> Daily snapshots (ShadowValidationSnapshot)
  ->
PRODUCTION_CANDIDATE
  -> All gates pass (Gate 0, Gate 1, Gate 2, Gate 3)
  -> Operating envelope: rg18_03_operating_envelope.py
  -> Register OperatingEnvelopeVersion in Core
  -> Wait for human approval (Gate 4)
  ->
PRODUCTION
  -> Deployment with ExecutionProfile
```

### 4.3 Dependency Health for RG18

Ensure these dependencies are registered and healthy:

| Dependency | Criticality | Validator |
|------------|-------------|-----------|
| India VIX | CRITICAL | `IndiaVIXValidator` |
| MDS (Market Data) | CRITICAL | Built-in |
| NIFTY 1H candles | CRITICAL | Built-in |
| RG16 participation framework | HIGH | Custom |

### 4.4 Acceptance Criteria (Phase 4)

- [ ] RG18 candidate registered in Core with state machine
- [ ] RG18 walk-forward validation complete with 4+ windows
- [ ] RG18 regime analysis shows positive edge in LOW volatility regime
- [ ] RG18 shadow validation runs for minimum 3 months / 1,000 predictions
- [ ] RG18 operating envelope registered in Core with `research_context_id`
- [ ] Gate 0 passes (all critical dependencies healthy)
- [ ] Gate 1 passes (deployment context matches research context)
- [ ] Gate 2 passes (research validation criteria met)
- [ ] Gate 3 passes (shadow validation criteria met)
- [ ] Gate 4 passes (human approval documented)
- [ ] RG18 promoted to production with full evidence hash
- [ ] Deployment tracked with ResearchContext + OperatingEnvelope + ExecutionProfile

---

## smarttrade-common Leverage Map

| What We Need | smarttrade-common Provides | Where to Use |
|--------------|---------------------------|--------------|
| Model base classes | `UUIDMixin`, `TimestampMixin` | All AMIS models |
| Repository base | `BaseRepository[T]` | All AMIS repositories |
| Transaction management | `@transactional` | Service layer methods |
| Database session injection | `get_database_session()` | FastAPI dependencies |
| Service-to-service HTTP | `BaseServiceClient` | Core-Client, Lab-Client |
| Configuration | `CommonSettings` | `AMISCoreSettings`, `AMISLabSettings` |
| Error handling | `SmartTradeError` | All AMIS errors |
| Event-driven integration | `@subscribe`, `EventBus`, `DomainEventPublisher` | Core-Lab events |
| Request tracing | `RequestContextMiddleware` | Both services |
| Background tasks | `system_context()` | Shadow validation, cron jobs |
| Idempotency | `IdempotencyService` | Promotion decisions |
| Transactional events | `OutboxService` | Artifact registration events |
| Health checks | `register_health_check()` | DB, EventBus, dependency health |
| Audit logging | `AuditLog` model | All governance actions |
| Auth/RBAC | `require_policy`, `@public_endpoint` | API routes |
| Rate limiting | `RateLimitMiddleware` | Public-facing endpoints |

---

## Inter-Service Communication Map

| Caller | Endpoint | Callee | Purpose |
|--------|----------|--------|---------|
| Lab | `POST /api/v1/registry/features` | Core | Register feature schema |
| Lab | `POST /api/v1/registry/datasets` | Core | Register dataset |
| Lab | `POST /api/v1/registry/labels` | Core | Register label version |
| Lab | `POST /api/v1/registry/models` | Core | Register model artifact |
| Lab | `POST /api/v1/registry/operating-envelopes` | Core | Register operating envelope |
| Lab | `POST /api/v1/promotion/scorecards` | Core | Submit scorecard |
| Lab | `POST /api/v1/research/candidates/{id}/transition` | Core | Transition candidate state |
| Core | `GET /api/v1/operations/dependencies/health` | Lab | Query dependency health (Gate 0) |
| Core | `GET /api/v1/operations/incidents` | Lab | Query incidents (Gate 0) |
| UI | `GET /api/v1/programs` | Core | Display programs |
| UI | `GET /api/v1/research/contexts` | Core | Display research contexts |
| UI | `GET /api/v1/research/candidates` | Core | Display candidates |
| UI | `GET /api/v1/operations/dependencies/health` | Lab | Display dependency health |
| UI | `GET /api/v1/promotion/artifacts` | Core | Display promotion queue |
| UI | `GET /api/v1/registry/lineage/upstream/{id}` | Core | Display upstream lineage |
| UI | `GET /api/v1/registry/lineage/downstream/{id}` | Core | Display downstream lineage |

