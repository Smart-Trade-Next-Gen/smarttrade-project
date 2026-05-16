# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**smarttrade-project** is the centralized repository for SmartTrade design documents, architecture guides, and cross-service planning. It is a separate git repo from the main monorepo (`smarttrade-mono`), ensuring design decisions are versioned and discoverable independently.

## Purpose

This repo holds:
- **Architecture documents** — System-wide design (current: `FINAL_TARGET_ARCHITECTURE_v4.0.md`)
- **Cross-service design documents** — Features spanning 2+ services (in `design/cross-service/`)
- **Active ADRs / shared-library designs** — e.g., the instrument-master trio.
- **Roadmap** — cross-service direction (`ROADMAP.md`).

Service-specific documentation lives in each service's own repository (see README for links).

## Directory Structure

```
smarttrade-project/
  FINAL_TARGET_ARCHITECTURE_v4.0.md     ← System architecture (source of truth)
  ARCHITECTURE_DECISION_RECORD_instrument_master.md
  DESIGN_smarttrade_common_instrument_master.md
  IMPLEMENTATION_SUMMARY_instrument_master.md
  design/
    cross-service/                      ← Multi-service feature designs
  README.md                             ← Index / quick links
  ROADMAP.md                            ← Cross-service roadmap
  CLAUDE.md                             ← This file
```

Service-specific documentation is maintained in each service repository
(see README.md for the full mapping).

## Design Document Workflow

SmartTrade uses a **Streaming Design Model** where design and implementation happen in separate phases:

### Phase 1: Design (in smarttrade-project)

1. Create a new design doc in `design/cross-service/` following the naming convention:
   ```
   YYYY-MM-DD-<feature-name>-v1.md
   Example: 2026-03-28-pie-feature-v1.md
   ```

2. Structure the doc with these sections:
   - **Executive Summary** — What, why, which services, success criteria
   - **Architecture Diagram** — Service flows, events, APIs
   - **Service-Specific Sections** — One per affected service (what to implement)
   - **Detailed Specifications** — Event schemas, API contracts, data models, error codes
   - **Dependencies & Sequencing** — What blocks what, implementation order
   - **Testing Strategy** — Unit, integration, E2E test coverage

3. Commit the design doc:
   ```bash
   git add design/cross-service/YYYY-MM-DD-feature-v1.md
   git commit -m "Design: <feature-name> (v1)"
   ```

4. Create symlinks in affected service directories:
   ```bash
   # In smarttrade-mono/broker-adapter-service/design/
   ln -s ../../smarttrade-project/design/cross-service/YYYY-MM-DD-feature-v1.md FEATURE_NAME
   git add FEATURE_NAME
   git commit -m "Design: Add FEATURE_NAME reference"
   ```

### Phase 2: Implementation (in individual service repos)

Implementations happen in the main monorepo (`smarttrade-mono`), with each service reading its design doc via symlink.

### Phase 3: Design Iteration

If implementation reveals gaps, create a new design version:

```bash
1. Update design doc in smarttrade-project/
2. Save as: YYYY-MM-DD-feature-v2.md
3. Commit: git add && git commit
4. Update symlinks in service repos (optional)
5. Continue implementation with v2

# Full version history is preserved in git
```

## Key Rules

### Design Doc Placement (MANDATORY)

**In smarttrade-project**:
- ✓ Cross-service designs → `design/cross-service/` (features spanning 2+ services)
- ✓ Architecture documents → Repository root (current: `FINAL_TARGET_ARCHITECTURE_v4.0.md`)

**In Service Repositories** (NOT in smarttrade-project):
- ✓ Service-specific implementation docs → Service's own `docs/` directory
- ✓ LLDs, API specs, design decisions → Service's own `docs/` directory
- ✓ Broker API references (e.g., Fyers) → Service's own `docs/` directory
- ✓ Test strategies (E2E) → `smarttrade-tests/docs/`

**What NOT to do**:
- ✗ Never place service-specific designs in `smarttrade-project/design/<service-name>/`
- ✗ Never duplicate docs across repos (single source of truth = service's own docs/)
- ✗ Never put implementation docs in smarttrade-project (they belong in service repos)

### API and Data Contracts

Design docs **must** include:
- REST endpoint specs (method, path, request/response schemas)
- WebSocket message schemas (if applicable)
- Event schemas (Pydantic models with field descriptions)
- Error codes with descriptions
- Database schema changes (if applicable)

Use concrete examples, not abstract descriptions.

### Financial Code

If the design touches trading, positions, or settlement:
- Reference the Financial Correctness guide (`docs/FINANCIAL_CORRECTNESS.md`)
- Include validation rules (e.g., position limits, daily loss caps)
- Document audit trail requirements
- Specify rounding rules and precision (e.g., decimal places for prices)

### Naming Conventions

- **Feature names**: lowercase with hyphens (e.g., `pie-feature`, `portfolio-sync`)
- **Services**: match the main repo (`broker-adapter-service`, `market-data-service`, etc.)
- **Error codes**: 3-letter prefix + number (e.g., `ORD_001`, `MDS_002`)

## Commands

### Design Docs

```bash
# Create new design doc
vi design/cross-service/YYYY-MM-DD-<feature>-v1.md

# Verify doc compiles and links are valid
# (Use VS Code to check markdown rendering)

# Commit design doc
git add design/cross-service/...
git commit -m "Design: <feature-name> (v1)"

# Create symlink in service directory
cd <monorepo>/broker-adapter-service/design/
ln -s ../../smarttrade-project/design/cross-service/YYYY-MM-DD-feature-v1.md FEATURE
git add FEATURE && git commit -m "Design: Add FEATURE reference"
```

### Service Documentation

All service-specific documentation is maintained in the service's own repository:

- **Broker Adapter Service**: See `broker-adapter-service/docs/` — stateless
  execution kernel; broker adapter system, WebSocket account-event interface,
  event-schema contracts, Fyers API reference.
- **Market Data Service**: See `market-data-service/docs/` — target HLD,
  WebSocket protocol, reconnect/replay architecture, auth strategy.
- **Paper Broker Service**: See `paper-broker-service/docs/` — HLD (v4),
  quote freshness model, MDS alignment.
- **Journal / Portfolio / Strategy / Notification Services**: see each
  service's own `CLAUDE.md` and (where present) `docs/`.
- **Shared library**: See `smarttrade-common/docs/`.
- **E2E Testing**: See `smarttrade-tests/docs/E2E_TESTING_STRATEGY.md`.

These docs are the source of truth for implementation. Components like the
Order State Machine, Execution Orchestrator, Outbox Pattern, and in-BAS Risk
Engine have been removed during the stateless refactor — do not implement
against historical docs found in git history.

## Testing Strategy

- **Unit tests** → Service repo (same service)
- **Integration tests** → Service repo (same service)
- **E2E tests** → `smarttrade-tests/` repo (cross-service workflows)
- **E2E testing strategy & patterns** → `smarttrade-tests/docs/E2E_TESTING_STRATEGY.md`

Never put E2E tests in individual service repos or smarttrade-project.

## Collaboration

When working on a design:

1. **Draft locally** — Create the doc in your branch
2. **Solicit feedback** — Share the doc URL with team before committing
3. **Commit when ready** — Design doc is finalized
4. **Implement in parallel** — Services can start implementation once design is committed
5. **Iterate if needed** — Create v2 if clarifications emerge during implementation

## FAQ

**Q: Where do service-specific docs go?**
A: In the service's own repository (e.g., `broker-adapter-service/docs/`). Not in smarttrade-project. This keeps docs close to code.

**Q: Where do cross-service designs go?**
A: In `smarttrade-project/design/cross-service/`. Use naming: `YYYY-MM-DD-feature-name-v1.md`.

**Q: When should I create a new version (v2) of a design?**
A: If implementation reveals gaps or changes, create v2. Git preserves the full evolution.

**Q: Who approves designs?**
A: No approval gate. Commit when confident. Fast feedback beats slow approval.

**Q: Can services reference docs in smarttrade-project?**
A: Yes, cross-service designs (via git submodule symlinks if needed). But service-specific docs must live in the service repo.

**Q: I found outdated docs in git history. Should I use them?**
A: No. Always use current docs in each service's active `docs/` directory.
Historical phase plans, completion reports, and LLDs for removed components
have been deleted from the working tree; git history preserves them if you
need to look them up.
