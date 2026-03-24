# SmartTrade Monorepo Workflow Design
**Date:** 2026-03-24 | **Version:** v1 | **Status:** Design Approved

---

## Executive Summary

SmartTrade is a microservices-based trading platform with 5 core services (Authentication, Broker Adapter, Market Data, Mock, Frontend) that require coordinated cross-service design decisions. This design establishes a **Streaming Design Model** where:

1. **All cross-service decisions** (data contracts, event schemas, API changes, architecture) are designed and documented in `smarttrade-project/design/` before implementation
2. **Design docs are committed to git immediately** (no approval gate) with timestamp-versioned filenames
3. **Implementation happens in isolated service directories**, each in its own Claude session, referencing the design doc
4. **Service directories include symlinked references** to design docs for discoverability
5. **Versions track design evolution** naturally if clarifications are needed during implementation

**Goal:** Token-efficient workflow that keeps design and implementation concerns separate while maintaining full traceability.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Root Monorepo                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────┐        ┌──────────────────────┐  │
│  │ smarttrade-project/  │        │   Service Repos      │  │
│  │ (Design Authority)   │        │ (Implementation)     │  │
│  ├──────────────────────┤        ├──────────────────────┤  │
│  │ design/              │        │ broker-adapter-      │  │
│  │ ├─ cross-service/    │◄──────┤   service/           │  │
│  │ │  └─ 2026-03-24-    │        │ ├─ design/          │  │
│  │ │    monorepo-v1.md  │        │ │  └─ symlinks ───┐ │  │
│  │ │  └─ 2026-03-24-    │        │ └─ src/            │ │  │
│  │ │    pie-feature-v1. │        │ └─ tests/           │ │  │
│  │ │    md              │        │                     │ │  │
│  │ │  └─ [feature...]   │        │ market-data-        │  │
│  │ ├─ broker-adapter-   │        │   service/          │  │
│  │ │  service/          │        │ ├─ design/          │  │
│  │ │  (service-specific)│        │ └─ ...              │  │
│  │ └─ ...               │        │                     │  │
│  │ docs/                │        │ authentication-     │  │
│  │ └─ [guides...]       │        │   service/          │  │
│  │                      │        │ └─ ...              │  │
│  └──────────────────────┘        │                     │  │
│         ▲ Git Commit             │ mock-service/       │  │
│         │ (Design Versioning)    │ smarttrade-         │  │
│         │                        │   frontend/         │  │
│         └────────────────────────┴─ ...               │  │
│                                                        │  │
│  ┌──────────────────────────────────────────────────┐ │  │
│  │ smarttrade-tests/                                 │ │  │
│  │ (E2E & Cross-Service Validation)                 │ │  │
│  └──────────────────────────────────────────────────┘ │  │
│                                                        │  │
└────────────────────────────────────────────────────────┘
```

**Flow:**
1. Design docs created in smarttrade-project/design/cross-service/
2. Committed to git (version tracking via filename)
3. Service directories have symlinks to relevant design docs
4. Implementation sessions start in service directories, reference design via symlink
5. E2E tests validate cross-service contracts

---

## Directory & Repository Organization

### smarttrade-project (Design Authority)
```
smarttrade-project/
├── design/
│   ├── cross-service/              # Multi-service designs
│   │   ├── 2026-03-24-monorepo-workflow-v1.md
│   │   ├── 2026-03-24-pie-feature-v1.md
│   │   ├── 2026-03-24-pie-feature-v2.md (if updated)
│   │   └── [YYYY-MM-DD-feature-vN.md]
│   ├── broker-adapter-service/     # Service-specific designs
│   │   ├── Design.md
│   │   ├── RISK_ENGINE_DESIGN.md
│   │   └── ...
│   ├── market-data-service/
│   └── ...
├── docs/
│   ├── architecture-overview.md
│   ├── event-bus-patterns.md
│   └── ...
└── README.md
```

### Individual Service Directories (Implementation)
```
broker-adapter-service/
├── design/                         # Design references (symlinked)
│   ├── README.md                   # Index of designs affecting this service
│   ├── PIE_FEATURE → symlink to ../../smarttrade-project/design/cross-service/2026-03-24-pie-feature-v1.md
│   ├── ORDER_LIFECYCLE → symlink to existing design
│   └── ...
├── src/
│   ├── models/
│   ├── schemas/
│   ├── services/
│   ├── routes/
│   └── ...
├── tests/
├── README.md
└── ...

[Same structure for market-data-service/, authentication-service/, etc.]
```

### smarttrade-tests (E2E & Cross-Service)
```
smarttrade-tests/
├── postman/                        # API contract tests
├── playwright/                     # UI end-to-end tests
├── pytest/                         # Cross-service pytest suites
│   ├── e2e_order_placement_flow.py
│   ├── e2e_settlement_lifecycle.py
│   └── ...
└── README.md
```

---

## Design Doc Workflow & Versioning

### Naming Convention
```
YYYY-MM-DD-<feature-name>-v<N>.md

Examples:
- 2026-03-24-pie-feature-v1.md        (initial design)
- 2026-03-24-pie-feature-v2.md        (clarification during implementation)
- 2026-03-24-portfolio-sync-v1.md     (new feature)
```

### Workflow Steps

| Step | Location | Actor | Action |
|------|----------|-------|--------|
| 1 | smarttrade-project/ | Architect (Claude) | Write design doc following **Tiered Structure** |
| 2 | smarttrade-project/ | Architect | Commit design doc to git: `git add design/cross-service/YYYY-MM-DD-feature-v1.md && git commit` |
| 3 | Service directories | Architect | Create symlinks: `ln -s ../../smarttrade-project/design/cross-service/YYYY-MM-DD-feature-v1.md FEATURE_NAME` |
| 4 | Service directories | Architect | Commit symlinks: `git add design/FEATURE_NAME && git commit` |
| 5 | Individual service | Implementer | Start new Claude session in service directory, reference `./design/FEATURE_NAME` |
| 6 | Individual service | Implementer | Implement against design spec, write tests, create PR |
| 7 | smarttrade-tests/ | Implementer | Add E2E tests validating cross-service contracts |
| 8 | smarttrade-project/ | Architect (if needed) | If design needs clarification: update design → v2 → commit new version → symlinks auto-resolve |

### Design Doc Update Scenario
```
Initial: 2026-03-24-pie-feature-v1.md (design approved, implementation starts)
         ↓
         Implementation in BAS reveals missing error case
         ↓
Updated: 2026-03-24-pie-feature-v2.md (clarification committed)
         ↓
         Symlinks still point to v1.md (manual update if needed for clarity)
         ↓
         OR: Update symlinks to v2.md
         ↓
         Implementer continues with updated spec
```

Git history shows all versions → full audit trail.

---

## Design Doc Structure (Tiered)

Each cross-service design doc MUST include (in order):

### 1. Executive Summary (1-2 paragraphs)
- What feature/change is this?
- Why? (business/technical driver)
- Which services are affected?
- Success criteria

### 2. Architecture Diagram (ASCII or description)
- Show service boundaries
- Data flows, event flows, API calls
- Helps implementers understand the big picture at a glance

### 3. Service-Specific Sections (one per affected service)
For each affected service, include:
- **What this service implements**
- New models, schemas, routes, services
- Changes to existing behavior
- Database migrations needed
- Events published/consumed
- Dependencies on other services

Example structure:
```
### Broker Adapter Service
- New models: `Order`, `Trade`, `Settlement`
- New events: `order.placed`, `order.filled`, `trade.settled`
- New routes: `POST /api/v1/orders`, `GET /api/v1/orders/{order_id}`
- Database: New tables `orders`, `trades`
- Dependencies: Consumes `market.tick` from MDS
- Error handling: New error codes `ORD_001`, `ORD_002`
```

### 4. Detailed Specifications
- **Event Schemas** (JSON/YAML) — full schema definitions
- **API Contracts** (OpenAPI/YAML) — endpoint specifications
- **Data Models** — database schema, migrations
- **Error Codes** — new error codes with meanings
- **Validation Rules** — input validation, business rules

### 5. Dependencies & Sequencing
- What must be implemented first?
- What blocks what?
- Can services be implemented in parallel?
- Does smarttrade-common need changes?

Example:
```
Sequencing:
1. smarttrade-common: Add Order, Trade models → BLOCKS all services
2. BAS: Implement order creation → BLOCKS MDS price streaming
3. MDS: Implement price updates → BLOCKS BAS settlement
4. Frontend: Implement order UI → can start after BAS has API
```

### 6. Testing Strategy
- Unit test coverage targets (critical for trading logic — minimize mocks)
- Integration test scenarios (real DB, real events)
- E2E test flows (cross-service workflows in smarttrade-tests/)
- Data validation edge cases

---

## Session Workflow: Design → Implementation

### Design Session (in smarttrade-project/)
```bash
$ cd ~/Work/Smart-Trade/smarttrade-project
$ claude

→ Brainstorm & design feature (cross-service decisions)
→ Write design doc: design/cross-service/YYYY-MM-DD-feature-v1.md
→ Verify: All service sections complete, specs clear, versioning correct
→ Commit:
   git add design/cross-service/YYYY-MM-DD-feature-v1.md
   git commit -m "Design: Add feature spec (2026-03-24-feature-v1)"
→ Create symlinks in affected service directories
→ Exit session
```

### Implementation Sessions (one per service in service directory)
```bash
$ cd ~/Work/Smart-Trade/broker-adapter-service
$ claude

→ Read design doc from ./design/FEATURE_NAME
→ Understand: What does BAS need to implement?
→ Implement: Models → Schemas → Repos → Services → Routes
→ Write tests: Unit + integration against design spec
→ Verify cross-service contracts (events, APIs match design)
→ Create PR with service implementation
→ Exit session
```

**Repeat for each affected service (separate sessions, scoped focus).**

### E2E Testing Session (smarttrade-tests/)
```bash
$ cd ~/Work/Smart-Trade/smarttrade-tests
$ claude

→ Read cross-service design doc
→ Add E2E test scenarios validating contracts
→ Run full test suite to validate all services work together
→ Create PR with tests
```

---

## Design Doc Discoverability

### Service Directory Index

Each service has `design/README.md`:

```markdown
# Broker Adapter Service - Design References

## Active Features
- [Portfolio Insights Engine (PIE)](./PIE_FEATURE)
  - Doc: `smarttrade-project/design/cross-service/2026-03-24-pie-feature-v1.md`
  - Status: Implementing
  - Affected: BAS, MDS, Frontend

- [Order Lifecycle](./ORDER_LIFECYCLE)
  - Doc: [link]
  - Status: Complete
  - Affected: BAS, MDS, Auth

## Completed Features
- Position Management (completed 2026-03-15)
- Settlement Logic (completed 2026-03-10)

## To Reference a Design
1. Open the symlink: `cat ./PIE_FEATURE` (shows design doc content)
2. Or navigate to `smarttrade-project/design/cross-service/`
3. All symlinks auto-resolve if design doc is updated to vN
```

---

## Design Doc Updates (Versioning)

### Scenario: Design needs clarification during implementation

```
Situation:
  - 2026-03-24-pie-feature-v1.md committed
  - BAS implementing, discovers ambiguous error handling spec

Solution:
  1. Update design doc in smarttrade-project/
  2. Save as: 2026-03-24-pie-feature-v2.md
  3. Commit new version: git add && git commit -m "Design: PIE v2 - clarify error handling"
  4. Update symlinks (optional): ln -sf ../../smarttrade-project/design/cross-service/2026-03-24-pie-feature-v2.md PIE_FEATURE
  5. Continue implementation with v2

Result: Git history shows v1 → v2 evolution, full audit trail
```

---

## Tooling & Validation

### Setup (One-Time)
```bash
# In each service directory, create design folder
cd broker-adapter-service/
mkdir -p design
cd design

# Create symlinks (after design doc committed)
ln -s ../../smarttrade-project/design/cross-service/2026-03-24-pie-feature-v1.md PIE_FEATURE
ln -s ../../smarttrade-project/design/cross-service/2026-03-24-order-lifecycle-v1.md ORDER_LIFECYCLE

# Commit symlinks
git add PIE_FEATURE ORDER_LIFECYCLE
git commit -m "Design: Add PIE and ORDER_LIFECYCLE design references"
```

### Design Doc Validation (Optional Automation)
```bash
# JSON schema validation for event specs in design doc
jq -r '.event_schemas' design/cross-service/2026-03-24-pie-feature-v1.md | jq -s . > /tmp/events.json
ajv validate -s schema.json /tmp/events.json

# Cross-reference checks (verify all mentioned services exist)
grep -o "Service: [A-Za-z-]*" design/cross-service/*.md | sort | uniq

# Symlink validation (ensure all links are valid)
find . -type l -exec test -e {} \; -print
```

### Implementation Validation
- PR review confirms implementation matches design doc
- E2E tests in smarttrade-tests/ validate cross-service contracts
- All tests pass before merge

---

## Complete Workflow Checklist

### Design Phase
- [ ] Write design doc with all 6 sections (Exec Summary → Tiered Structure)
- [ ] Verify all service sections are complete and clear
- [ ] Commit design doc to smarttrade-project/design/cross-service/
- [ ] Create symlinks in each affected service directory
- [ ] Commit symlinks to git in each service
- [ ] Update service design/README.md with feature reference

### Implementation Phase (Per Service)
- [ ] Start new Claude session in service directory
- [ ] Read design doc from ./design/FEATURE_NAME
- [ ] Identify what this service must implement
- [ ] Implement: Models → Schemas → Repos → Services → Routes
- [ ] Write unit tests (against business logic, not mocks)
- [ ] Write integration tests (real DB, real events)
- [ ] Verify events/APIs match design spec exactly
- [ ] Create PR with clear reference to design doc
- [ ] PR review confirms implementation matches design

### E2E & Validation Phase
- [ ] Add E2E tests to smarttrade-tests/ validating cross-service contracts
- [ ] Run full test suite (all services + E2E)
- [ ] All tests pass
- [ ] Create PR in smarttrade-tests/
- [ ] Code review approved
- [ ] Ready for deployment

---

## Summary

| Aspect | Approach |
|--------|----------|
| **Design Location** | smarttrade-project/design/cross-service/ (centralized authority) |
| **Design Visibility** | Symlinked in service directories for discoverability |
| **Versioning** | Timestamp-versioned filenames (v1, v2, ...) with git history |
| **Session Structure** | Design session (root) → Implementation sessions (per service) → E2E session |
| **Token Efficiency** | Separate focused sessions per phase (no context bloat) |
| **Approval Gate** | Commit-based (no PR review), version-tracked |
| **Scalability** | Works for solo or team; git history is source of truth |
| **Discoverability** | Service directories have design README + symlinks |
| **Traceability** | Git commits + versioning = full audit trail |

**This workflow ensures:**
- ✅ Clean separation of design vs. implementation concerns
- ✅ Token-efficient (focused sessions)
- ✅ Fully traceable (git versioning)
- ✅ Discoverable (symlinks + README index)
- ✅ Fast (no approval gates, streaming design)
- ✅ Flexible (versioning allows clarifications)

---

**Design Status:** ✅ **APPROVED**

**Next Step:** Implementation planning via writing-plans skill
