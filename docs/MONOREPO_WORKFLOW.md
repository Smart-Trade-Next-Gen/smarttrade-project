# SmartTrade Monorepo Workflow Guide

This guide explains the **Streaming Design Model** for making cross-service decisions in SmartTrade.

## Quick Summary

1. **Design Phase** (in smarttrade-project/) — Architect writes cross-service design doc
2. **Commit** — Design doc is committed to `design/cross-service/YYYY-MM-DD-feature-v1.md`
3. **Symlink** — Symlinks created in service directories pointing to design doc
4. **Implement** (in service directory) — Implementer reads design doc and implements per service
5. **Validate** — E2E tests validate cross-service contracts

**Why this approach?**
- Token-efficient: Separate focused sessions per phase
- Traceable: Git versioning tracks all design decisions
- Discoverable: Service directories have symlinks to relevant designs
- Fast: No approval gates, streaming workflow

---

## Detailed Workflow

### Phase 1: Design (in smarttrade-project/)

```bash
$ cd ~/Work/Smart-Trade/smarttrade-project
$ claude

# Use superpowers:brainstorming skill to design feature
→ Brainstorm cross-service requirements
→ Write design doc: design/cross-service/YYYY-MM-DD-feature-v1.md
→ Verify all 6 sections complete (see design-template.md)
→ Commit: git add design/cross-service/YYYY-MM-DD-feature-v1.md && git commit -m "Design: ..."
```

**Design Doc Structure:**
See `design-template.md` for complete template. Must include:
1. Executive Summary
2. Architecture Diagram
3. Service-Specific Sections (one per affected service)
4. Detailed Specifications (event schemas, APIs, data models)
5. Dependencies & Sequencing
6. Testing Strategy

---

### Phase 2: Symlink Setup

After design doc is committed, create symlinks in affected service directories:

```bash
# Example: BAS needs to implement PIE feature
cd ~/Work/Smart-Trade/broker-adapter-service/design
ln -s ../../smarttrade-project/design/cross-service/2026-03-24-pie-feature-v1.md PIE_FEATURE
git add PIE_FEATURE
git commit -m "Design: Add PIE_FEATURE reference"

# Do the same for other affected services (MDS, Frontend, etc.)
```

After symlinks committed, update service `design/README.md` with feature link:

```markdown
## Active Features
- [Portfolio Insights Engine (PIE)](./PIE_FEATURE)
  - Doc: `smarttrade-project/design/cross-service/2026-03-24-pie-feature-v1.md`
  - Status: Implementing
  - Affected: BAS, MDS, Frontend
```

---

### Phase 3: Implementation (one per service)

For each affected service:

```bash
$ cd ~/Work/Smart-Trade/broker-adapter-service
$ claude

# Start fresh session in service directory
→ Read design doc: cat ./design/PIE_FEATURE
→ Understand: What does BAS need to implement?
→ Implement according to design:
  - Models (from design specs)
  - Schemas (request/response)
  - Repositories (data access)
  - Services (business logic)
  - Routes (HTTP endpoints)
→ Write tests (unit + integration)
→ Verify events/APIs match design exactly
→ Create PR with reference to design doc
```

**Repeat for each affected service** (separate Claude sessions).

---

### Phase 4: E2E Testing & Validation

```bash
$ cd ~/Work/Smart-Trade/smarttrade-tests
$ claude

→ Read cross-service design doc
→ Add E2E test scenarios validating contracts
→ Run full test suite (all services + E2E)
→ All tests pass
→ Create PR with tests
```

---

## Design Doc Updates (Versioning)

If clarifications are needed during implementation:

```
1. Update design doc in smarttrade-project/
2. Save as: 2026-03-24-pie-feature-v2.md
3. Commit new version: git add && git commit -m "Design: PIE v2 - clarify error handling"
4. Update symlinks (optional): ln -sf ../../smarttrade-project/design/cross-service/2026-03-24-pie-feature-v2.md PIE_FEATURE
5. Continue implementation with v2
```

Git history shows v1 → v2 evolution (full audit trail).

---

## Design Doc Naming & Locations

**Naming Convention:**
```
YYYY-MM-DD-<feature-name>-v<N>.md

Examples:
- 2026-03-24-pie-feature-v1.md        (initial design)
- 2026-03-24-pie-feature-v2.md        (clarification during implementation)
- 2026-03-24-portfolio-sync-v1.md     (new feature)
```

**Location:**
```
smarttrade-project/design/cross-service/YYYY-MM-DD-feature-vN.md
```

Service-specific designs go in:
```
smarttrade-project/design/<service-name>/
```

---

## Token Efficiency Benefits

| Phase | Session | Focus | Tokens |
|-------|---------|-------|--------|
| Design | smarttrade-project/ | Cross-service decisions, event schemas, API contracts | Full design context |
| Implement | broker-adapter-service/ | Only BAS logic, models, services, routes | Service-scoped context |
| Implement | market-data-service/ | Only MDS logic | Service-scoped context |
| E2E Test | smarttrade-tests/ | Contract validation | E2E test context |

**Result:** No token bloat from trying to implement all services in one session.

---

## Checklist for Your First Design

- [ ] Write design doc with all 6 sections
- [ ] Commit design doc to smarttrade-project/design/cross-service/
- [ ] Create symlinks in each affected service directory
- [ ] Commit symlinks to git
- [ ] Update service design/README.md with feature link
- [ ] Start implementation session in first service directory
- [ ] Read design doc from ./design/FEATURE_NAME
- [ ] Implement & test per design spec
- [ ] Create PR with clear reference to design doc
- [ ] Repeat for other affected services
- [ ] Add E2E tests to smarttrade-tests/
- [ ] All tests pass, ready for deployment
