# AMIS Platform Architecture - Final Approved Design

**Document Version**: 2.1  
**Date**: 2026-06-16  
**Status**: Final Approved Architecture  
**Scope**: Complete system architecture for AMIS Core, AMIS Lab, and SmartTrade UI AMIS Module  
**Audience**: Engineering Team, Research Team, Operations Team, Product Owner  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architectural Principles](#2-architectural-principles)
3. [System Architecture](#3-system-architecture)
4. [High Level Design (HLD)](#4-high-level-design-hld)
5. [smarttrade-common Responsibilities](#5-smtrade-common-responsibilities)
6. [AMIS Core Design](#6-amis-core-design)
7. [AMIS Lab Design](#7-amis-lab-design)
8. [SmartTrade UI AMIS Module](#8-smtrade-ui-amis-module)
9. [Data Model Design](#9-data-model-design)
10. [Gate Framework](#10-gate-framework)
11. [API Specifications](#11-api-specifications)
12. [Research Workflow](#12-research-workflow)
13. [Operational Workflow](#13-operational-workflow)
14. [UI User Journeys](#14-ui-user-journeys)
15. [Implementation Roadmap](#15-implementation-roadmap)
16. [Current Program Status](#16-current-program-status)
17. [Risk Analysis](#17-risk-analysis)
18. [Future Extensibility](#18-future-extensibility)

---

## 1. Executive Summary

### 1.1 Platform Purpose

AMIS (AI Market Intelligence System) is a **Decision Governance Platform for Algorithmic Trading** that provides:

- **Research Governance**: Systematic validation, promotion, and deployment of AI trading models and research assets
- **Operational Governance**: Dependency health monitoring, incident management, and production safety (within AMIS Lab)
- **Lineage & Auditability**: Complete artifact lineage from raw data to production deployment
- **Mandatory Regime Governance**: India VIX, Volatility, Market Structure, and HTF Alignment analysis for all research
- **Decision Traceability**: Every production decision explainable through research lineage, operational lineage, promotion evidence, and audit trail

### 1.2 Business Objectives

| Objective | Metric |
|-----------|--------|
| Systematic research validation with multi-window walk-forward | 4+ independent windows required for production |
| Operational dependency blocking for unsafe deployments | 100% of dependency failures block promotion (Gate 0) |
| Complete artifact lineage and auditability | 100% of production artifacts traceable to source |
| Mandatory regime governance | 100% of research includes VIX, Volatility, Market Structure, HTF Alignment |
| Research-Operations separation | Zero operational dependencies in research path |
| UI-driven R&D workflows | 100% of research workflows accessible via SmartTrade UI |

### 1.3 Key Discoveries from Research

- **RG16**: Failed as alpha model but succeeded as participation framework and regime discovery framework
- **RG18**: Positive edge but requires VIX and RG16 gating; currently SHADOW_PRODUCTION_CANDIDATE
- **India VIX**: Critical regime discriminator for all models
- **Operational Dependency Failures**: Must block deployment via Gate 0
- **Research vs Operations**: Separate governance concerns, but operational capabilities live in AMIS Lab

### 1.4 Decision Governance Philosophy

AMIS is not merely a model registry. AMIS is a **Decision Governance Platform**.

Every production decision must be explainable through:
- **Research Lineage**: What research led to this decision?
- **Operational Lineage**: What operational state enabled this decision?
- **Promotion Evidence**: What gates were passed and why?
- **Dependency Health**: What operational dependencies were satisfied?
- **Audit Trail**: Who approved this decision and when?

---

## 2. Architectural Principles

1. **Keep it Simple**: Avoid unnecessary complexity and over-engineering
2. **Separate Governance from Research**: AMIS Core governs, AMIS Lab researches
3. **Use Common Library First**: All shared infrastructure and contracts live in smarttrade-common
4. **No Separate AMIS UI Service**: AMIS is a module inside smarttrade-ui
5. **No Separate AMIS Operations Service**: All operational capabilities belong to AMIS Lab
6. **Operational Capabilities in Lab**: Dependency monitoring, incident management, shadow validation live in AMIS Lab
7. **UI Module in SmartTrade UI**: AMIS UI is `src/modules/amis` in smarttrade-ui
8. **Mandatory Regime Governance**: VIX, Volatility, Market Structure, HTF Alignment required for all research
9. **Decision Traceability**: Every production decision must be traceable through lineage
10. **Data-Driven Governance**: All governance decisions must be data-driven and auditable

---

## 3. System Architecture

### 3.1 Overall Platform Architecture

```mermaid
graph TB
    subgraph "External Systems"
        MDS[Market Data Service]
        BAS[Broker Adapter Service]
        AUTH[Authentication Service]
    end
    
    subgraph "Shared Infrastructure"
        COMMON[smarttrade-common]
        REDIS[(Redis Event Bus)]
        ARTIFACT_STORE[(Artifact Store)]
    end
    
    subgraph "AMIS Core - Governance System"
        CORE_REGISTRY[Registry Service]
        CORE_PROMOTION[Promotion Service]
        CORE_GATE[Gate Engine]
        CORE_LINEAGE[Artifact Lineage]
        CORE_AUDIT[Audit Service]
        CORE_DEPLOY[Deployment Governance]
    end
    
    subgraph "AMIS Lab - Research and Validation System"
        LAB_RESEARCH[Research Module]
        LAB_VALIDATION[Validation Module]
        LAB_OPERATIONS[Operations Module]
        LAB_REPORTING[Reporting Module]
    end
    
    subgraph "SmartTrade UI"
        UI_AMIS[AMIS Module]
    end
    
    subgraph "Databases"
        PG_CORE[(AMIS Core DB)]
        PG_LAB[(AMIS Lab DB)]
    end
    
    COMMON -->|Shared Contracts| CORE_REGISTRY
    COMMON -->|Shared Contracts| LAB_RESEARCH
    COMMON -->|Shared Contracts| LAB_OPERATIONS
    
    MDS -->|market.quote| LAB_VALIDATION
    MDS -->|market.quote| LAB_OPERATIONS
    
    AUTH -->|JWT| CORE_REGISTRY
    AUTH -->|JWT| LAB_RESEARCH
    
    LAB_RESEARCH -->|promotion.artifacts| CORE_PROMOTION
    LAB_VALIDATION -->|promotion.scorecards| CORE_PROMOTION
    LAB_OPERATIONS -->|gate0.health| CORE_GATE
    
    CORE_PROMOTION -->|deployment.decision| CORE_DEPLOY
    CORE_DEPLOY -->|production.model| BAS
    
    UI_AMIS -->|REST| CORE_REGISTRY
    UI_AMIS -->|REST| LAB_RESEARCH
    UI_AMIS -->|REST| LAB_OPERATIONS
    
    CORE_REGISTRY --> PG_CORE
    CORE_PROMOTION --> PG_CORE
    LAB_RESEARCH --> PG_LAB
    LAB_VALIDATION --> PG_LAB
    LAB_OPERATIONS --> PG_LAB
    
    CORE_LINEAGE --> ARTIFACT_STORE
    LAB_RESEARCH --> ARTIFACT_STORE
    
    CORE_REGISTRY -.->|EventBus| REDIS
    CORE_PROMOTION -.->|EventBus| REDIS
    LAB_RESEARCH -.->|EventBus| REDIS
    LAB_OPERATIONS -.->|EventBus| REDIS
```

### 3.2 Repository Structure

```text
smarttrade-common/
├── src/smarttrade_common/
│   ├── config/
│   ├── database/
│   ├── events/
│   ├── middleware/
│   ├── resilience/
│   ├── security/
│   └── contracts/
│       ├── research_status.py
│       ├── dependency_health.py
│       └── artifact_reference.py

smarttrade-amis-core/
├── src/amis_core/
│   ├── registry/
│   ├── promotion/
│   ├── gates/
│   ├── lineage/
│   ├── audit/
│   └── deployment/

smarttrade-amis-lab/
├── src/amis_lab/
│   ├── research/
│   ├── validation/
│   ├── operations/
│   └── reporting/

smarttrade-ui/
└── src/modules/amis/
    ├── dashboard/
    ├── programs/
    ├── research/
    ├── validation/
    ├── governance/
    ├── operations/
    ├── lineage/
    ├── deployments/
    └── shared/
```

### 3.3 Service Ownership Boundaries

| Service | Domain | Database | Port |
|---------|--------|----------|------|
| **AMIS Core** | Governance System | `smarttrade_amis_core` | 8015 |
| **AMIS Lab** | Research and Validation System | `smarttrade_amis_lab` | 8016 |
| **SmartTrade UI** | User Interface (AMIS Module) | — | 5173 |

### 3.4 External Dependencies

| Service | Dependency | Purpose |
|---------|------------|---------|
| AMIS Core | Authentication Service | JWT validation, RBAC |
| AMIS Core | Market Data Service | Historical data for validation |
| AMIS Lab | Market Data Service | Real-time and historical candles |
| AMIS Lab | AMIS Core | Artifact registration, promotion |
| SmartTrade UI | AMIS Core | Governance dashboard data |
| SmartTrade UI | AMIS Lab | Research and operations data |

### 3.5 Event Bus Architecture

**EventBus Pattern** (Domain Events):
- `research.asset_created` - Lab publishes, Core subscribes
- `research.validation_completed` - Lab publishes, Core subscribes
- `promotion.artifact_submitted` - Core publishes, UI subscribes
- `promotion.gate_passed` - Core publishes, UI subscribes
- `promotion.decision_made` - Core publishes, Lab subscribes
- `operational.dependency_failed` - Lab publishes, Core subscribes
- `operational.incident_created` - Lab publishes, UI subscribes

**BaseStreamConsumer Pattern** (Market Data):
- `market.quote` - Lab consumes for real-time analysis
- `market.candle` - Lab consumes for historical backfill

---

## 4. High Level Design (HLD)

### 4.1 Service Decomposition

#### AMIS Core (Governance System)
- **Registry Service**: Immutable artifact registry (features, datasets, models, labels, research assets)
- **Promotion Service**: Gate evaluation, promotion decisions, deployment orchestration
- **Gate Engine**: Configurable gate definitions and evaluation logic
- **Artifact Lineage**: Complete parent-child graph for all artifacts
- **Audit Service**: Immutable audit log for all governance actions
- **Deployment Governance**: Production deployment safety and rollback

**AMIS Core answers**:
- What exists?
- What is approved?
- What is production?
- Why was it approved?

**AMIS Core must never**:
- Train models
- Run validation
- Calculate VIX
- Run shadow mode
- Perform research

#### AMIS Lab (Research and Validation System)

**Research Module**:
- Candidate Research
- Feature Engineering
- Dataset Generation
- Training
- Feature Families

**Validation Module**:
- Walk Forward Validation
- Temporal Validation
- Regime Validation
- Shadow Validation

**Operations Module**:
- Dependency Monitoring
- Incident Management
- Dependency Validation
- Data Reliability Monitoring

**Reporting Module**:
- Research Reports
- Validation Reports
- Regime Reports
- Shadow Reports

**AMIS Lab answers**:
- Does it work?
- Why does it work?
- Can it survive deployment?

#### SmartTrade UI (AMIS Module)

**AMIS Dashboard**: Mission Control Center
- Research Assets
- Production Candidates
- Production Models
- Open Incidents
- Dependency Health
- Shadow Validation Status
- Promotion Queue

**Sub-modules**:
- Research Dashboard
- Validation Dashboard
- Governance Dashboard
- Operations Dashboard
- Lineage Explorer
- Deployment Monitor

### 4.2 Database Ownership

| Database | Owner | Tables |
|----------|-------|--------|
| `smarttrade_amis_core` | AMIS Core | feature_schemas, label_versions, stop_constraint_versions, datasets, training_runs, model_artifacts, research_assets, promotion_scorecards, promotion_decisions, gate_definitions, audit_log, operating_envelope_versions, production_deployments, artifact_lineage |
| `smarttrade_amis_lab` | AMIS Lab | candidates, experiments, feature_families, validation_jobs, regime_analyses, vix_analyses, dependency_definitions, dependency_validation_runs, dependency_validation_snapshots, dependency_incidents, data_reliability_metrics, shadow_validation_metrics, research_reports |

### 4.3 Authentication Model

- **JWT-based authentication** via smarttrade-common
- **Shared JWT secret** with Authentication Service
- **Token-based authorization** with role claims
- **Service-to-service** authentication using service accounts

### 4.4 RBAC Model

| Role | Permissions |
|------|-------------|
| `amis_researcher` | Create candidates, run experiments, submit artifacts |
| `amis_reviewer` | Review artifacts, approve/reject promotions |
| `amis_operator` | Monitor operations, manage incidents |
| `amis_admin` | Full administrative access |
| `amis_lab_pipeline` | Service account for Lab automation |
| `amis_core_pipeline` | Service account for Core automation |

---

## 5. smarttrade-common Responsibilities

The common repository owns all shared infrastructure and contracts.

### 5.1 Shared Infrastructure

- Configuration
- Database
- Session Management
- Repository Base Classes
- Event Dispatcher
- Event Contracts
- Retry Policies
- Circuit Breakers
- RBAC
- Authentication Helpers
- API Response Models

### 5.2 Shared Contracts

#### ResearchStatus

```python
from enum import Enum

class ResearchStatus(str, Enum):
    EXPERIMENTAL = "EXPERIMENTAL"
    RESEARCH_SIGNAL_DISCOVERED = "RESEARCH_SIGNAL_DISCOVERED"
    RESEARCH_EXPLANATION_DISCOVERED = "RESEARCH_EXPLANATION_DISCOVERED"
    RESEARCH_VALIDATED = "RESEARCH_VALIDATED"
    OPERATIONALLY_VALIDATED = "OPERATIONALLY_VALIDATED"
    SHADOW_PRODUCTION_CANDIDATE = "SHADOW_PRODUCTION_CANDIDATE"
    PRODUCTION = "PRODUCTION"
    REJECTED = "REJECTED"
```

#### DependencyHealth

```python
from enum import Enum

class DependencyHealth(str, Enum):
    HEALTHY = "HEALTHY"
    DEGRADED = "DEGRADED"
    FAILED = "FAILED"
```

#### ArtifactReference

```python
from pydantic import BaseModel
from uuid import UUID

class ArtifactReference(BaseModel):
    artifact_id: UUID
    artifact_type: str
    semantic_version: str
```

All shared enums and contracts must live in smarttrade-common.

---

## 6. AMIS Core Design

### 6.1 Registry Service

**Responsibilities**: Immutable artifact registry with content-addressable identity

**Key Features**:
- Content-addressable artifact storage (hash-based identity)
- Immutable artifact versions (no updates, only new versions)
- Complete lineage tracking (parent-child relationships)
- Metadata and provenance tracking

**API Endpoints**:
- `POST /api/v1/registry/features` - Register feature schema
- `POST /api/v1/registry/labels` - Register label version
- `POST /api/v1/registry/datasets` - Register dataset
- `POST /api/v1/registry/models` - Register model artifact
- `POST /api/v1/registry/research-assets` - Register research asset
- `GET /api/v1/registry/features/{id}` - Get feature schema
- `GET /api/v1/registry/lineage/{artifact_id}` - Get artifact lineage

**Data Models**:
- `FeatureSchema` - Feature schema definitions
- `LabelVersion` - Label computation versions
- `StopConstraintVersion` - Stop constraint parameter versions
- `Dataset` - Training dataset registry
- `TrainingRun` - Training run registry
- `ModelArtifact` - Model artifact registry
- `ResearchAsset` - Research asset registry (NEW)

### 6.2 ResearchAsset (New First-Class Artifact)

**Purpose**: A model can fail while the knowledge survives.

**Examples**:
- RG16 Alpha Model → REJECTED
- RG16 Participation Framework → RESEARCH_VALIDATED
- India VIX Discovery → RESEARCH_EXPLANATION_DISCOVERED
- RG18 Operating Envelope → SHADOW_PRODUCTION_CANDIDATE

**Data Model**:
```python
class ResearchAsset(UUIDMixin, TimestampMixin, table=True):
    __tablename__ = "research_assets"
    
    asset_type: str = Field(index=True, nullable=False)  # MODEL, FRAMEWORK, DISCOVERY, ENVELOPE
    name: str = Field(unique=True, index=True, nullable=False)
    description: Optional[str] = Field(default=None)
    status: str = Field(default="EXPERIMENTAL", index=True)  # Uses ResearchStatus enum
    
    parent_artifact_ids: list = Field(sa_column=Column(JSON), default_factory=list)
    evidence_references: list = Field(sa_column=Column(JSON), default_factory=list)
    semantic_version: str = Field(nullable=False)
    
    created_by: str = Field(nullable=False)
    
    extra_metadata: dict = Field(default_factory=dict, alias="metadata", sa_column=Column("metadata", JSON))
```

**ResearchAsset must support**:
- asset_type (MODEL, FRAMEWORK, DISCOVERY, ENVELOPE)
- name
- description
- status (from ResearchStatus enum)
- parent_artifact_ids
- evidence_references
- semantic_version

### 6.3 ExperimentArtifact (New First-Class Artifact)

**Purpose**: Track experiments as searchable, versioned assets.

**Examples**:
- TRACK-RG16-GATING
- TRACK-RG18-04C
- TRACK-RG18-05

**Data Model**:
```python
class ExperimentArtifact(UUIDMixin, TimestampMixin, table=True):
    __tablename__ = "experiment_artifacts"
    
    experiment_name: str = Field(unique=True, index=True, nullable=False)
    experiment_type: str = Field(nullable=False)  # HYPOTHESIS, VALIDATION, ABLATION
    status: str = Field(default="EXPERIMENTAL", index=True)
    
    # Lab that performed the experiment
    performing_lab: str = Field(nullable=False)
    
    # Opaque experiment definition (Lab owns the schema)
    experiment_definition_json: dict = Field(sa_column=Column(JSON, nullable=False))
    
    # Results
    results_json: Optional[dict] = Field(default=None, sa_column=Column(JSON))
    conclusion: Optional[str] = Field(default=None)
    
    parent_artifact_ids: list = Field(sa_column=Column(JSON), default_factory=list)
    semantic_version: str = Field(nullable=False)
    
    created_by: str = Field(nullable=False)
    
    extra_metadata: dict = Field(default_factory=dict, alias="metadata", sa_column=Column("metadata", JSON))
```

### 6.4 Candidate Registration (First-Class Artifact in Core)

**Purpose**: Register Candidate in Core once created for governance lineage.

**Data Model**:
```python
class CandidateArtifact(UUIDMixin, TimestampMixin, table=True):
    __tablename__ = "candidate_artifacts"
    
    candidate_name: str = Field(unique=True, index=True, nullable=False)
    research_group: str = Field(index=True, nullable=False)  # RG16, RG17, RG18
    candidate_type: str = Field(nullable=False)  # ALPHA, PARTICIPATION, OPPORTUNITY
    status: str = Field(default="DRAFT", index=True)
    
    feature_schema_id: Optional[UUID] = Field(default=None)
    label_version_id: Optional[UUID] = Field(default=None)
    base_model_id: Optional[UUID] = Field(default=None)
    
    # Lab that created the candidate
    originating_lab: str = Field(nullable=False)
    
    semantic_version: str = Field(nullable=False)
    
    created_by: str = Field(nullable=False)
    
    extra_metadata: dict = Field(default_factory=dict, alias="metadata", sa_column=Column("metadata", JSON))
```

**Lineage Flow**:
```text
CandidateArtifact → ExperimentArtifact → Validation → ResearchAsset
```

### 6.5 Report Artifacts (Immutable in Core)

**Purpose**: Promotion decisions must be traceable to the exact report version used.

**Types**:
- `ResearchReportArtifact`
- `ValidationReportArtifact`
- `RegimeReportArtifact`

**Data Model**:
```python
class ReportArtifact(UUIDMixin, TimestampMixin, table=True):
    __tablename__ = "report_artifacts"
    
    report_type: str = Field(index=True, nullable=False)  # RESEARCH, VALIDATION, REGIME
    report_name: str = Field(nullable=False)
    
    # Content-addressable identity
    report_hash: str = Field(unique=True, index=True, nullable=False)
    semantic_version: str = Field(nullable=False)
    
    # Storage location
    storage_backend: str = Field(default="filesystem", nullable=False)
    storage_path: str = Field(nullable=False)
    storage_size_bytes: int = Field(nullable=False)
    storage_checksum: str = Field(nullable=False)
    
    # Linked artifacts
    linked_artifact_ids: list = Field(sa_column=Column(JSON), default_factory=list)
    
    # Lab that generated the report
    generating_lab: str = Field(nullable=False)
    
    created_by: str = Field(nullable=False)
    
    extra_metadata: dict = Field(default_factory=dict, alias="metadata", sa_column=Column("metadata", JSON))
```

### 6.6 Research Program Management (New in Core)

**Purpose**: Drive R&D from UI. AMIS becomes a real R&D control tower.

**Hierarchy**:
```text
ResearchProgram
    ↓
ResearchTrack
    ↓
ResearchMilestone
```

**Example**:
```text
Program: RG18
    Track: Validation
        Milestone: RG18-01 (Feature Engineering)
        Milestone: RG18-02 (Model Training)
        Milestone: RG18-03 (Walk-Forward)
    Track: Envelope
        Milestone: RG18-04 (VIX Threshold)
        Milestone: RG18-05 (Dependency Health)
    Track: Shadow
        Milestone: RG18-06 (Shadow Launch)
        Milestone: RG18-07 (Shadow Validation)
```

**Data Models**:

```python
class ResearchProgram(UUIDMixin, TimestampMixin, table=True):
    __tablename__ = "research_programs"
    
    program_name: str = Field(unique=True, index=True, nullable=False)
    description: Optional[str] = Field(default=None)
    status: str = Field(default="ACTIVE", index=True)  # ACTIVE, FROZEN, COMPLETED
    
    lead_researcher: str = Field(nullable=False)
    
    created_by: str = Field(nullable=False)
    
    extra_metadata: dict = Field(default_factory=dict, alias="metadata", sa_column=Column("metadata", JSON))

class ResearchTrack(UUIDMixin, TimestampMixin, table=True):
    __tablename__ = "research_tracks"
    
    program_id: UUID = Field(foreign_key="research_programs.id", index=True)
    track_name: str = Field(nullable=False)
    description: Optional[str] = Field(default=None)
    status: str = Field(default="ACTIVE", index=True)
    
    created_by: str = Field(nullable=False)
    
    extra_metadata: dict = Field(default_factory=dict, alias="metadata", sa_column=Column("metadata", JSON))

class ResearchMilestone(UUIDMixin, TimestampMixin, table=True):
    __tablename__ = "research_milestones"
    
    track_id: UUID = Field(foreign_key="research_tracks.id", index=True)
    milestone_name: str = Field(nullable=False)
    description: Optional[str] = Field(default=None)
    status: str = Field(default="PENDING", index=True)  # PENDING, IN_PROGRESS, COMPLETED, BLOCKED
    
    # Linked artifacts
    linked_artifact_ids: list = Field(sa_column=Column(JSON), default_factory=list)
    
    target_date: Optional[datetime] = Field(default=None)
    completed_date: Optional[datetime] = Field(default=None)
    
    created_by: str = Field(nullable=False)
    
    extra_metadata: dict = Field(default_factory=dict, alias="metadata", sa_column=Column("metadata", JSON))
```

**API Endpoints**:
- `POST /api/v1/programs` - Create research program
- `GET /api/v1/programs` - List research programs
- `POST /api/v1/programs/{id}/tracks` - Add track to program
- `POST /api/v1/tracks/{id}/milestones` - Add milestone to track
- `GET /api/v1/programs/{id}/status` - Get program status overview

### 6.7 Promotion Service

**Responsibilities**: Gate evaluation, promotion decisions, deployment orchestration

**Key Features**:
- Configurable gate definitions
- Multi-gate evaluation (Gate 0-3)
- Baseline comparison
- Promotion decision workflow
- Deployment orchestration
- Rollback management

**API Endpoints**:
- `POST /api/v1/promotion/artifacts` - Submit artifact for promotion
- `POST /api/v1/promotion/scorecards` - Submit validation scorecard
- `POST /api/v1/promotion/artifacts/{id}/submit` - Request promotion
- `POST /api/v1/promotion/artifacts/{id}/approve` - Approve promotion
- `POST /api/v1/promotion/artifacts/{id}/reject` - Reject promotion
- `POST /api/v1/promotion/artifacts/{id}/rollback` - Rollback deployment
- `GET /api/v1/promotion/artifacts/{id}/status` - Get promotion status
- `GET /api/v1/promotion/current-production` - Get current production artifact

**Data Models**:
- `PromotionScorecard` - Validation scorecard
- `PromotionDecision` - Promotion decision record
- `GateDefinition` - Gate configuration
- `GateEvaluation` - Gate evaluation result

### 6.8 Gate Engine

**Responsibilities**: Configurable gate definitions and evaluation logic

**Key Features**:
- Configurable gate definitions
- Pluggable gate evaluators
- Gate dependency management
- Gate status aggregation

**Gate Definitions**:
- **Gate 0**: Dependency Health - Operational dependency validation (MANDATORY, no overrides)
- **Gate 1**: Research Validation - Walk-forward validation, regime analysis
- **Gate 2**: Shadow Validation - Shadow mode performance validation
- **Gate 3**: Human Approval - Manual review and approval

### 6.9 Artifact Lineage

**Responsibilities**: Complete parent-child graph for all artifacts

**Key Features**:
- Generic parent-child relationships
- Lineage query API
- Lineage visualization support
- Impact analysis (what breaks if this artifact changes)

**Core owns ALL lineage** — both research and operational.

Lab generates artifacts. Core stores lineage.

**Lineage Types**:
- **Research Lineage**: FeatureSchema → Dataset → TrainingRun → ModelArtifact → PromotionDecision
- **Operational Lineage**: DependencyDefinition → DependencyValidationRun → DependencyValidationSnapshot → DependencyIncident

**Unified Lineage Query**: Everything is queryable from one place in Core.

```text
FeatureSchema
Dataset
TrainingRun
ModelArtifact
ResearchAsset
OperatingEnvelope
DependencyDefinition
DependencyValidationRun
DependencyIncident
PromotionDecision
Candidate
ExperimentArtifact
ReportArtifact
```

### 6.10 Audit Service

**Responsibilities**: Immutable audit log for all governance actions

**Key Features**:
- Immutable audit log (append-only)
- Complete action tracking
- User attribution
- Query and export capabilities

### 6.11 Deployment Governance

**Responsibilities**: Production deployment safety and rollback

**Key Features**:
- Deployment safety checks
- Deployment orchestration
- Rollback management
- Deployment status tracking

**Data Models**:
- `OperatingEnvelopeVersion` - Operating envelope version registry
- `ProductionDeployment` - Production deployment tracking

**OperatingEnvelopeVersion**:

Core stores envelopes. Lab defines envelopes. Core never understands trading-specific concepts.

```python
class OperatingEnvelopeVersion(UUIDMixin, TimestampMixin, table=True):
    __tablename__ = "operating_envelope_versions"
    
    envelope_name: str = Field(unique=True, index=True, nullable=False)
    semantic_version: str = Field(nullable=False)
    
    # Lab-defined envelope content (opaque to Core)
    envelope_definition_json: dict = Field(sa_column=Column(JSON, nullable=False))
    
    # Lab that defined this envelope
    defining_lab: str = Field(nullable=False)  # e.g., "amis-lab"
    
    status: str = Field(default="EXPERIMENTAL", index=True)
    
    created_by: str = Field(nullable=False)
    
    extra_metadata: dict = Field(default_factory=dict, alias="metadata", sa_column=Column("metadata", JSON))
```

**Core should never understand**: VIX, RG16, RG18, ATR, Compression, or any trading-specific concepts. Those belong in Lab.

---

## 7. AMIS Lab Design

### 7.1 Research Module

#### Candidate Research
- Candidate lifecycle management
- Experiment tracking
- Feature family management

#### Feature Engineering
- Feature extraction from market data
- Dataset generation
- Feature schema registration
- Dataset quality reporting

#### Training
- Training run management
- Hyperparameter logging
- Model artifact generation

### 7.2 Validation Module

#### Walk Forward Validation
- Multi-window walk-forward validation
- Temporal validation
- Performance aggregation

#### Regime Validation
- Regime detection (volatility, trend, structure)
- Regime-specific performance analysis
- Regime transition detection
- Regime attribution

#### Shadow Validation
- Shadow mode performance tracking
- Live vs shadow comparison
- Drift detection
- Shadow-to-production promotion readiness

### 7.3 Operations Module

#### Dependency Monitoring
- Dependency registration
- Health check scheduling
- Health status aggregation
- Alert generation

#### Incident Management
- Incident creation and tracking
- Incident severity classification
- Incident assignment and escalation
- Incident resolution and post-mortem

#### Dependency Validation
- Validation job scheduling
- Validation execution
- Result storage
- Snapshot management

#### Data Reliability Monitoring
- Data quality metrics
- Gap detection
- Latency monitoring
- Data freshness tracking

### 7.4 Reporting Module

#### Research Reports
- Automated research report generation
- Template-based reporting
- Multi-format output (PDF, HTML, Markdown)

#### Validation Reports
- Validation scorecard generation
- Gate evaluation reports
- Performance metrics reports

#### Regime Reports
- Regime analysis reports
- VIX analysis reports
- Regime stability reports

#### Shadow Reports
- Shadow validation reports
- Drift analysis reports
- Promotion readiness assessments

### 7.5 Mandatory Regime Governance

**Current Required Regime Dimensions**:
1. India VIX
2. Volatility Regime
3. Market Structure
4. HTF Alignment

**Future Dimensions May Include**:
- Market Breadth
- Sector Leadership
- Liquidity
- Macro Regimes

**All Future Candidates Must Include Regime Analysis**.

---

## 8. SmartTrade UI AMIS Module

### 8.1 Module Structure

```text
src/modules/amis/
├── dashboard/           # AMIS Dashboard (Mission Control)
├── research/            # Research workflows
├── validation/          # Validation workflows
├── governance/          # Governance workflows
├── operations/          # Operations monitoring
├── lineage/             # Lineage exploration
├── deployments/         # Deployment monitoring
└── shared/              # Shared components
```

### 8.2 AMIS Dashboard (Mission Control Center)

**Purpose**: Default landing page for AMIS

**Display**:
- Research Assets
- Production Candidates
- Production Models
- Open Incidents
- Dependency Health
- Shadow Validation Status
- Promotion Queue

### 8.3 Programs Dashboard (R&D Control Tower)

**Purpose**: Drive R&D from UI. Programs become the landing experience.

**Display**:
- Active Research Programs (e.g., RG16, RG18)
- Program Status (Frozen, Active, Shadow Candidate)
- Tracks per Program (Validation, Envelope, Shadow)
- Milestones per Track
- Blocked Milestones with reasons

**Example View**:
```text
RG16
  Status: FROZEN

RG18
  Status: Shadow Candidate
  Track: Validation
    Milestone: RG18-05 (Dependency Health) — BLOCKED by DEP-001

Operations
  Status: Blocked by DEP-001
```

**Key Features**:
- Program creation and management
- Track and milestone tracking
- Program status overview
- Drill-down to artifacts, validations, and deployments

### 8.4 Research Dashboard

**Purpose**: Research workflow orchestration

**Key Features**:
- Candidate creation wizard
- Experiment tracking
- Validation job monitoring
- Research report generation

### 8.5 Validation Dashboard

**Purpose**: Validation monitoring and analysis

**Key Features**:
- Validation job tracking
- Regime analysis visualization
- Shadow validation monitoring
- Performance metrics

### 8.6 Governance Dashboard

**Purpose**: Governance workflow management

**Key Features**:
- Promotion request list
- Gate evaluation visualization
- Approval/rejection workflow
- Rollback management

### 8.7 Operations Dashboard

**Purpose**: Operational health monitoring

**Key Features**:
- Dependency health visualization
- Incident tracking
- Data reliability metrics
- Alert configuration

### 8.8 Lineage Explorer

**Purpose**: Artifact lineage visualization

**Key Features**:
- Research lineage visualization
- Operational lineage visualization
- Impact analysis
- Decision traceability

### 8.9 Deployment Monitor

**Purpose**: Production deployment monitoring

**Key Features**:
- Current production artifact
- Deployment history
- Operating envelope status
- Rollback status

---

## 9. Data Model Design

### 9.1 Entity Relationship Diagram

```mermaid
erDiagram
    %% Core Registry Entities
    feature_schemas ||--o{ datasets : "defines"
    label_versions ||--o{ datasets : "defines"
    stop_constraint_versions ||--o{ datasets : "constrains"
    datasets ||--o{ training_runs : "trains"
    training_runs ||--o{ model_artifacts : "produces"
    model_artifacts ||--o{ promotion_scorecards : "evaluates"
    promotion_scorecards ||--o{ promotion_decisions : "informs"
    research_assets ||--o{ promotion_decisions : "governs"
    candidate_artifacts ||--o{ experiment_artifacts : "has"
    candidate_artifacts ||--o{ validation_jobs : "validates"
    experiment_artifacts ||--o{ model_artifacts : "produces"
    report_artifacts ||--o{ promotion_scorecards : "supports"
    operating_envelope_versions ||--o{ production_deployments : "defines"
    research_programs ||--o{ research_tracks : "has"
    research_tracks ||--o{ research_milestones : "contains"
    artifact_lineage ||--o{ model_artifacts : "parent"
    artifact_lineage ||--o{ model_artifacts : "child"
    
    %% Lab Research Entities
    validation_jobs ||--o{ promotion_scorecards : "generates"
    validation_jobs ||--o{ regime_analyses : "includes"
    validation_jobs ||--o{ vix_analyses : "includes"
    
    %% Lab Operational Entities
    dependency_definitions ||--o{ dependency_validation_runs : "validates"
    dependency_definitions ||--o{ dependency_incidents : "has"
    dependency_validation_runs ||--o{ dependency_validation_snapshots : "captures"
    
    model_artifacts {
        UUID id PK
        str artifact_hash UK
        str semantic_version
        UUID training_run_id FK
        dict hyperparameters
        dict training_metrics
        str status
    }
    
    research_assets {
        UUID id PK
        str asset_type
        str name UK
        str status
        list parent_artifact_ids
        list evidence_references
        str semantic_version
    }
    
    candidate_artifacts {
        UUID id PK
        str candidate_name UK
        str research_group
        str candidate_type
        str status
        UUID feature_schema_id FK
        UUID label_version_id FK
        str originating_lab
        str semantic_version
    }
    
    experiment_artifacts {
        UUID id PK
        str experiment_name UK
        str experiment_type
        str status
        str performing_lab
        dict experiment_definition_json
        dict results_json
        list parent_artifact_ids
        str semantic_version
    }
    
    report_artifacts {
        UUID id PK
        str report_type
        str report_name
        str report_hash UK
        str semantic_version
        str storage_path
        list linked_artifact_ids
        str generating_lab
    }
    
    feature_schemas {
        UUID id PK
        str schema_hash UK
        str semantic_version
        str domain
        list feature_definitions
    }
    
    label_versions {
        UUID id PK
        str label_hash UK
        str semantic_version
        str domain
        str label_type
        dict computation_logic
    }
    
    datasets {
        UUID id PK
        str dataset_hash UK
        str semantic_version
        UUID feature_schema_id FK
        UUID label_version_id FK
        UUID stop_constraint_version_id FK
        str storage_path
    }
    
    training_runs {
        UUID id PK
        str run_hash UK
        UUID dataset_id FK
        str status
        dict hyperparameters
        dict training_metrics
    }
    
    promotion_scorecards {
        UUID id PK
        UUID artifact_id FK
        str validation_type
        dict gates
        dict metrics
        bool overall_passed
    }
    
    promotion_decisions {
        UUID id PK
        UUID artifact_id FK
        UUID scorecard_id FK
        str decision
        str reason
        str decided_by
    }
    
    operating_envelope_versions {
        UUID id PK
        str envelope_name UK
        str semantic_version
        dict envelope_definition_json
        str defining_lab
        str status
    }
    
    research_programs {
        UUID id PK
        str program_name UK
        str description
        str status
        str lead_researcher
    }
    
    research_tracks {
        UUID id PK
        UUID program_id FK
        str track_name
        str description
        str status
    }
    
    research_milestones {
        UUID id PK
        UUID track_id FK
        str milestone_name
        str description
        str status
        list linked_artifact_ids
        datetime target_date
        datetime completed_date
    }
    
    validation_jobs {
        UUID id PK
        str job_name
        UUID candidate_id FK
        str validation_type
        str status
        list walk_forward_windows
        dict results
    }
    
    regime_analyses {
        UUID id PK
        UUID candidate_id FK
        UUID validation_job_id FK
        str regime_type
        dict regime_classification
        dict regime_performance
    }
    
    vix_analyses {
        UUID id PK
        UUID candidate_id FK
        UUID validation_job_id FK
        str vix_regime
        dict vix_performance
        bool includes_regime_governance
    }
    
    dependency_definitions {
        UUID id PK
        str dependency_name UK
        str dependency_type
        str criticality
        str current_status
    }
    
    dependency_validation_runs {
        UUID id PK
        UUID dependency_id FK
        str run_type
        str status
        dict validation_results
        bool passed
    }
    
    dependency_incidents {
        UUID id PK
        str incident_id UK
        UUID dependency_id FK
        str severity
        str status
        str title
        str description
    }
```

### 9.2 Research Lineage

```text
FeatureSchema
    ↓
Dataset
    ↓
TrainingRun
    ↓
ModelArtifact
    ↓
PromotionDecision
```

### 9.3 Operational Lineage

```text
DependencyDefinition
    ↓
DependencyValidationRun
    ↓
DependencyValidationSnapshot
    ↓
DependencyIncident
```

**Operational failures are first-class artifacts.**

### 9.4 Versioning Strategy

**Content-Addressable Identity**:
- Primary identifier: cryptographic hash of content
- Secondary identifier: UUID for foreign keys
- Ensures immutability and deduplication

**Semantic Versioning**:
- Human-readable version strings (e.g., "rg16_stop_v3.2.1")
- Follows semver pattern: MAJOR.MINOR.PATCH
- Supports version comparison and ordering

**Lineage Versioning**:
- Parent-child relationships via `artifact_lineage`
- Supports derived artifacts, version chains, dependencies
- Enables impact analysis and rollback

---

## 10. Gate Framework

### 10.1 Gate 0: Dependency Health (MANDATORY)

**Purpose**: Operational dependency validation

**Definition**: Critical Dependencies Healthy

**Gate 0 consumes**: Dependency Health from AMIS Lab Operations

**Evaluation Criteria**:
- All CRITICAL dependencies must be HEALTHY
- No more than 1 HIGH dependency can be DEGRADED
- No CRITICAL or HIGH incidents in OPEN state
- Data reliability metrics within thresholds

**If Gate 0 fails**: Promotion Blocked

**No overrides.**

### 10.2 Gate 1: Research Validation

**Purpose**: Research validation and regime analysis

**Evaluation Criteria**:
- Minimum 4 independent walk-forward windows passed
- Expected R lift > 0 (positive edge)
- Regime stability score > 0.7
- **Mandatory Regime Governance**: India VIX, Volatility, Market Structure, HTF Alignment analysis included and significant
- Baseline comparison passed (beats current production)

### 10.3 Gate 2: Shadow Validation

**Purpose**: Shadow mode performance validation

**Evaluation Criteria**:
- Minimum 3 months of shadow data
- Minimum 1,000 shadow predictions
- Shadow Expected R lift > 0
- Calibration error < 10%
- No significant drift vs production
- Cross-instrument lift > 0

### 10.4 Gate 3: Human Approval

**Purpose**: Manual review and approval

**Evaluation Criteria**:
- Manual review completed
- Risk assessment documented
- Rollback plan documented
- Approval by authorized reviewer

---

## 11. API Specifications

### 11.1 REST API Patterns

All APIs follow SmartTrade conventions:
- **Base URL**: `/api/v1/{service}/{resource}`
- **Content Type**: `application/json`
- **Authentication**: Bearer JWT via smarttrade-common
- **Error Format**: SmartTradeError with error codes
- **Pagination**: Cursor-based with `limit` and `cursor` params

### 11.2 AMIS Core APIs

#### Registry Service

**POST /api/v1/registry/research-assets** (NEW)
```python
class ResearchAssetRequest(BaseModel):
    asset_type: Literal["MODEL", "FRAMEWORK", "DISCOVERY", "ENVELOPE"]
    name: str
    description: Optional[str] = None
    parent_artifact_ids: List[UUID] = []
    evidence_references: List[dict] = []
    semantic_version: str
    created_by: str
    metadata: Optional[dict] = None

class ResearchAssetResponse(BaseModel):
    id: UUID
    asset_type: str
    name: str
    status: str
    semantic_version: str
    created_at: datetime
```

### 11.3 AMIS Lab APIs

#### Operations Module

**GET /api/v1/operations/dependency-health**
```python
class DependencyHealthResponse(BaseModel):
    overall_health: DependencyHealth
    dependencies: List[DependencyHealthStatus]
    critical_failures: List[str]
    last_updated: datetime
```

### 11.4 Event Contracts

#### Research Events
- `research.asset_created` - Lab publishes, Core subscribes
- `research.validation_completed` - Lab publishes, Core subscribes

#### Promotion Events
- `promotion.artifact_submitted` - Core publishes, UI subscribes
- `promotion.gate_passed` - Core publishes, UI subscribes
- `promotion.decision_made` - Core publishes, Lab subscribes

#### Operational Events
- `operational.dependency_failed` - Lab publishes, Core subscribes
- `operational.incident_created` - Lab publishes, UI subscribes

---

## 12. Research Workflow

### 12.1 Create Candidate

1. Researcher creates candidate via SmartTrade UI AMIS Module
2. Define feature schema (or select existing)
3. Define label version (or select existing)
4. Configure candidate parameters
5. Submit candidate

### 12.2 Create Dataset

1. Select feature schema and label version
2. Configure data range and instruments
3. Run feature extraction
4. Generate dataset
5. Register dataset with AMIS Core

### 12.3 Run Validation

1. Configure validation job (walk-forward windows)
2. Submit validation job
3. Monitor validation progress
4. Review validation results
5. Generate scorecard

### 12.4 Run Regime Analysis (MANDATORY)

1. Configure regime analysis parameters
2. Run regime analysis (India VIX, Volatility, Market Structure, HTF Alignment)
3. Review regime-specific performance
4. Analyze regime stability

### 12.5 Generate Scorecard

1. Aggregate validation results
2. Include regime analysis (MANDATORY)
3. Compare with baseline
4. Generate scorecard

### 12.6 Promotion Review

1. Submit artifact for promotion
2. Attach scorecard
3. Gate 0 evaluation (operational health) - MANDATORY, no overrides
4. Gate 1 evaluation (research validation)
5. Gate 2 evaluation (shadow validation)
6. Gate 3 evaluation (human approval)
7. Promotion decision

---

## 13. Operational Workflow

### 13.1 Register Dependency

1. Define dependency (type, criticality, endpoints)
2. Configure health check parameters
3. Set SLA thresholds
4. Register dependency

### 13.2 Run Validation

1. Schedule validation run
2. Execute health checks
3. Validate SLA compliance
4. Generate validation snapshot
5. Store validation results

### 13.3 Create Incident

1. Detect dependency failure
2. Classify severity
3. Create incident record
4. Assign owner
5. Track resolution

### 13.4 Health Aggregation

1. Collect dependency health status
2. Aggregate operational metrics
3. Compute overall health score
4. Generate status report
5. Publish operational status

### 13.5 Promotion Blocking (Gate 0)

1. Pre-deployment Gate 0 check - MANDATORY
2. Validate dependency health
3. Check for open incidents
4. Block or allow deployment
5. No overrides

---

## 14. UI User Journeys

### 14.1 Researcher

**Goal**: Develop and validate AI trading models

**Journey**:
1. Log in to SmartTrade UI
2. Navigate to AMIS Dashboard
3. Navigate to Research Dashboard
4. Create new candidate
5. Define feature schema
6. Configure experiments
7. Run validation jobs
8. Review results (including mandatory regime analysis)
9. Generate research report
10. Submit for promotion

### 14.2 Reviewer

**Goal**: Review and approve promotion requests

**Journey**:
1. Log in to SmartTrade UI
2. Navigate to AMIS Dashboard
3. Navigate to Governance Dashboard
4. Review pending promotions
5. Evaluate gate results (including Gate 0)
6. Review research report
7. Assess risk
8. Approve or reject promotion
9. Monitor deployment

### 14.3 Operations Engineer

**Goal**: Monitor operational health and manage incidents

**Journey**:
1. Log in to SmartTrade UI
2. Navigate to AMIS Dashboard
3. Navigate to Operations Dashboard
4. Review dependency health
5. Investigate incidents
6. Resolve issues
7. Configure alerts
8. Generate operational reports

### 14.4 Administrator

**Goal**: Manage system configuration and access

**Journey**:
1. Log in to SmartTrade UI
2. Navigate to AMIS Dashboard
3. Navigate to Lineage Explorer
4. Review audit logs
5. Manage research assets
6. Configure gate definitions
7. Review deployment history

---

## 15. Implementation Roadmap

### 15.1 Priority 1: Core Foundation (Weeks 1-6)

**Objective**: Establish AMIS Core governance foundation and AMIS Lab research foundation

**Core Deliverables**:
- smarttrade-common shared contracts (ResearchStatus, DependencyHealth, ArtifactReference)
- AMIS Core Registry Service (FeatureSchema, LabelVersion, Dataset, ModelArtifact)
- AMIS Core ResearchAsset (first-class artifact)
- AMIS Core ExperimentArtifact (experiment tracking)
- AMIS Core Candidate Registration (candidate artifacts)
- AMIS Core ResearchProgram / ResearchTrack / ResearchMilestone
- AMIS Core Artifact Lineage (unified lineage authority)
- AMIS Core Gate Engine (Gate definitions, evaluators)
- Database schemas and migrations
- Basic REST APIs
- Integration with smarttrade-common

**Lab Deliverables**:
- AMIS Lab Candidate lifecycle management
- AMIS Lab Validation framework
- AMIS Lab Regime Analysis (VIX, Volatility, Market Structure, HTF Alignment)
- AMIS Lab Dependency Validation

**UI Deliverables**:
- SmartTrade UI AMIS Module — Programs Dashboard (R&D Control Tower)

**Success Criteria**:
- ResearchAsset registration and retrieval
- ExperimentArtifact registration and retrieval
- Candidate artifact registration
- ResearchProgram/Track/Milestone creation and tracking
- Feature schema registration and retrieval
- Dataset registration with lineage
- Model artifact registration
- Unified lineage query (research + operational)
- Gate evaluation framework
- Programs Dashboard displays RG16, RG18 status

### 15.2 Priority 2: Operations Service Layer (Weeks 7-10)

**Objective**: Complete operational capabilities, Gate 0 enforcement, incident management

**Lab Deliverables**:
- AMIS Lab Operations Module (full service layer)
- Gate 0 enforcement (dependency health → promotion blocking)
- Incident Management (lifecycle, escalation)
- Dependency Monitoring (health checks, status aggregation)
- Data Reliability Monitoring (quality, gaps, latency)

**Core Deliverables**:
- AMIS Core Audit Service (immutable audit log)
- Gate 0 integration (consume dependency health from Lab)

**Success Criteria**:
- Dependency health API
- Incident creation and tracking
- Gate 0 blocking logic (no overrides)
- Data reliability metrics
- Audit log for all governance actions

### 15.3 Priority 3: Promotion Workflows and Deployment Governance (Weeks 11-14)

**Objective**: Complete promotion, deployment, and UI workflows

**Core Deliverables**:
- AMIS Core Promotion Service (full promotion workflow)
- AMIS Core Deployment Governance (operating envelopes, deployments, rollback)
- AMIS Core Report Artifacts (ResearchReport, ValidationReport, RegimeReport)

**UI Deliverables**:
- AMIS Dashboard (Mission Control)
- Research Dashboard
- Validation Dashboard
- Governance Dashboard (promotion center)
- Operations Dashboard
- Lineage Explorer
- Deployment Monitor

**Success Criteria**:
- Full promotion workflow end-to-end
- Deployment orchestration with operating envelopes
- Report artifacts registered and retrievable
- All dashboards functional
- Lineage visualization
- Rollback capability

### 15.4 Priority 4: RG18 Live Validation (Weeks 15-18)

**Objective**: Complete RG18 live validation and promotion

**Deliverables**:
- RG18 Operating Envelope definition (Lab) and registration (Core)
- RG18 shadow validation automation
- RG18 promotion workflow
- TRACK-RG18-05 completion

**Success Criteria**:
- RG18 Operating Envelope registered in Core
- RG18 shadow validation complete
- RG18 promotion ready (pending dependency health + Gate 0)

**No new alpha research should begin until the operational platform is complete.**

---

## 16. Current Program Status

### 16.1 RG16

**Status**: `RESEARCH_EXPLANATION_DISCOVERED` (FROZEN)

**Outcome**:
- RG16 failed as an alpha model
- RG16 succeeded as:
  - Participation Framework
  - Regime Discovery Framework
  - VIX Discovery Vehicle

**ResearchAsset**: RG16 Participation Framework registered as RESEARCH_VALIDATED

### 16.2 RG18

**Status**: `SHADOW_PRODUCTION_CANDIDATE`

**Operating Envelope**:
```
RG18
+
India VIX < 15
+
RG16 Score >= 20
```

**Promotion remains blocked until**:
- Dependency health is restored
- TRACK-RG18-05 completes successfully

**ResearchAsset**: RG18 Operating Envelope registered as SHADOW_PRODUCTION_CANDIDATE

---

## 17. Risk Analysis

### 17.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Service communication failures | Medium | High | Circuit breakers, retry logic, health checks |
| Database performance issues | Medium | High | Connection pooling, query optimization, caching |
| Event bus failures | Low | High | Redis clustering, event replay, dead letter queues |
| Artifact store failures | Low | High | Redundant storage, backup, disaster recovery |
| Gate evaluation errors | Low | High | Comprehensive testing, manual override (except Gate 0) |

### 17.2 Operational Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Dependency failures blocking deployment | High | Medium | Dependency redundancy, graceful degradation |
| Gate 0 false positives | Medium | High | Conservative thresholds, manual review process |
| Incident response delays | Medium | High | Automated alerting, escalation procedures |
| Data quality issues | Medium | High | Data validation, quality monitoring |

### 17.3 Research Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Regime governance compliance failures | Medium | High | Automated enforcement, gate blocking |
| Overfitting to historical data | High | High | Walk-forward validation, out-of-sample testing |
| Regime instability | Medium | High | Regime analysis, multi-dimensional regime governance |
| Model decay in production | High | High | Shadow validation, drift monitoring |

### 17.4 Governance Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Decision traceability gaps | Low | High | Comprehensive audit logging, lineage tracking |
| ResearchAsset misuse | Low | Medium | Strict access controls, approval workflows |
| Gate 0 bypass attempts | Low | High | No override capability, strict enforcement |

---

## 18. Future Extensibility

### 18.1 Additional Research Groups

**RG19: Playbook Router**
- Integration of RG16 (opportunity) and RG18 (timing)
- Multi-model decision routing
- Portfolio-level optimization

**RG20: Decision Lineage**
- Complete decision tracking from research to execution
- Decision attribution and impact analysis
- Post-trade decision validation

### 18.2 Enhanced Regime Governance

**Future Regime Dimensions**:
- Market Breadth
- Sector Leadership
- Liquidity
- Macro Regimes

**All future candidates must include regime analysis.**

### 18.3 UI Enhancements

**AI-Powered Insights**
- Natural language query interface
- Automated anomaly detection
- Predictive analytics

**Collaboration Features**
- Shared workspaces
- Comment and annotation
- Change request workflows

### 18.4 Integration Extensions

**Additional Broker Integrations**
- Zerodha
- Interactive Brokers
- Additional brokers

**Additional Data Sources**
- Alternative data providers
- News and sentiment data
- Macroeconomic indicators

### 18.5 Advanced Analytics

**Causal Inference**
- Causal relationship discovery
- Counterfactual analysis
- Causal effect estimation

**Reinforcement Learning**
- Online learning from live data
- Adaptive strategy optimization
- Exploration-exploitation balancing

---

## Appendix A: Service Port Mapping

| Service | Internal Port | External Port | Database |
|---------|---------------|---------------|----------|
| AMIS Core | 8000 | 8015 | smarttrade_amis_core |
| AMIS Lab | 8000 | 8016 | smarttrade_amis_lab |
| SmartTrade UI | — | 5173 | — |

## Appendix B: Environment Variables

### AMIS Core
```
SERVICE_NAME=amis-core
DATABASE_URL=postgresql+asyncpg://user:pass@host:port/smarttrade_amis_core
REDIS_URL=redis://host:port
JWT_SECRET_KEY=${JWT_SECRET_KEY}
TOKEN_ENCRYPTION_KEY=${TOKEN_ENCRYPTION_KEY}
ARTIFACT_STORE_BACKEND=filesystem
ARTIFACT_STORE_PATH=/data/artifacts
ENVIRONMENT=local|dev|staging|prod
```

### AMIS Lab
```
SERVICE_NAME=amis-lab
DATABASE_URL=postgresql+asyncpg://user:pass@host:port/smarttrade_amis_lab
REDIS_URL=redis://host:port
JWT_SECRET_KEY=${JWT_SECRET_KEY}
TOKEN_ENCRYPTION_KEY=${TOKEN_ENCRYPTION_KEY}
AMIS_CORE_URL=http://amis-core:8015
MDS_URL=http://market-data-service:8004
ENVIRONMENT=local|dev|staging|prod
```

## Appendix C: Mandatory Regime Governance Checklist

All research artifacts must include:

- [ ] India VIX analysis
- [ ] Volatility regime analysis
- [ ] Market structure analysis
- [ ] HTF alignment analysis
- [ ] Regime-specific performance analysis
- [ ] Regime stability assessment
- [ ] Regime vs non-regime performance comparison
- [ ] Regime analysis in research report
- [ ] Regime analysis in promotion scorecard

**Enforcement Points**:
- Gate 1 evaluation (research validation)
- Promotion submission API
- Research report generation

## Appendix D: Gate 0 Compliance Checklist

Gate 0 must pass before all promotion decisions:

- [ ] All CRITICAL dependencies are HEALTHY
- [ ] No more than 1 HIGH dependency is DEGRADED
- [ ] No CRITICAL or HIGH incidents in OPEN state
- [ ] Data reliability metrics within thresholds
- [ ] Dependency health validated within last 24 hours

**No overrides.**

---

**Document Status**: Final Approved Architecture  
**Next Steps**: Implementation Priority 1 (Core Foundation)  
**Contact**: Architecture Team  
