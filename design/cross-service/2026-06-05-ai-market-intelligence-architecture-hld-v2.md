> **DEPRECATED**: This document describes the legacy `ai-service` (port 8014), which has been deprecated. Its functionality has been migrated to `amis-core-service` (port 8000), `amis-lab-service` (port 8016), and `signal-processor-service` (port 8012). See `ai-service-deprecation-plan.md` for details.

# High Level Design (HLD): AI Market Intelligence Service Architecture Refactoring

**Document Version**: 2.0  
**Date**: 2026-06-05  
**Status**: Design Draft (Revised)  
**Author**: SmartTrade Architecture Team  
**Scope**: Cross-service architecture refactoring (Signal Processor → AI Market Intelligence, Strategy Service refactoring)  
**Previous Version**: v1.0

---

## Executive Summary

This HLD defines the architecture for removing the Signal Processor Service and introducing the AI Market Intelligence Service as the central intelligence and learning engine for SmartTrade. The refactoring maintains the existing execution plane vs async plane architecture, preserves BAS as the sole execution authority, and ensures AI components remain advisory and decision-support only.

**Key Changes**:
- **Remove**: Signal Processor Service (SPS) - pattern detection and analysis capabilities move to AI Market Intelligence
- **Add**: AI Market Intelligence Service (AMIS) - unified intelligence platform for market learning, setup intelligence, and probability generation
- **Refactor**: Strategy Service - evolves into Decision Engine with reduced scope, consuming AI intelligence to produce trade recommendations
- **Add**: Risk Engine (NEW) - Centralized risk validation and policy enforcement
- **Add**: Position Manager (NEW) - Position sizing, pyramiding, and trailing stop management

**Success Criteria**:
1. Zero disruption to BAS execution authority and trading operations
2. Seamless migration of SPS capabilities to AMIS with enhanced learning features
3. Strategy Service successfully refactored as Decision Engine with AI intelligence integration
4. All services maintain stateless architecture and event-driven communication patterns
5. Phase 1 (statistical learning) delivers production-ready setup intelligence without ML dependencies
6. Architecture supports future AI features: AI Trade Coach, AI Day Planner, AI Smart Watchlist, Runtime Position Advisor

---

## 1. Updated Service Landscape

### 1.1 Current Architecture (Before Refactoring)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SmartTrade Microservices                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Auth Svc   │  │     MDS      │  │     PBS      │          │
│  │    (8001)    │  │    (8004)    │  │    (8002)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │      BAS     │  │     SPS      │  │  Strategy    │          │
│  │    (8005)    │  │    (8013)    │  │    (8006)    │          │
│  │ EXECUTION    │  │  ANALYSIS    │  │  DECISIONS   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Journal    │  │  Portfolio   │  │ Notification │          │
│  │    (8007)    │  │    (8008)    │  │    (8011)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Target Architecture (After Refactoring)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SmartTrade Microservices                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Auth Svc   │  │     MDS      │  │     PBS      │          │
│  │    (8001)    │  │    (8004)    │  │    (8002)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │      BAS     │  │     AMIS     │  │  Strategy    │          │
│  │    (8005)    │  │    (8014)    │  │    (8006)    │          │
│  │ EXECUTION    │  │ INTELLIGENCE │  │  DECISION    │          │
│  └──────────────┘  └──────────────┘  │   ENGINE     │          │
│                     ┌──────────────┐  └──────────────┘          │
│                     │ PostgreSQL    │                                │
│                     │  + pgvector   │                                │
│                     └──────────────┘                                │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Journal    │  │  Portfolio   │  │ Notification │          │
│  │    (8007)    │  │    (8008)    │  │    (8011)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐                          │
│  │   Risk       │  │   Position   │                          │
│  │   Engine     │  │   Manager    │                          │
│  │    (8015)    │  │    (8016)    │                          │
│  └──────────────┘  └──────────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Service Port Mapping

| Service | Current Port | Target Port | Change |
|---------|--------------|-------------|--------|
| Authentication Service | 8001 | 8001 | No change |
| Paper Broker Service | 8002 | 8002 | No change |
| Market Data Service | 8004 | 8004 | No change |
| Broker Adapter Service | 8005 | 8005 | No change |
| Strategy Service | 8006 | 8006 | No change (refactored) |
| Journal Service | 8007 | 8007 | No change |
| Portfolio Service | 8008 | 8008 | No change |
| Notification Service | 8011 | 8011 | No change |
| Signal Processor Service | 8013 | **REMOVED** | Decommissioned |
| AI Market Intelligence Service | **N/A** | **8014** | **NEW** |
| Risk Engine | **N/A** | **8015** | **NEW** |
| Position Manager | **N/A** | **8016** | **NEW** |

---


## 2. Service Responsibilities

### 2.1 AI Market Intelligence Service (AMIS) - NEW

**Port**: 8014  
**Database**: `smarttrade_ai_market_intelligence` (PostgreSQL + pgvector)  
**Plane**: ASYNC (intelligence generation, learning, and analysis)

#### Core Responsibilities

**MUST OWN**:
1. **Candlestick Pattern Detection**
   - Real-time detection of candlestick patterns (Bullish/Bearish Engulfing, Hammer, Shooting Star, Doji, Inside Bar, Outside Bar, Morning Star, Evening Star)
   - Multi-timeframe pattern analysis (1m, 3m, 5m, 15m, 30m, 1h, 4h, 1D, 1W, 1M)
   - Pattern confidence scoring and classification

2. **Price Action Analysis**
   - Breakout/breakdown detection
   - Retest identification and validation
   - Pullback analysis
   - Trend continuation detection
   - Range expansion/compression
   - Liquidity sweep detection
   - Support/resistance rejection

3. **Market Structure Analysis**
   - Higher High/Higher Low detection
   - Lower High/Lower Low detection
   - Trend change identification
   - Structure break (BOS/CHOCH) detection
   - Market shift recognition
   - Accumulation/distribution phases

4. **Setup Intelligence Engine** (NEW - First-Class Domain)
   - Detect multi-factor setups (pattern + structure + volume + trend + context)
   - Maintain setup catalog with versioning
   - Track setup performance (success rate, failure rate, future returns)
   - Support setup lifecycle management (emerged, confirmed, failed, invalidated)
   - Generate setup quality scores (0-100)
   - Example: Bullish Continuation Setup = Bullish Engulfing + Higher Low + Volume Expansion + Breakout Confirmation

5. **Outcome Evaluation Engine** (NEW - Dedicated)
   - Evaluate outcomes after predefined horizons (1h, 4h, 1d, 1w, 1m)
   - Measure success/failure classification
   - Measure future returns
   - Calculate drawdown and reward achieved
   - Update setup statistics
   - Update pattern statistics
   - Feed probability engine with outcome data

6. **Market Regime Engine** (NEW)
   - Trending market detection (uptrend, downtrend)
   - Range-bound market detection
   - Breakout day classification
   - Gap-up day classification
   - Gap-down day classification
   - High volatility classification
   - Low volatility classification
   - Event-driven market classification
   - Sector regime classification

7. **Watchlist Intelligence Engine** (NEW)
   - Opportunity ranking across instruments
   - Instrument scoring based on setup quality
   - Watchlist generation and prioritization
   - Unusual activity detection
   - Setup discovery across watchlist
   - Pre-market watchlist generation

8. **Setup Quality Scoring** (NEW - Core Domain Concept)
   - Pattern quality scoring
   - Structure quality scoring
   - Volume quality scoring
   - Trend quality scoring
   - Context quality scoring
   - Composite setup score generation (0-100)
   - Confidence level calculation
   - Ranking score generation

9. **Similarity Search**
   - Feature vector generation for market situations
   - Vector types: Pattern Embeddings, Structure Embeddings, Setup Embeddings, Market Regime Embeddings
   - Nearest-neighbor search using pgvector (Phase 1)
   - Pattern clustering and grouping
   - Historical analog discovery
   - Case-based reasoning for similar market situations and setups

10. **Probability Engine**
    - Generate statistical probabilities from historical outcomes
    - Trend continuation probability
    - Breakout success probability
    - Reversal probability
    - Expected move probability
    - Risk-reward expectancy
    - Confidence score calculation

11. **Feature Generation**
    - Generate ML-ready features for training
    - ATR, Volume Ratio, Relative Strength
    - Gap Percentage, Distance from VWAP/EMA
    - Body Size Ratio, Wick Ratio
    - Momentum Score, Trend Strength

12. **Historical Dataset Management**
    - Maintain training datasets for ML models
    - Versioned dataset storage
    - Data quality validation
    - Feature store management

13. **Pattern Performance Tracking**
    - Track win rates by pattern type
    - Track failure rates by market condition
    - Track future returns by pattern
    - Track holding period performance
    - Track time-of-day performance

14. **Market Behavior Learning**
    - Volatility regime classification
    - Market condition classification
    - Intraday vs overnight behavior
    - Sector-specific patterns

**MUST NOT DO**:
- ❌ Place orders, modify orders, or cancel orders
- ❌ Directly trigger execution or broker communication
- ❌ Own risk validation (moved to Risk Engine)
- ❌ Manage positions or portfolio state
- ❌ Act as execution authority
- ❌ Make trading decisions (advisory only)
- ❌ Perform position sizing (moved to Position Manager)
- ❌ Manage pyramiding logic (moved to Position Manager)
- ❌ Manage trailing stops (moved to Position Manager)

**Data Ownership**:
- Pattern occurrences and outcomes
- Setup catalog and performance
- Historical market context database
- Feature vectors and embeddings (pgvector)
- Pattern performance metrics
- Probability distributions
- Training datasets
- Model artifacts (future phases)
- Market regime classifications
- Watchlist intelligence data

**Stateless Design**:
- No runtime state persistence for analysis
- All intelligence derived from historical data and real-time market data
- Vector database (pgvector) for similarity search (stateless queries)
- Pattern database for historical learning (append-only writes)

---

### 2.2 Strategy Service - REFACTORED (Reduced Scope)

**Port**: 8006  
**Database**: `smarttrade_strategy_service` (PostgreSQL)  
**Plane**: ASYNC (decision engine and trade recommendations)

#### New Responsibility: Decision Engine (Reduced Scope)

**MUST OWN**:
1. **Trade Qualification**
   - Qualify trades based on multi-factor analysis
   - Rank trade opportunities
   - Apply confidence thresholds
   - Filter by market conditions

2. **Trade Recommendation Generation**
   - Consume AI Market Intelligence outputs (setups, probabilities, similarity scores)
   - Apply user-defined strategy rules
   - Generate trade recommendations (ENTER, EXIT, HOLD)

3. **Entry Rule Evaluation**
   - Evaluate entry conditions based on AI intelligence
   - Apply user-defined entry filters
   - Validate setup quality criteria
   - Check risk-reward ratios

4. **Exit Rule Evaluation**
   - Evaluate exit conditions based on AI intelligence
   - Apply user-defined exit filters
   - Validate exit quality criteria

5. **Decision Scoring**
   - Calculate composite decision scores from multiple intelligence sources
   - Apply confidence thresholds
   - Generate decision rankings

**Inputs**:
- AI Market Intelligence events (`ai.setup.detected`, `ai.probability.generated`, `ai.similarity.found`)
- User settings and preferences
- Portfolio state (from Portfolio Service)
- Market data (from MDS)

**Outputs**:
- Trade recommendations (`strategy.decision` events)
- Decision events for automation
- Trade qualification scores

**MUST NOT DO** (MOVED TO OTHER SERVICES):
- ❌ Place orders directly (BAS is sole execution authority)
- ❌ Perform pattern detection (AMIS responsibility)
- ❌ Calculate probabilities (AMIS responsibility)
- ❌ Maintain historical pattern database (AMIS responsibility)
- ❌ Risk validation (moved to Risk Engine)
- ❌ Position sizing (moved to Position Manager)
- ❌ Pyramiding logic (moved to Position Manager)
- ❌ Trailing stop logic (moved to Position Manager)

---

### 2.3 Risk Engine - NEW

**Port**: 8015  
**Database**: `smarttrade_risk_engine` (PostgreSQL)  
**Plane**: ASYNC (risk validation and policy enforcement)

#### Core Responsibilities

**MUST OWN**:
1. **Risk Validation**
   - Validate order requests against risk parameters
   - Check position limits
   - Check daily loss limits
   - Check portfolio-level risk limits
   - Validate leverage constraints

2. **Risk Policy Management**
   - Maintain user risk policies
   - Apply risk rules dynamically
   - Update risk limits based on market conditions

3. **Risk Monitoring**
   - Monitor real-time risk exposure
   - Generate risk alerts
   - Track risk metrics

**Inputs**:
- Order requests from Strategy Service
- User risk policies
- Portfolio state from Portfolio Service
- Market data from MDS

**Outputs**:
- Risk validation results
- Risk alerts
- Risk metrics

---

### 2.4 Position Manager - NEW

**Port**: 8016  
**Database**: `smarttrade_position_manager` (PostgreSQL)  
**Plane**: ASYNC (position management and sizing)

#### Core Responsibilities

**MUST OWN**:
1. **Position Sizing**
   - Calculate optimal position sizes based on risk parameters
   - Apply position sizing algorithms (fixed dollar, fixed percentage, volatility-based)
   - Validate position sizes against account constraints

2. **Pyramiding Management**
   - Manage pyramiding policies
   - Calculate add-on positions
   - Validate pyramiding rules

3. **Trailing Stop Management**
   - Calculate trailing stop levels
   - Update trailing stops dynamically
   - Validate trailing stop rules

4. **Position Monitoring**
   - Monitor open positions
   - Track position performance
   - Generate position alerts

**Inputs**:
- Trade recommendations from Strategy Service
- User position management policies
- Portfolio state from Portfolio Service
- Market data from MDS

**Outputs**:
- Position sizes
- Pyramiding recommendations
- Trailing stop levels
- Position alerts

---

### 2.5 Broker Adapter Service - UNCHANGED

**Port**: 8005  
**Plane**: EXECUTION (stateless execution kernel)

**No changes to BAS responsibilities**. Remains the sole execution authority with:
- Order placement, cancellation, modification
- Broker adapter plugins (Fyers, Paper)
- Event publishing via outbox pattern
- Hybrid broker state synchronization
- Local instrument master replica

---


## 3. Bounded Context Analysis

### 3.1 Domain Ownership Map

| Domain | Owner Service | Data Store | Event Publisher |
|--------|--------------|------------|-----------------|
| Candlestick Patterns | AMIS | PostgreSQL + pgvector | `ai.pattern.detected` |
| Price Action Analysis | AMIS | PostgreSQL + pgvector | `ai.pattern.detected` |
| Market Structure | AMIS | PostgreSQL + pgvector | `ai.pattern.detected` |
| Setup Intelligence | AMIS | PostgreSQL + pgvector | `ai.setup.detected` |
| Setup Quality Scoring | AMIS | PostgreSQL + pgvector | `ai.setup.scored` |
| Outcome Evaluation | AMIS | PostgreSQL | `ai.outcome.evaluated` |
| Market Regime | AMIS | PostgreSQL | `ai.regime.classified` |
| Watchlist Intelligence | AMIS | PostgreSQL | `ai.watchlist.generated` |
| Historical Patterns | AMIS | PostgreSQL + pgvector | `ai.pattern.stored` |
| Similarity Search | AMIS | pgvector | `ai.similarity.found` |
| Probability Engine | AMIS | PostgreSQL | `ai.probability.generated` |
| Feature Generation | AMIS | PostgreSQL + pgvector | Internal (not published) |
| Pattern Performance | AMIS | PostgreSQL | `ai.pattern.performance` |
| Decision Making | Strategy Service | PostgreSQL | `strategy.decision` |
| Entry/Exit Rules | Strategy Service | PostgreSQL | Internal |
| Trade Qualification | Strategy Service | PostgreSQL | Internal |
| Risk Validation | Risk Engine | PostgreSQL | `risk.validation.result` |
| Position Sizing | Position Manager | PostgreSQL | `position.size.calculated` |
| Pyramiding | Position Manager | PostgreSQL | `position.pyramid.added` |
| Trailing Stops | Position Manager | PostgreSQL | `position.trailing.stop.updated` |
| Order Execution | BAS | PostgreSQL (minimal) | `order.updated` |
| Market Data | MDS | Redis Streams | `market.quote` |
| Portfolio State | Portfolio Service | PostgreSQL | `portfolio.updated` |
| Trade History | Journal Service | PostgreSQL | `trade.executed` |

---

## 4. Execution Plane Impact

### 4.1 BAS Execution Authority - UNCHANGED

**Critical Constraint**: BAS remains the sole execution authority. No changes to execution plane architecture.

**BAS Responsibilities (Unchanged)**:
- Order placement, cancellation, modification
- Broker adapter plugins (Fyers, Paper)
- Event publishing via outbox pattern
- Hybrid broker state synchronization
- Local instrument master replica

**BAS MUST NOT**:
- ❌ Consume AI intelligence events (stays stateless)
- ❌ Make trading decisions (stays execution-only)
- ❌ Perform pattern detection (AMIS responsibility)
- ❌ Validate complex risk rules (Risk Engine responsibility)
- ❌ Calculate position sizes (Position Manager responsibility)

### 4.2 Execution Plane Isolation

The execution plane remains isolated from intelligence/decision plane:
- BAS has no dependency on AMIS, Strategy Service, Risk Engine, or Position Manager
- BAS consumes only market data (`market.quote`) for execution freshness
- All trading decisions flow through user/automation → BAS (no direct AI → BAS path)
- Event-driven communication ensures loose coupling

---

## 5. Async Plane Impact

### 5.1 New Event Flows

#### Intelligence Flow (AMIS → Strategy Service)

```
MDS (market.quote) → AMIS (Setup Detection) → 
Event: ai.setup.detected → 
Strategy Service (Decision Engine) → 
Event: strategy.decision
```

#### Risk Validation Flow (Strategy Service → Risk Engine)

```
Strategy Service (Trade Recommendation) → 
Risk Engine (Risk Validation) → 
Event: risk.validation.result → 
Position Manager (Position Sizing)
```

#### Position Management Flow (Risk Engine → Position Manager)

```
Risk Engine (Validated Trade) → 
Position Manager (Position Sizing) → 
Event: position.size.calculated → 
User/Automation (Order Placement)
```

#### Learning Flow (Journal Service → AMIS)

```
BAS (order.updated, trade.executed) → 
Journal Service (Trade History) → 
Event: trade.executed → 
AMIS (Outcome Evaluation) → 
Event: ai.outcome.evaluated → 
AMIS (Probability Engine Update)
```

#### Similarity Search Flow (AMIS Internal)

```
AMIS (Feature Generation) → 
pgvector (Vector Search) → 
AMIS (Similarity Scoring) → 
Event: ai.similarity.found → 
Strategy Service (Decision Engine)
```

### 5.2 Event Bus Integration

**AMIS Published Events**:
- `ai.pattern.detected` - Real-time pattern detection results
- `ai.setup.detected` - Multi-factor setup detection
- `ai.setup.scored` - Setup quality scores
- `ai.probability.generated` - Statistical probabilities from historical outcomes
- `ai.similarity.found` - Similar historical situations discovered
- `ai.pattern.performance` - Pattern performance metrics (periodic)
- `ai.setup.performance` - Setup performance metrics (periodic)
- `ai.outcome.evaluated` - Outcome evaluation results
- `ai.regime.classified` - Market regime classification
- `ai.watchlist.generated` - Watchlist intelligence results

**AMIS Consumed Events**:
- `market.quote` - Real-time market data (via Redis Streams)
- `trade.executed` - Trade execution events (from Journal Service via EventBus)
- `market.instrument` - Instrument master updates (from MDS)

**Strategy Service Published Events**:
- `strategy.decision` - Trade recommendations (ENTER, EXIT, HOLD)
- `strategy.automation.triggered` - Automation rule triggers

**Strategy Service Consumed Events**:
- `ai.setup.detected` - Setup intelligence from AMIS
- `ai.setup.scored` - Setup quality scores from AMIS
- `ai.probability.generated` - Probability intelligence from AMIS
- `ai.similarity.found` - Similarity intelligence from AMIS
- `ai.regime.classified` - Market regime intelligence from AMIS
- `portfolio.updated` - Portfolio state from Portfolio Service
- `market.quote` - Real-time market data (via Redis Streams)

**Risk Engine Published Events**:
- `risk.validation.result` - Risk validation results
- `risk.alert` - Risk alerts

**Risk Engine Consumed Events**:
- `strategy.decision` - Trade recommendations from Strategy Service
- `portfolio.updated` - Portfolio state from Portfolio Service
- `market.quote` - Real-time market data (via Redis Streams)

**Position Manager Published Events**:
- `position.size.calculated` - Position size calculations
- `position.pyramid.added` - Pyramiding additions
- `position.trailing.stop.updated` - Trailing stop updates

**Position Manager Consumed Events**:
- `risk.validation.result` - Validated trades from Risk Engine
- `portfolio.updated` - Portfolio state from Portfolio Service
- `market.quote` - Real-time market data (via Redis Streams)

---

## 6. Data Ownership

### 6.1 Database Per Service Architecture

| Service | Database | Primary Tables | Data Retention |
|---------|----------|----------------|----------------|
| AMIS | `smarttrade_ai_market_intelligence` | patterns, setups, setup_outcomes, features, probabilities, market_regimes, watchlists | 2+ years (historical learning) |
| Strategy Service | `smarttrade_strategy_service` | strategies, rules, decisions, automation_settings | Indefinite (user configuration) |
| Risk Engine | `smarttrade_risk_engine` | risk_policies, risk_limits, risk_alerts, risk_metrics | Indefinite (user configuration) |
| Position Manager | `smarttrade_position_manager` | position_sizes, pyramiding_rules, trailing_stops, position_alerts | Indefinite (user configuration) |
| BAS | `smarttrade_broker_adapter_service` | broker_connections, trading_accounts, instruments | Indefinite (minimal state) |
| MDS | `smarttrade_market_data_service` | instruments, trading_calendar | Indefinite (authoritative source) |
| Journal Service | `smarttrade_journal_service` | trades, orders, journal_entries | Indefinite (audit trail) |
| Portfolio Service | `smarttrade_portfolio_service` | portfolios, positions, holdings | Indefinite (read-only aggregation) |

### 6.2 Vector Database Strategy

**Phase 1**: PostgreSQL + pgvector
- **Rationale**: Simpler deployment, fewer moving parts, local-first development, easier backup strategy, sufficient scale for MVP
- **Collection**: Single pgvector extension in AMIS database
- **Vector Types**: Pattern Embeddings, Structure Embeddings, Setup Embeddings, Market Regime Embeddings

**Future Phase**: Qdrant (if needed)
- **Condition**: Only if vector volume or latency requirements exceed pgvector capabilities
- **Migration Path**: Export pgvector data → Import to Qdrant → Update AMIS configuration

### 6.3 Vector Collection Schema

**pgvector Collections**:
- `pattern_vectors` - Feature vectors for pattern similarity search
- `structure_vectors` - Feature vectors for structure similarity search
- `setup_vectors` - Feature vectors for setup similarity search
- `regime_vectors` - Feature vectors for market regime similarity search

**Data Retention**: 2+ years (aligned with PostgreSQL pattern database)

**Access Pattern**: AMIS only (no direct access from other services)

---


## 7. Event Contracts

### 7.1 New Events: AI Market Intelligence

#### ai.setup.detected (NEW)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ai.setup.detected",
  "description": "Emitted when AMIS detects a multi-factor setup",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "instrument_id",
    "timeframe",
    "setup_type",
    "setup_direction",
    "setup_score",
    "components"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.setup.detected"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "instrument_id": { "type": "string" },
    "timeframe": { 
      "type": "string",
      "enum": ["1m", "3m", "5m", "15m", "30m", "1h", "4h", "1D", "1W", "1M"]
    },
    "setup_type": {
      "type": "string",
      "enum": [
        "BULLISH_CONTINUATION", "BEARISH_CONTINUATION", 
        "BULLISH_REVERSAL", "BEARISH_REVERSAL",
        "BREAKOUT_SETUP", "BREAKDOWN_SETUP",
        "PULLBACK_SETUP", "RETEST_SETUP"
      ]
    },
    "setup_direction": { "type": "string", "enum": ["LONG", "SHORT", "NEUTRAL"] },
    "setup_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "confidence_level": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "ranking_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "components": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "component_type": { "type": "string" },
          "component_value": { "type": "string" },
          "component_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" }
        }
      }
    },
    "pattern_quality_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "structure_quality_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "volume_quality_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "trend_quality_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "context_quality_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "setup_id": { "type": "string" },
    "candle_timestamp": { "type": "string", "format": "date-time" }
  }
}
```

#### ai.setup.scored (NEW)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ai.setup.scored",
  "description": "Emitted when AMIS scores a setup's quality",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "setup_id",
    "setup_score",
    "quality_breakdown"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.setup.scored"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "setup_id": { "type": "string" },
    "setup_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "quality_breakdown": {
      "type": "object",
      "properties": {
        "pattern_quality": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "structure_quality": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "volume_quality": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "trend_quality": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "context_quality": { "type": "string", "pattern": "^\\d+\\.\\d+$" }
      }
    },
    "confidence_level": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "ranking_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" }
  }
}
```

#### ai.outcome.evaluated (NEW)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ai.outcome.evaluated",
  "description": "Emitted when AMIS evaluates the outcome of a setup or pattern",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "setup_id",
    "outcome_type",
    "future_returns",
    "drawdown",
    "reward_achieved"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.outcome.evaluated"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "setup_id": { "type": "string" },
    "pattern_id": { "type": "string" },
    "outcome_type": { "type": "string", "enum": ["SUCCESS", "FAILURE", "NEUTRAL"] },
    "evaluation_horizon": { "type": "string", "enum": ["1h", "4h", "1d", "1w", "1m"] },
    "future_returns": {
      "type": "object",
      "properties": {
        "return_1h": { "type": "string", "pattern": "^-?\\d+\\.\\d+$" },
        "return_4h": { "type": "string", "pattern": "^-?\\d+\\.\\d+$" },
        "return_1d": { "type": "string", "pattern": "^-?\\d+\\.\\d+$" },
        "return_1w": { "type": "string", "pattern": "^-?\\d+\\.\\d+$" },
        "return_1m": { "type": "string", "pattern": "^-?\\d+\\.\\d+$" }
      }
    },
    "drawdown": { "type": "string", "pattern": "^-?\\d+\\.\\d+$" },
    "reward_achieved": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "holding_period_minutes": { "type": "integer" },
    "exit_reason": { "type": "string" }
  }
}
```

#### ai.regime.classified (NEW)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ai.regime.classified",
  "description": "Emitted when AMIS classifies market regime",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "instrument_id",
    "regime_type",
    "regime_confidence"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.regime.classified"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "instrument_id": { "type": "string" },
    "regime_type": {
      "type": "string",
      "enum": [
        "TRENDING_UPTREND", "TRENDING_DOWNTREND",
        "RANGE_BOUND", "BREAKOUT_DAY",
        "GAP_UP", "GAP_DOWN",
        "HIGH_VOLATILITY", "LOW_VOLATILITY",
        "EVENT_DRIVEN", "SECTOR_REGIME"
      ]
    },
    "regime_confidence": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "volatility_level": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "trend_strength": { "type": "string", "pattern": "^\\d+\\.\\d+$" }
  }
}
```

#### ai.watchlist.generated (NEW)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ai.watchlist.generated",
  "description": "Emitted when AMIS generates watchlist intelligence",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "watchlist_type",
    "instruments"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.watchlist.generated"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "watchlist_type": {
      "type": "string",
      "enum": ["PRE_MARKET", "INTRADAY", "OPPORTUNITY", "UNUSUAL_ACTIVITY"]
    },
    "instruments": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "instrument_id": { "type": "string" },
          "score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
          "rank": { "type": "integer" },
          "reason": { "type": "string" }
        }
      }
    }
  }
}
```

#### ai.pattern.detected

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ai.pattern.detected",
  "description": "Emitted when AMIS detects a candlestick pattern or price action structure",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "instrument_id",
    "timeframe",
    "pattern_type",
    "pattern_direction",
    "confidence_score",
    "market_context"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.pattern.detected"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "instrument_id": { "type": "string" },
    "timeframe": { 
      "type": "string",
      "enum": ["1m", "3m", "5m", "15m", "30m", "1h", "4h", "1D", "1W", "1M"]
    },
    "pattern_type": {
      "type": "string",
      "enum": [
        "BULLISH_ENGULFING", "BEARISH_ENGULFING", "HAMMER", "SHOOTING_STAR",
        "DOJI", "INSIDE_BAR", "OUTSIDE_BAR", "MORNING_STAR", "EVENING_STAR",
        "BREAKOUT", "BREAKDOWN", "RETEST", "PULLBACK", "LIQUIDITY_SWEEP",
        "SUPPORT_REJECTION", "RESISTANCE_REJECTION", "BOS", "CHOCH"
      ]
    },
    "pattern_direction": { "type": "string", "enum": ["LONG", "SHORT", "NEUTRAL"] },
    "confidence_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "market_context": {
      "type": "object",
      "properties": {
        "trend": { "type": "string", "enum": ["UPTREND", "DOWNTREND", "RANGING"] },
        "volatility": { "type": "string", "enum": ["HIGH", "LOW", "MEDIUM"] },
        "volume_ratio": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "atr": { "type": "string", "pattern": "^\\d+\\.\\d+$" }
      }
    },
    "pattern_id": { "type": "string" },
    "candle_timestamp": { "type": "string", "format": "date-time" }
  }
}
```

#### ai.probability.generated

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ai.probability.generated",
  "description": "Emitted when AMIS generates statistical probabilities from historical outcomes",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "instrument_id",
    "timeframe",
    "setup_id",
    "probabilities"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.probability.generated"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "instrument_id": { "type": "string" },
    "timeframe": { "type": "string" },
    "setup_id": { "type": "string" },
    "probabilities": {
      "type": "object",
      "properties": {
        "trend_continuation": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "breakout_success": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "reversal": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "expected_move": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "risk_reward_ratio": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "confidence_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" }
      }
    },
    "sample_size": { "type": "integer" },
    "historical_win_rate": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "time_horizon": { "type": "string", "enum": ["1h", "4h", "1D", "1W"] }
  }
}
```

#### ai.similarity.found

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ai.similarity.found",
  "description": "Emitted when AMIS finds historically similar market situations or setups",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "instrument_id",
    "timeframe",
    "vector_type",
    "query_vector_id",
    "similar_situations"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.similarity.found"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "instrument_id": { "type": "string" },
    "timeframe": { "type": "string" },
    "vector_type": {
      "type": "string",
      "enum": ["pattern", "structure", "setup", "regime"]
    },
    "query_vector_id": { "type": "string" },
    "similar_situations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "setup_id": { "type": "string" },
          "pattern_id": { "type": "string" },
          "similarity_score": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
          "historical_outcome": { "type": "string", "enum": ["SUCCESS", "FAILURE", "NEUTRAL"] },
          "future_return": { "type": "string", "pattern": "^-?\\d+\\.\\d+$" },
          "holding_period": { "type": "string" },
          "occurrence_date": { "type": "string", "format": "date" }
        }
      }
    }
  }
}
```

### 7.2 New Events: Risk Engine

#### risk.validation.result

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "risk.validation.result",
  "description": "Emitted when Risk Engine validates a trade recommendation",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "decision_id",
    "validation_result",
    "risk_metrics"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["risk.validation.result"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "decision_id": { "type": "string" },
    "validation_result": { "type": "string", "enum": ["APPROVED", "REJECTED", "CONDITIONAL"] },
    "risk_metrics": {
      "type": "object",
      "properties": {
        "position_risk": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "portfolio_risk": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
        "daily_loss": { "type": "string", "pattern": "^-?\\d+\\.\\d+$" },
        "leverage": { "type": "string", "pattern": "^\\d+\\.\\d+$" }
      }
    },
    "rejection_reason": { "type": "string" },
    "conditional_requirements": { "type": "array", "items": { "type": "string" } }
  }
}
```

### 7.3 New Events: Position Manager

#### position.size.calculated

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "position.size.calculated",
  "description": "Emitted when Position Manager calculates position size",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "decision_id",
    "position_size",
    "sizing_method"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["position.size.calculated"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "decision_id": { "type": "string" },
    "position_size": { "type": "integer" },
    "sizing_method": { "type": "string", "enum": ["FIXED_DOLLAR", "FIXED_PERCENTAGE", "VOLATILITY_BASED"] },
    "risk_amount": { "type": "string", "pattern": "^\\d+\\.\\d+$" },
    "stop_loss": { "type": "string", "pattern": "^\\d+\\.\\d+$" }
  }
}
```

---


## 8. REST Contracts

### 8.1 AMIS REST API (BaseServiceClient Pattern)

#### Setup Intelligence Endpoints (NEW)

```
GET /api/v1/setups/{instrument_id}/{timeframe}
Description: Get latest detected setups for instrument and timeframe
Response: SetupDetectionResponse

GET /api/v1/setups/{setup_id}
Description: Get setup details by ID
Response: SetupDetailResponse

GET /api/v1/setups/{instrument_id}/{timeframe}/history
Description: Get historical setup occurrences
Query Params: from_date, to_date, setup_type
Response: SetupHistoryResponse

GET /api/v1/setups/performance/{setup_type}
Description: Get performance metrics for setup type
Response: SetupPerformanceResponse
```

#### Pattern Intelligence Endpoints

```
GET /api/v1/patterns/{instrument_id}/{timeframe}
Description: Get latest detected patterns for instrument and timeframe
Response: PatternDetectionResponse

GET /api/v1/patterns/{pattern_id}
Description: Get pattern details by ID
Response: PatternDetailResponse
```

#### Probability Endpoints

```
GET /api/v1/probabilities/{instrument_id}/{timeframe}
Description: Get latest probability estimates
Response: ProbabilityResponse

GET /api/v1/probabilities/{setup_id}
Description: Get probabilities for specific setup
Response: ProbabilityDetailResponse
```

#### Similarity Search Endpoints

```
POST /api/v1/similarity/search
Description: Find similar historical situations or setups
Request: SimilaritySearchRequest
Response: SimilaritySearchResponse

GET /api/v1/similarity/{query_id}
Description: Get similarity search results
Response: SimilaritySearchResponse
```

#### Market Regime Endpoints (NEW)

```
GET /api/v1/regime/{instrument_id}
Description: Get current market regime for instrument
Response: MarketRegimeResponse

GET /api/v1/regime/market
Description: Get overall market regime classification
Response: MarketRegimeResponse
```

#### Watchlist Intelligence Endpoints (NEW)

```
GET /api/v1/watchlist/pre-market
Description: Get pre-market watchlist
Response: WatchlistResponse

GET /api/v1/watchlist/opportunities
Description: Get opportunity watchlist
Response: WatchlistResponse

GET /api/v1/watchlist/unusual-activity
Description: Get unusual activity watchlist
Response: WatchlistResponse
```

### 8.2 Strategy Service REST API (Reduced Scope)

#### Decision Engine Endpoints

```
POST /api/v1/decisions/evaluate
Description: Evaluate trade conditions based on AI intelligence
Request: DecisionEvaluationRequest
Response: DecisionEvaluationResponse

GET /api/v1/decisions/{decision_id}
Description: Get decision details
Response: DecisionDetailResponse

GET /api/v1/decisions/user/{user_id}
Description: Get recent decisions for user
Response: DecisionHistoryResponse
```

#### Strategy Configuration Endpoints

```
POST /api/v1/strategies
Description: Create strategy configuration
Request: StrategyConfigRequest
Response: StrategyConfigResponse

PUT /api/v1/strategies/{strategy_id}
Description: Update strategy configuration
Request: StrategyConfigUpdateRequest
Response: StrategyConfigResponse

GET /api/v1/strategies/{strategy_id}
Description: Get strategy configuration
Response: StrategyConfigResponse
```

### 8.3 Risk Engine REST API (NEW)

#### Risk Validation Endpoints

```
POST /api/v1/risk/validate
Description: Validate trade recommendation against risk policies
Request: RiskValidationRequest
Response: RiskValidationResponse

GET /api/v1/risk/policies/{user_id}
Description: Get user risk policies
Response: RiskPoliciesResponse

PUT /api/v1/risk/policies/{policy_id}
Description: Update risk policy
Request: RiskPolicyUpdateRequest
Response: RiskPolicyResponse
```

### 8.4 Position Manager REST API (NEW)

#### Position Management Endpoints

```
POST /api/v1/positions/calculate-size
Description: Calculate position size based on risk parameters
Request: PositionSizeRequest
Response: PositionSizeResponse

POST /api/v1/positions/pyramid
Description: Calculate pyramiding additions
Request: PyramidingRequest
Response: PyramidingResponse

POST /api/v1/positions/trailing-stop
Description: Calculate trailing stop level
Request: TrailingStopRequest
Response: TrailingStopResponse
```

### 8.5 BaseServiceClient Usage Pattern

All service-to-service communication MUST use `BaseServiceClient` from `smarttrade-common`:

```python
from smarttrade_common.http_client import BaseServiceClient

# AMIS client in Strategy Service
amis_client = BaseServiceClient(
    service_name="ai-market-intelligence-service",
    base_url=settings.AI_MARKET_INTELLIGENCE_SERVICE_URL,
    timeout=5.0
)

# Get setup intelligence
setups = await amis_client.get(
    f"/api/v1/setups/{instrument_id}/{timeframe}"
)

# Search similarity
similar_situations = await amis_client.post(
    "/api/v1/similarity/search",
    json=SimilaritySearchRequest(...)
)

# Risk Engine client in Position Manager
risk_client = BaseServiceClient(
    service_name="risk-engine",
    base_url=settings.RISK_ENGINE_URL,
    timeout=5.0
)

# Validate risk
validation = await risk_client.post(
    "/api/v1/risk/validate",
    json=RiskValidationRequest(...)
)
```

---

## 9. Database Design

### 9.1 AMIS Database Schema (Enhanced)

#### patterns table

```sql
CREATE TABLE patterns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pattern_id VARCHAR(255) UNIQUE NOT NULL,
    instrument_id VARCHAR(255) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    pattern_type VARCHAR(100) NOT NULL,
    pattern_direction VARCHAR(10) NOT NULL,
    confidence_score DECIMAL(5,4) NOT NULL,
    
    -- Market context
    trend VARCHAR(20) NOT NULL,
    volatility VARCHAR(20) NOT NULL,
    volume_ratio DECIMAL(10,4),
    atr DECIMAL(15,4),
    
    -- Price context
    open_price DECIMAL(15,4),
    high_price DECIMAL(15,4),
    low_price DECIMAL(15,4),
    close_price DECIMAL(15,4),
    
    -- Timestamps
    candle_timestamp TIMESTAMP NOT NULL,
    detected_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Metadata
    pattern_status VARCHAR(20) DEFAULT 'ACTIVE',
    lifecycle_stage VARCHAR(20) DEFAULT 'EMERGED',
    
    -- Indexes
    INDEX idx_patterns_instrument_timeframe (instrument_id, timeframe),
    INDEX idx_patterns_type (pattern_type),
    INDEX idx_patterns_timestamp (candle_timestamp),
    INDEX idx_patterns_status (pattern_status)
);
```

#### setups table (NEW)

```sql
CREATE TABLE setups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    setup_id VARCHAR(255) UNIQUE NOT NULL,
    instrument_id VARCHAR(255) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    setup_type VARCHAR(100) NOT NULL,
    setup_direction VARCHAR(10) NOT NULL,
    
    -- Setup scoring
    setup_score DECIMAL(5,2) NOT NULL,
    confidence_level DECIMAL(5,4) NOT NULL,
    ranking_score DECIMAL(5,4) NOT NULL,
    
    -- Quality breakdown
    pattern_quality_score DECIMAL(5,4),
    structure_quality_score DECIMAL(5,4),
    volume_quality_score DECIMAL(5,4),
    trend_quality_score DECIMAL(5,4),
    context_quality_score DECIMAL(5,4),
    
    -- Market context
    trend VARCHAR(20) NOT NULL,
    volatility VARCHAR(20) NOT NULL,
    market_regime VARCHAR(50),
    
    -- Timestamps
    candle_timestamp TIMESTAMP NOT NULL,
    detected_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Metadata
    setup_status VARCHAR(20) DEFAULT 'ACTIVE',
    lifecycle_stage VARCHAR(20) DEFAULT 'EMERGED',
    version INTEGER DEFAULT 1,
    
    -- Indexes
    INDEX idx_setups_instrument_timeframe (instrument_id, timeframe),
    INDEX idx_setups_type (setup_type),
    INDEX idx_setups_score (setup_score),
    INDEX idx_setups_timestamp (candle_timestamp),
    INDEX idx_setups_status (setup_status)
);
```

#### setup_components table (NEW)

```sql
CREATE TABLE setup_components (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    setup_id VARCHAR(255) NOT NULL REFERENCES setups(setup_id),
    component_type VARCHAR(100) NOT NULL,
    component_value VARCHAR(255) NOT NULL,
    component_score DECIMAL(5,4) NOT NULL,
    
    -- Metadata
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_setup_components_setup (setup_id),
    INDEX idx_setup_components_type (component_type)
);
```

#### setup_outcomes table (NEW)

```sql
CREATE TABLE setup_outcomes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    setup_id VARCHAR(255) NOT NULL REFERENCES setups(setup_id),
    
    -- Outcome classification
    outcome_type VARCHAR(20) NOT NULL,
    outcome_confidence DECIMAL(5,4),
    
    -- Performance metrics
    future_return_1h DECIMAL(10,4),
    future_return_4h DECIMAL(10,4),
    future_return_1d DECIMAL(10,4),
    future_return_1w DECIMAL(10,4),
    future_return_1m DECIMAL(10,4),
    
    -- Risk metrics
    drawdown DECIMAL(10,4),
    reward_achieved DECIMAL(10,4),
    
    -- Holding period
    holding_period_minutes INTEGER,
    exit_reason VARCHAR(50),
    
    -- Market context at outcome
    outcome_trend VARCHAR(20),
    outcome_volatility VARCHAR(20),
    outcome_regime VARCHAR(50),
    
    -- Timestamps
    outcome_timestamp TIMESTAMP NOT NULL,
    recorded_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_setup_outcomes_setup (setup_id),
    INDEX idx_setup_outcomes_type (outcome_type),
    INDEX idx_setup_outcomes_timestamp (outcome_timestamp)
);
```

#### market_regimes table (NEW)

```sql
CREATE TABLE market_regimes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id VARCHAR(255) NOT NULL,
    regime_type VARCHAR(50) NOT NULL,
    regime_confidence DECIMAL(5,4) NOT NULL,
    
    -- Regime metrics
    volatility_level DECIMAL(10,4),
    trend_strength DECIMAL(5,4),
    range_width DECIMAL(10,4),
    
    -- Timestamps
    regime_timestamp TIMESTAMP NOT NULL,
    classified_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_regimes_instrument (instrument_id),
    INDEX idx_regimes_type (regime_type),
    INDEX idx_regimes_timestamp (regime_timestamp)
);
```

#### watchlists table (NEW)

```sql
CREATE TABLE watchlists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    watchlist_type VARCHAR(50) NOT NULL,
    watchlist_name VARCHAR(255) NOT NULL,
    
    -- Metadata
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_watchlists_user (user_id),
    INDEX idx_watchlists_type (watchlist_type),
    INDEX idx_watchlists_active (is_active)
);
```

#### watchlist_items table (NEW)

```sql
CREATE TABLE watchlist_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    watchlist_id UUID NOT NULL REFERENCES watchlists(id),
    instrument_id VARCHAR(255) NOT NULL,
    score DECIMAL(5,4) NOT NULL,
    rank INTEGER NOT NULL,
    reason VARCHAR(500),
    
    -- Metadata
    added_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_watchlist_items_watchlist (watchlist_id),
    INDEX idx_watchlist_items_instrument (instrument_id),
    INDEX idx_watchlist_items_score (score)
);
```

#### pattern_outcomes table

```sql
CREATE TABLE pattern_outcomes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pattern_id VARCHAR(255) NOT NULL REFERENCES patterns(pattern_id),
    
    -- Outcome classification
    outcome_type VARCHAR(20) NOT NULL,
    outcome_confidence DECIMAL(5,4),
    
    -- Performance metrics
    future_return_1h DECIMAL(10,4),
    future_return_4h DECIMAL(10,4),
    future_return_1d DECIMAL(10,4),
    future_return_1w DECIMAL(10,4),
    
    -- Holding period
    holding_period_minutes INTEGER,
    exit_reason VARCHAR(50),
    
    -- Market context at outcome
    outcome_trend VARCHAR(20),
    outcome_volatility VARCHAR(20),
    
    -- Timestamps
    outcome_timestamp TIMESTAMP NOT NULL,
    recorded_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_outcomes_pattern (pattern_id),
    INDEX idx_outcomes_type (outcome_type),
    INDEX idx_outcomes_timestamp (outcome_timestamp)
);
```

#### features table

```sql
CREATE TABLE features (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pattern_id VARCHAR(255) REFERENCES patterns(pattern_id),
    setup_id VARCHAR(255) REFERENCES setups(setup_id),
    instrument_id VARCHAR(255) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    
    -- Technical indicators
    atr DECIMAL(15,4),
    volume_ratio DECIMAL(10,4),
    relative_strength DECIMAL(10,4),
    gap_percentage DECIMAL(10,4),
    distance_from_vwap DECIMAL(10,4),
    distance_from_ema DECIMAL(10,4),
    
    -- Candlestick features
    body_size_ratio DECIMAL(5,4),
    wick_ratio DECIMAL(5,4),
    upper_shadow_ratio DECIMAL(5,4),
    lower_shadow_ratio DECIMAL(5,4),
    
    -- Momentum features
    momentum_score DECIMAL(5,4),
    trend_strength DECIMAL(5,4),
    
    -- Context features
    time_of_day VARCHAR(10),
    day_of_week VARCHAR(10),
    
    -- Timestamps
    feature_timestamp TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_features_pattern (pattern_id),
    INDEX idx_features_setup (setup_id),
    INDEX idx_features_instrument (instrument_id, timeframe)
);
```

#### probabilities table

```sql
CREATE TABLE probabilities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    setup_id VARCHAR(255) REFERENCES setups(setup_id),
    pattern_id VARCHAR(255) REFERENCES patterns(pattern_id),
    instrument_id VARCHAR(255) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    
    -- Probability estimates
    trend_continuation_prob DECIMAL(5,4),
    breakout_success_prob DECIMAL(5,4),
    reversal_prob DECIMAL(5,4),
    expected_move_prob DECIMAL(5,4),
    
    -- Risk-reward metrics
    risk_reward_ratio DECIMAL(10,4),
    confidence_score DECIMAL(5,4),
    
    -- Sample information
    sample_size INTEGER NOT NULL,
    historical_win_rate DECIMAL(5,4),
    
    -- Time horizon
    time_horizon VARCHAR(10) NOT NULL,
    
    -- Timestamps
    calculated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    valid_until TIMESTAMP NOT NULL,
    
    -- Indexes
    INDEX idx_probabilities_setup (setup_id),
    INDEX idx_probabilities_pattern (pattern_id),
    INDEX idx_probabilities_instrument (instrument_id, timeframe),
    INDEX idx_probabilities_horizon (time_horizon)
);
```

#### vectors table (pgvector)

```sql
CREATE TABLE vectors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vector_id VARCHAR(255) UNIQUE NOT NULL,
    vector_type VARCHAR(50) NOT NULL,
    embedding vector(100),
    
    -- Reference
    pattern_id VARCHAR(255) REFERENCES patterns(pattern_id),
    setup_id VARCHAR(255) REFERENCES setups(setup_id),
    instrument_id VARCHAR(255) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    
    -- Metadata
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_vectors_type (vector_type),
    INDEX idx_vectors_instrument (instrument_id)
);

-- Create vector similarity index
CREATE INDEX ON vectors USING ivfflat (embedding vector_cosine_ops);
```

### 9.2 Strategy Service Database Schema (Reduced Scope)

#### strategies table

```sql
CREATE TABLE strategies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    strategy_name VARCHAR(255) NOT NULL,
    strategy_type VARCHAR(50) NOT NULL,
    
    -- Strategy configuration
    entry_rules JSONB NOT NULL,
    exit_rules JSONB NOT NULL,
    
    -- Intelligence integration
    setup_filters JSONB,
    probability_thresholds JSONB,
    similarity_thresholds JSONB,
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_strategies_user (user_id),
    INDEX idx_strategies_active (is_active)
);
```

#### decisions table

```sql
CREATE TABLE decisions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    decision_id VARCHAR(255) UNIQUE NOT NULL,
    user_id UUID NOT NULL,
    strategy_id UUID REFERENCES strategies(id),
    
    -- Decision details
    symbol VARCHAR(255) NOT NULL,
    action VARCHAR(20) NOT NULL,
    confidence DECIMAL(5,4) NOT NULL,
    
    -- Intelligence sources
    intelligence_sources JSONB NOT NULL,
    
    -- Reasoning
    reasoning JSONB NOT NULL,
    
    -- Timestamps
    decision_timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_decisions_user (user_id),
    INDEX idx_decisions_symbol (symbol),
    INDEX idx_decisions_timestamp (decision_timestamp)
);
```

### 9.3 Risk Engine Database Schema (NEW)

#### risk_policies table

```sql
CREATE TABLE risk_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    policy_name VARCHAR(255) NOT NULL,
    
    -- Risk limits
    max_position_size DECIMAL(15,4),
    max_daily_loss DECIMAL(15,4),
    max_portfolio_risk DECIMAL(5,4),
    max_leverage DECIMAL(5,4),
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_risk_policies_user (user_id),
    INDEX idx_risk_policies_active (is_active)
);
```

#### risk_metrics table

```sql
CREATE TABLE risk_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    
    -- Current risk metrics
    position_risk DECIMAL(15,4),
    portfolio_risk DECIMAL(5,4),
    daily_loss DECIMAL(15,4),
    current_leverage DECIMAL(5,4),
    
    -- Timestamps
    calculated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_risk_metrics_user (user_id),
    INDEX idx_risk_metrics_timestamp (calculated_at)
);
```

### 9.4 Position Manager Database Schema (NEW)

#### position_sizes table

```sql
CREATE TABLE position_sizes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    decision_id VARCHAR(255) NOT NULL,
    user_id UUID NOT NULL,
    
    -- Position sizing
    position_size INTEGER NOT NULL,
    sizing_method VARCHAR(50) NOT NULL,
    risk_amount DECIMAL(15,4),
    stop_loss DECIMAL(15,4),
    
    -- Timestamps
    calculated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_position_sizes_decision (decision_id),
    INDEX idx_position_sizes_user (user_id)
);
```

#### pyramiding_rules table

```sql
CREATE TABLE pyramiding_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    
    -- Pyramiding configuration
    max_additions INTEGER NOT NULL,
    addition_threshold DECIMAL(5,4),
    addition_size DECIMAL(5,4),
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_pyramiding_rules_user (user_id),
    INDEX idx_pyramiding_rules_active (is_active)
);
```

#### trailing_stops table

```sql
CREATE TABLE trailing_stops (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    position_id VARCHAR(255) NOT NULL,
    
    -- Trailing stop configuration
    trailing_stop_type VARCHAR(50) NOT NULL,
    trailing_distance DECIMAL(10,4),
    current_stop_level DECIMAL(15,4),
    
    -- Timestamps
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    -- Indexes
    INDEX idx_trailing_stops_user (user_id),
    INDEX idx_trailing_stops_position (position_id)
);
```

---


## 10. Training Architecture

### 10.1 Phased Training Roadmap

#### Phase 1: Statistical Learning Engine (P0)

**Objective**: No ML. Store setups, outcomes, and probabilities using statistical methods.

**Components**:
- Setup occurrence tracking (PostgreSQL)
- Outcome measurement (future returns at multiple horizons)
- Probability calculation (frequentist statistics)
- Setup performance analytics (aggregation queries)
- Market regime classification (statistical rules)

**Deliverables**:
- Setup database with 2+ year history
- Outcome tracking for all setups
- Statistical probability engine
- Setup performance dashboard
- Historical learning pipeline
- Market regime classification

**No ML Dependencies**: Pure statistical analysis using SQL and Python data processing.

#### Phase 2: Classical Machine Learning (P1)

**Objective**: Introduce supervised learning for setup classification and outcome prediction.

**Recommended Models**:
- XGBoost (gradient boosting)
- LightGBM (lightweight gradient boosting)
- Random Forest (ensemble learning)

**Use Cases**:
- Setup quality classification
- Outcome prediction (win/loss)
- Probability regression
- Feature importance analysis
- Market regime prediction

#### Phase 3: Sequence Learning (Future)

**Objective**: Introduce deep learning for sequential pattern recognition and market prediction.

**Recommended Models**:
- LSTM (Long Short-Term Memory)
- Temporal CNN (Convolutional Neural Networks)
- Transformer (attention-based)

**Use Cases**:
- Multi-candle setup recognition
- Market regime prediction
- Volatility forecasting
- Trend continuation prediction

---

## 11. Historical Learning Architecture

### 11.1 Setup Occurrence Tracking

**Data Flow**:
```
Real-time Setup Detection → Setup Database → 
Outcome Tracking (from trade events) → Setup Performance Analytics
```

**Components**:
1. **Setup Collector**: Ingests setup detection events
2. **Outcome Tracker**: Monitors trade execution events for setup outcomes
3. **Performance Calculator**: Aggregates setup performance metrics
4. **Learning Scheduler**: Periodic batch processing for historical analysis

### 11.2 Outcome Measurement Strategy

**Time Horizons**:
- 1-hour future return
- 4-hour future return
- 1-day future return
- 1-week future return
- 1-month future return

**Outcome Classification**:
- SUCCESS: Positive return above threshold
- FAILURE: Negative return below threshold
- NEUTRAL: Return within threshold range

**Thresholds** (configurable):
- SUCCESS: > 0.5% return
- FAILURE: < -0.3% return
- NEUTRAL: -0.3% to 0.5% return

### 11.3 Historical Learning Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│              Historical Learning Pipeline                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Setup Detection (Real-time)                                 │
│     └─> Store in setups table                                    │
│                                                                  │
│  2. Trade Execution (from Journal Service)                       │
│     └─> Link to setups via instrument/timeframe                   │
│     └─> Calculate future returns at multiple horizons           │
│     └─> Store in setup_outcomes table                           │
│                                                                  │
│  3. Performance Aggregation (Periodic - daily)                   │
│     └─> Calculate win rates by setup type                         │
│     └─> Calculate average returns by setup type                   │
│     └─> Calculate risk-reward ratios                            │
│     └─> Update setup performance metrics                          │
│                                                                  │
│  4. Probability Calculation (On-demand)                        │
│     └─> Query historical outcomes for similar setups               │
│     └─> Calculate statistical probabilities                      │
│     └─> Cache in probabilities table                            │
│                                                                  │
│  5. Similarity Search (On-demand)                               │
│     └─> Generate feature vectors for current setup               │
│     └─> Query pgvector for similar historical setups              │
│     └─> Return historical outcomes for similar setups              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 11.4 Market Regime Classification

**Classification Logic** (statistical rules for Phase 1):
- **Trending Uptrend**: Higher Highs + Higher Lows for 20+ candles
- **Trending Downtrend**: Lower Highs + Lower Lows for 20+ candles
- **Range Bound**: Price oscillating within 2% range for 20+ candles
- **Breakout Day**: Price breaks range with > 2% move + volume expansion
- **Gap Up**: Open > previous close by > 1%
- **Gap Down**: Open < previous close by > 1%
- **High Volatility**: ATR > 2% of price
- **Low Volatility**: ATR < 0.5% of price

---

## 12. Similarity Search Architecture

### 12.1 Vector Generation Strategy

**Vector Types** (Enhanced for Phase 1):
1. **Pattern Embeddings**: Feature vectors for candlestick patterns
   - Body size, wick ratios, volume profile
   - Pattern type encoding
   - Timeframe normalization

2. **Structure Embeddings**: Feature vectors for market structure
   - HH/HL/LH/LL sequences
   - Trend strength indicators
   - Support/resistance levels

3. **Setup Embeddings**: Feature vectors for multi-factor setups
   - Composite of pattern + structure + volume + trend
   - Quality score components
   - Market regime context

4. **Market Regime Embeddings**: Feature vectors for market conditions
   - Volatility regime
   - Trend regime
   - Sector regime

### 12.2 Similarity Search Flow

```
┌─────────────────────────────────────────────────────────────────┐
│              Similarity Search Flow                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Query Setup Detected                                        │
│     └─> Generate feature vector (embedding)                     │
│                                                                  │
│  2. Vector Search (pgvector)                                    │
│     └─> Query for similar vectors (cosine similarity)           │
│     └─> Return top K similar historical setups                 │
│                                                                  │
│  3. Outcome Retrieval                                           │
│     └─> Fetch historical outcomes for similar setups            │
│     └─> Calculate aggregate statistics (win rate, avg return)   │
│                                                                  │
│  4. Intelligence Publication                                    │
│     └─> Publish ai.similarity.found event                       │
│     └─> Include historical outcomes and probabilities           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 12.3 pgvector Configuration

**Phase 1 Setup**:
- Extension: `pgvector` (PostgreSQL extension)
- Vector dimension: 100 (configurable)
- Similarity metric: Cosine similarity
- Index type: IVFFlat (approximate nearest neighbor)
- Index parameters: `lists = 100` (tunable for performance)

**Migration to Qdrant (if needed)**:
- Export vectors from pgvector
- Import to Qdrant collection
- Update AMIS configuration to use Qdrant client
- Phase out pgvector dependency

---

## 13. Probability Engine Design

### 13.1 Statistical Probability Calculation

**Frequentist Approach** (Phase 1):
```
P(success | setup_type, market_regime) = 
    Count(successful_outcomes) / Count(total_outcomes)
```

**Conditional Probabilities**:
- Probability by setup type
- Probability by market regime
- Probability by time of day
- Probability by sector

**Bayesian Enhancement** (optional):
- Incorporate prior probabilities
- Update with new evidence
- Smooth with regularization

### 13.2 Probability Calculation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│              Probability Calculation Flow                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Setup Detected                                              │
│     └─> Query historical outcomes for similar setups           │
│                                                                  │
│  2. Statistical Aggregation                                     │
│     └─> Count successful outcomes                               │
│     └─> Count total outcomes                                    │
│     └─> Calculate win rate                                      │
│     └─> Calculate average returns                               │
│                                                                  │
│  3. Probability Generation                                       │
│     └─> Generate trend continuation probability                 │
│     └─> Generate breakout success probability                   │
│     └─> Generate reversal probability                           │
│     └─> Calculate risk-reward expectancy                        │
│                                                                  │
│  4. Confidence Scoring                                           │
│     └─> Calculate confidence based on sample size                │
│     └─> Apply confidence intervals                              │
│                                                                  │
│  5. Publication                                                  │
│     └─> Publish ai.probability.generated event                   │
│     └─> Cache in probabilities table                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 13.3 Confidence Calculation

**Sample Size-Based Confidence**:
- Small sample (< 10): Low confidence (0.3-0.5)
- Medium sample (10-50): Medium confidence (0.5-0.7)
- Large sample (> 50): High confidence (0.7-0.9)

**Confidence Formula**:
```
confidence = min(0.9, sample_size / (sample_size + 10))
```

---

## 14. Strategy Service Interaction

### 14.1 Intelligence Consumption Pattern

**Strategy Service Decision Flow**:
```
1. Subscribe to AMIS events (ai.setup.detected, ai.probability.generated, ai.similarity.found)
2. Receive setup intelligence
3. Apply user-defined strategy rules
4. Evaluate entry/exit conditions
5. Generate trade recommendation
6. Publish strategy.decision event
```

### 14.2 Decision Scoring

**Composite Decision Score**:
```
decision_score = w1 * setup_score + 
                 w2 * probability_score + 
                 w3 * similarity_score + 
                 w4 * regime_alignment_score
```

**Weights** (configurable):
- `w1`: Setup quality weight (default: 0.4)
- `w2`: Probability weight (default: 0.3)
- `w3`: Similarity weight (default: 0.2)
- `w4`: Regime alignment weight (default: 0.1)

### 14.3 Trade Qualification

**Qualification Criteria**:
- Setup score > threshold (default: 70)
- Probability > threshold (default: 0.6)
- Similarity score > threshold (default: 0.7)
- Regime alignment > threshold (default: 0.5)

**Ranking**:
- Sort by composite decision score
- Apply user-defined filters
- Return top N opportunities

---

## 15. Future ML Architecture

### 15.1 Model Registry (Phase 2+)

**MLflow Integration**:
- Model versioning
- Experiment tracking
- Model deployment
- Performance monitoring

**Model Types**:
- Setup quality classifier (XGBoost)
- Outcome predictor (LightGBM)
- Market regime classifier (Random Forest)
- Probability regressor (XGBoost)

### 15.2 Feature Store (Phase 2+)

**Feature Store Architecture**:
- Offline feature computation (batch)
- Online feature serving (real-time)
- Feature versioning
- Feature lineage tracking

**Feature Categories**:
- Technical indicators (ATR, RSI, MACD)
- Price action features (body size, wick ratio)
- Market structure features (HH/HL/LH/LL)
- Context features (time of day, sector)

### 15.3 Training Pipeline (Phase 2+)

**Training Workflow**:
```
1. Data Collection (from setup_outcomes table)
2. Feature Engineering (Feature Store)
3. Label Generation (outcome classification)
4. Train/Test Split (time-based)
5. Model Training (XGBoost/LightGBM)
6. Model Evaluation (cross-validation)
7. Model Registration (MLflow)
8. Model Deployment (AMIS)
```

---

## 16. Migration Plan

### 16.1 Migration Phases

#### Phase 0: Preparation (Week 1-2)
- Create AMIS database schema
- Set up pgvector extension
- Implement AMIS service skeleton
- Create Risk Engine database schema
- Create Position Manager database schema
- Update service discovery configuration

#### Phase 1: SPS Decommissioning (Week 3-4)
- Migrate SPS pattern detection to AMIS
- Migrate SPS pattern database to AMIS
- Update event subscriptions (SPS → AMIS)
- Decommission SPS service
- Remove SPS from service discovery

#### Phase 2: Strategy Service Refactoring (Week 5-6)
- Remove risk validation from Strategy Service
- Remove position sizing from Strategy Service
- Remove pyramiding logic from Strategy Service
- Remove trailing stop logic from Strategy Service
- Update Strategy Service to consume AMIS events
- Implement decision engine logic

#### Phase 3: Risk Engine Implementation (Week 7-8)
- Implement Risk Engine service
- Implement risk validation logic
- Implement risk policy management
- Implement risk monitoring
- Integrate with Strategy Service

#### Phase 4: Position Manager Implementation (Week 9-10)
- Implement Position Manager service
- Implement position sizing logic
- Implement pyramiding logic
- Implement trailing stop logic
- Integrate with Risk Engine

#### Phase 5: AMIS Feature Implementation (Week 11-16)
- Implement Setup Intelligence Engine
- Implement Outcome Evaluation Engine
- Implement Market Regime Engine
- Implement Watchlist Intelligence Engine
- Implement Setup Quality Scoring
- Implement Similarity Search (pgvector)
- Implement Probability Engine
- Implement Historical Learning Pipeline

### 16.2 Rollback Strategy

**Rollback Triggers**:
- Critical bugs in AMIS
- Performance degradation
- Data integrity issues
- Event bus failures

**Rollback Steps**:
1. Stop AMIS, Risk Engine, Position Manager
2. Restore SPS service (if needed)
3. Restore Strategy Service to previous version
4. Update event subscriptions
5. Verify system stability

---

## 17. Phased Implementation Plan

### 17.1 Phase 1: Statistical Learning Engine (P0) - Weeks 1-8

**Week 1-2: Foundation**
- AMIS service skeleton
- Database schema (PostgreSQL + pgvector)
- Event bus integration
- Market data consumption (Redis Streams)

**Week 3-4: Pattern Detection**
- Candlestick pattern detection
- Price action analysis
- Market structure analysis
- Pattern database population

**Week 5-6: Setup Intelligence**
- Setup detection logic
- Setup quality scoring
- Setup catalog management
- Setup lifecycle management

**Week 7-8: Learning Foundation**
- Outcome evaluation engine
- Historical learning pipeline
- Statistical probability engine
- Market regime classification

**Deliverables**:
- AMIS service with pattern detection
- Setup intelligence engine
- Statistical probability engine
- Historical learning pipeline
- Market regime classification

### 17.2 Phase 2: Strategy Service Refactoring (P0) - Weeks 9-10

**Week 9: Risk and Position Extraction**
- Remove risk validation from Strategy Service
- Remove position sizing from Strategy Service
- Remove pyramiding logic from Strategy Service
- Remove trailing stop logic from Strategy Service

**Week 10: Decision Engine**
- Implement decision engine logic
- Integrate AMIS intelligence consumption
- Implement trade qualification
- Implement decision scoring

**Deliverables**:
- Refactored Strategy Service (reduced scope)
- Decision engine implementation
- AMIS intelligence integration

### 17.3 Phase 3: Risk Engine and Position Manager (P0) - Weeks 11-14

**Week 11-12: Risk Engine**
- Implement Risk Engine service
- Implement risk validation logic
- Implement risk policy management
- Implement risk monitoring
- Integrate with Strategy Service

**Week 13-14: Position Manager**
- Implement Position Manager service
- Implement position sizing logic
- Implement pyramiding logic
- Implement trailing stop logic
- Integrate with Risk Engine

**Deliverables**:
- Risk Engine service
- Position Manager service
- Risk and position management integration

### 17.4 Phase 4: Similarity Search and Watchlist Intelligence (P1) - Weeks 15-18

**Week 15-16: Similarity Search**
- Implement feature vector generation
- Implement pgvector integration
- Implement similarity search logic
- Implement vector types (pattern, structure, setup, regime)

**Week 17-18: Watchlist Intelligence**
- Implement watchlist generation
- Implement opportunity ranking
- Implement unusual activity detection
- Implement pre-market watchlist

**Deliverables**:
- Similarity search engine (pgvector)
- Watchlist intelligence engine
- Enhanced setup discovery

### 17.5 Phase 5: Classical Machine Learning (P1) - Weeks 19-24

**Week 19-20: ML Infrastructure**
- Set up MLflow
- Implement feature store
- Implement training pipeline
- Implement model registry

**Week 21-22: Model Training**
- Train setup quality classifier (XGBoost)
- Train outcome predictor (LightGBM)
- Train market regime classifier (Random Forest)
- Train probability regressor (XGBoost)

**Week 23-24: Model Deployment**
- Deploy models to AMIS
- Implement model serving
- Implement model monitoring
- Implement A/B testing

**Deliverables**:
- ML infrastructure (MLflow, feature store)
- Trained ML models
- Model deployment and monitoring

---

## 18. Risks and Mitigations

### 18.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| pgvector performance degradation | High | Medium | Monitor query latency; plan Qdrant migration |
| Event bus overload | High | Low | Implement event throttling; use message batching |
| Database schema migration issues | High | Medium | Comprehensive testing; rollback procedures |
| Setup detection accuracy issues | Medium | High | Extensive backtesting; user feedback loop |
| Similarity search quality | Medium | Medium | A/B testing with different vector configurations |

### 18.2 Operational Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| SPS decommissioning disruption | High | Low | Parallel running; gradual cutover |
| Strategy Service refactoring bugs | High | Medium | Comprehensive testing; rollback procedures |
| Risk Engine policy misconfiguration | High | Medium | Policy validation; sandbox testing |
| Position Manager calculation errors | High | Low | Extensive testing; user approval workflows |

### 18.3 Data Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Historical data loss | High | Low | Regular backups; data replication |
| Setup outcome tracking errors | Medium | Medium | Data validation; anomaly detection |
| Probability calculation errors | Medium | Low | Statistical validation; confidence intervals |
| Vector database corruption | Medium | Low | Regular backups; integrity checks |

### 18.4 Architecture Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Service boundary violations | Medium | Medium | Strict API contracts; code reviews |
| Event contract changes | High | Low | Versioned event schemas; backward compatibility |
| Database ownership conflicts | Medium | Low | Clear domain ownership; data access controls |
| ML model drift | Medium | High | Model monitoring; retraining schedules |

---

## 19. Future AI Features Alignment

### 19.1 Supported Future Features

The architecture supports the following future AI features:

**AI Trade Coach**:
- Setup intelligence provides trade coaching data
- Similarity search provides historical analogs
- Probability engine provides confidence estimates

**AI Day Planner**:
- Watchlist intelligence provides pre-market planning
- Market regime classification provides context
- Setup quality scoring provides opportunity ranking

**AI Smart Watchlist**:
- Watchlist intelligence engine provides automated watchlist generation
- Opportunity ranking provides prioritization
- Unusual activity detection provides alerts

**Runtime Position Advisor**:
- Similarity search provides historical analogs for current positions
- Market regime classification provides context
- Probability engine provides exit guidance

**Historical Setup Analysis**:
- Setup catalog provides historical setup database
- Setup performance tracking provides analytics
- Outcome evaluation provides learning data

**Trade Quality Scoring**:
- Setup quality scoring provides quality metrics
- Pattern quality scoring provides component analysis
- Structure quality scoring provides context analysis

**Trending vs Sideways Market Detection**:
- Market regime classification provides regime detection
- Trend strength metrics provide trend analysis
- Volatility classification provides volatility context

**Probability-Based Decision Support**:
- Probability engine provides statistical probabilities
- Similarity search provides historical analogs
- Confidence scoring provides reliability estimates

**Pattern Similarity Discovery**:
- Similarity search provides pattern analogs
- Vector embeddings provide similarity matching
- Historical outcomes provide performance data

**Behavioral Coaching**:
- Setup performance tracking provides behavioral insights
- Outcome evaluation provides learning data
- Market regime classification provides context

---

## 20. Appendix

### 20.1 Configuration

**AMIS Configuration**:
```yaml
ai_market_intelligence:
  database:
    host: localhost
    port: 5432
    name: smarttrade_ai_market_intelligence
  pgvector:
    enabled: true
    vector_dimension: 100
    similarity_metric: cosine
  market_data:
    redis_streams:
      host: localhost
      port: 6379
      stream: market.quote
  learning:
    outcome_horizons: [1h, 4h, 1d, 1w, 1m]
    success_threshold: 0.005
    failure_threshold: -0.003
```

**Risk Engine Configuration**:
```yaml
risk_engine:
  database:
    host: localhost
    port: 5432
    name: smarttrade_risk_engine
  validation:
    default_max_position_size: 10000
    default_max_daily_loss: 1000
    default_max_portfolio_risk: 0.02
    default_max_leverage: 2.0
```

**Position Manager Configuration**:
```yaml
position_manager:
  database:
    host: localhost
    port: 5432
    name: smarttrade_position_manager
  sizing:
    default_method: VOLATILITY_BASED
    default_risk_per_trade: 0.01
  pyramiding:
    default_max_additions: 3
    default_addition_threshold: 0.02
  trailing_stop:
    default_type: PERCENTAGE
    default_distance: 0.02
```

### 20.2 Monitoring

**AMIS Metrics**:
- Pattern detection rate (patterns/minute)
- Setup detection rate (setups/minute)
- Similarity search latency (ms)
- Probability calculation latency (ms)
- Historical learning pipeline status

**Risk Engine Metrics**:
- Risk validation rate (validations/minute)
- Risk rejection rate (%)
- Risk alert rate (alerts/minute)
- Portfolio risk exposure (%)

**Position Manager Metrics**:
- Position sizing rate (sizings/minute)
- Pyramiding additions (additions/day)
- Trailing stop updates (updates/day)
- Position monitoring status

### 20.3 Testing Strategy

**Unit Tests**:
- Pattern detection algorithms
- Setup detection logic
- Quality scoring algorithms
- Probability calculation logic
- Similarity search algorithms
- Risk validation logic
- Position sizing algorithms

**Integration Tests**:
- Event bus integration
- Database integration
- pgvector integration
- Service-to-service communication
- Market data consumption

**E2E Tests**:
- Setup detection → Decision flow
- Risk validation → Position sizing flow
- Historical learning pipeline
- Similarity search flow
- Watchlist generation flow

---

**Document End**

