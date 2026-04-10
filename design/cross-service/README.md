# Cross-Service Design Documents

This directory contains approved design documents for cross-service features in SmartTrade.

---

## 📌 Service Documentation Reorganization (2026-04-10)

Several cross-service documents have been moved to their respective service repositories for better maintainability:

| Document | Original Location | New Location | Status |
|----------|------------------|--------------|--------|
| Paper Broker Service HLD | `2026-04-04-mock-service-hld-v1.md` | `paper-broker-service/docs/Paper_Broker_Service_HLD_v1.md` | ✅ Moved (2026-04-10) |
| E2E Testing Strategy | `2026-04-04-e2e-testing-strategy-v1.md` | `smarttrade-tests/docs/E2E_TESTING_STRATEGY.md` | ✅ Moved (2026-04-10) |
| Fyers API Reference | `../fyers-api-reference/` | `broker-adapter-service/docs/fyers-api-reference/` | ✅ Moved (2026-04-10) |

**Rationale**: Service-specific and test-specific documentation is more maintainable when co-located with the code. Cross-service designs remain here.

---

## Current Cross-Service Designs

*Currently none. If a feature spans 2+ services and requires a design doc, add it here.*

See [`../README.md`](../README.md) for links to service-specific documentation now maintained in each service repository.
