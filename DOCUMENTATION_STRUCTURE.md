# SmartTrade Documentation Structure

**Last Updated**: 2026-05-16  
**Status**: Current - Reflects documentation reorganization to align with implementation

---

## Overview

SmartTrade uses a **tiered documentation structure** with clear separation between current operational guidance and historical architectural documents. This structure ensures that developers always have access to accurate, current information while preserving historical context.

---

## 📌 Primary Sources of Truth (Current & Accurate)

### Service-Level Documentation
Each service maintains a **CLAUDE.md** file that serves as the primary source of truth for architecture, implementation guidance, and development practices.

| Service | Primary Document | Status | Focus |
|---------|------------------|--------|-------|
| **Broker Adapter Service (BAS)** | [`broker-adapter-service/CLAUDE.md`](../broker-adapter-service/CLAUDE.md) | ✅ Accurate | Stateless execution kernel, broker communication, event publishing |
| **Market Data Service (MDS)** | [`market-data-service/claude.md`](../market-data-service/claude.md) | ✅ Accurate | Quote ingestion, instrument master, WebSocket protocol |
| **Paper Broker Service (PBS)** | [`paper-broker-service/claude.md`](../paper-broker-service/claude.md) | ✅ Accurate | Paper trading execution, market data consumption |
| **Other Services** | `*/CLAUDE.md` or `*/claude.md` | ✅ Accurate | Service-specific guidance |

### System-Level Architecture
- **Overall Architecture**: [`FINAL_TARGET_ARCHITECTURE_v4.0.md`](FINAL_TARGET_ARCHITECTURE_v4.0.md) - ✅ **Updated 2026-05-16** to reflect current stateless implementation
- **Project Guidelines**: [`CLAUDE.md`](CLAUDE.md) - ✅ Accurate - Cross-service patterns and conventions

---

## 🗑️ Removed Documents

The following detailed High-Level Design (HLD) documents have been **removed** to eliminate confusion. They described older architectural iterations and do not match the current implementation.

| Document | Removal Date | Reason |
|----------|-------------|--------|
| **BAS_TARGET_HLD.md** | 2026-05-16 | Described old stateful architecture; BAS is now stateless |
| **MDS_TARGET_HLD.md** | 2026-05-16 | Used outdated event naming (`market_data.quote` vs `market.quote`) |
| **PBS_HLD_v1.md** | 2026-05-16 | Superseded by CLAUDE.md; historical context preserved in git history |

**Historical Access**: These documents can still be accessed through git history if needed for historical reference, but they are not part of the current documentation tree to avoid confusion.

---

## 📚 Active Specialized Documentation

Some specialized documentation remains active and accurate:

### Broker Adapter Service (BAS)
- **WebSocket API**: [`broker-adapter-service/docs/WEBSOCKET_API.md`](../broker-adapter-service/docs/WEBSOCKET_API.md) - Current protocol specification
- **Event Schemas**: [`broker-adapter-service/docs/EVENT_SCHEMA_CONTRACTS_LLD.md`](../broker-adapter-service/docs/EVENT_SCHEMA_CONTRACTS_LLD.md) - Event contracts
- **Adapter System**: [`broker-adapter-service/docs/ADAPTER_SYSTEM.md`](../broker-adapter-service/docs/ADAPTER_SYSTEM.md) - Broker plugin architecture
- **Fyers API Reference**: [`broker-adapter-service/docs/fyers-api-reference/`](../broker-adapter-service/docs/fyers-api-reference/) - Broker API documentation

### Market Data Service (MDS)
- **WebSocket Protocol**: [`market-data-service/docs/MDS_WEBSOCKET_PROTOCOL.md`](../market-data-service/docs/MDS_WEBSOCKET_PROTOCOL.md) - Current protocol specification
- **Authentication Strategy**: [`market-data-service/docs/MDS_AUTHENTICATION_STRATEGY.md`](../market-data-service/docs/MDS_AUTHENTICATION_STRATEGY.md) - Auth mechanisms

### Paper Broker Service (PBS)
- **Quote Freshness**: [`paper-broker-service/docs/quote_freshness.md`](../paper-broker-service/docs/quote_freshness.md) - Quote consumption model
- **MDS Alignment**: [`paper-broker-service/docs/PBS_MDS_ALIGNMENT_2026-05-06.md`](../paper-broker-service/docs/PBS_MDS_ALIGNMENT_2026-05-06.md) - Integration patterns

### Cross-Service
- **E2E Testing**: [`smarttrade-tests/docs/E2E_TESTING_STRATEGY.md`](../smarttrade-tests/docs/E2E_TESTING_STRATEGY.md) - Testing approach
- **Design Documents**: [`smarttrade-project/design/cross-service/`](./design/cross-service/) - Cross-service feature designs

---

## 🔄 Documentation Maintenance Philosophy

### Principles
1. **Single Source of Truth**: CLAUDE.md files are the authoritative source for each service
2. **Accuracy Over Comprehensiveness**: Prefer accurate, concise documentation over outdated detailed docs
3. **Historical Preservation**: Deprecated documents are preserved but clearly marked
4. **Living Documentation**: Primary documents are updated to reflect implementation changes

### Update Workflow
1. **Implementation Changes**: Update CLAUDE.md immediately when architecture changes
2. **Documentation Drift**: Periodically audit and align docs with implementation
3. **Deprecation Process**: When detailed docs become outdated, deprecate rather than attempt costly updates

### Benefits
- **Reduced Maintenance**: Focus on maintaining a few accurate documents vs. many outdated ones
- **Developer Confidence**: Clear guidance on which documents to trust
- **Historical Context**: Preserved architecture evolution without confusion
- **Faster Onboarding**: New developers start with accurate, current information

---

## 📖 Quick Reference for Developers

### When Working on BAS:
1. **Start**: [`broker-adapter-service/CLAUDE.md`](../broker-adapter-service/CLAUDE.md)
2. **WebSocket Details**: [`broker-adapter-service/docs/WEBSOCKET_API.md`](../broker-adapter-service/docs/WEBSOCKET_API.md)
3. **Event Contracts**: [`broker-adapter-service/docs/EVENT_SCHEMA_CONTRACTS_LLD.md`](../broker-adapter-service/docs/EVENT_SCHEMA_CONTRACTS_LLD.md)

### When Working on MDS:
1. **Start**: [`market-data-service/claude.md`](../market-data-service/claude.md)
2. **WebSocket Protocol**: [`market-data-service/docs/MDS_WEBSOCKET_PROTOCOL.md`](../market-data-service/docs/MDS_WEBSOCKET_PROTOCOL.md)

### When Working on PBS:
1. **Start**: [`paper-broker-service/claude.md`](../paper-broker-service/claude.md)
2. **Quote Model**: [`paper-broker-service/docs/quote_freshness.md`](../paper-broker-service/docs/quote_freshness.md)

### Understanding Overall Architecture:
1. **Start**: [`FINAL_TARGET_ARCHITECTURE_v4.0.md`](FINAL_TARGET_ARCHITECTURE_v4.0.md)
2. **Cross-Service Patterns**: [`CLAUDE.md`](CLAUDE.md)

---

## 🚀 Future Improvements

### Potential Enhancements
1. **Documentation Validation**: Automated checks to ensure CLAUDE.md files remain accurate
2. **Reference Linking**: Better cross-referencing between related documents
3. **Visualization**: Architecture diagrams that stay in sync with implementation
4. **Onboarding Guide**: Simplified getting started documentation

### Maintenance Schedule
- **Quarterly Audit**: Review primary documents for implementation alignment
- **Major Changes**: Update CLAUDE.md immediately during architectural refactors
- **Deprecation Review**: Annual review of deprecated docs for archival or removal

---

## 📞 Questions or Issues?

If you find documentation that seems inaccurate or outdated:
1. Check the document's deprecation status
2. Verify against the service CLAUDE.md (primary source)
3. Check the overall architecture document
4. If discrepancies persist, raise an issue or update the primary source

**Remember**: CLAUDE.md files are always the primary source of truth for service-level architecture and implementation guidance.