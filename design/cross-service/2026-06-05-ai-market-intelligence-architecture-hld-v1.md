> **DEPRECATED**: This document describes the legacy `ai-service` (port 8014), which has been deprecated. Its functionality has been migrated to `amis-core-service` (port 8000), `amis-lab-service` (port 8016), and `signal-processor-service` (port 8012). See `ai-service-deprecation-plan.md` for details.

# High Level Design (HLD): AI Market Intelligence Service Architecture Refactoring

**Document Version**: 1.0  
**Date**: 2026-06-05  
**Status**: Design Draft  
**Author**: SmartTrade Architecture Team  
**Scope**: Cross-service architecture refactoring (Signal Processor → AI Market Intelligence, Strategy Service refactoring)

---

## Executive Summary

This HLD defines the architecture for removing the Signal Processor Service and introducing the AI Market Intelligence Service as the central intelligence and learning engine for SmartTrade. The refactoring maintains the existing execution plane vs async plane architecture, preserves BAS as the sole execution authority, and ensures AI components remain advisory and decision-support only.

**Key Changes**:
- **Remove**: Signal Processor Service (SPS) - pattern detection and analysis capabilities move to AI Market Intelligence
- **Add**: AI Market Intelligence Service (AMIS) - unified intelligence platform for market learning, pattern discovery, and probability generation
- **Refactor**: Strategy Service - evolves into Decision Engine, consuming AI intelligence to produce trade recommendations

**Success Criteria**:
1. Zero disruption to BAS execution authority and trading operations
2. Seamless migration of SPS capabilities to AMIS with enhanced learning features
3. Strategy Service successfully refactored as Decision Engine with AI intelligence integration
4. All services maintain stateless architecture and event-driven communication patterns
5. Phase 1 (statistical learning) delivers production-ready pattern intelligence without ML dependencies

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
│                     │   Qdrant     │                                │
│                     │  Vector DB   │                                │
│                     └──────────────┘                                │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Journal    │  │  Portfolio   │  │ Notification │          │
│  │    (8007)    │  │    (8008)    │  │    (8011)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
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

---

## 2. Service Responsibilities

### 2.1 AI Market Intelligence Service (AMIS) - NEW

**Port**: 8014  
**Database**: `smarttrade_ai_market_intelligence` (PostgreSQL)  
**Vector Database**: Qdrant (dedicated instance)  
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

4. **Historical Pattern Mining**
   - Store all pattern occurrences with full market context
   - Track pattern outcomes over multiple time horizons
   - Build pattern intelligence database with performance metrics
   - Maintain pattern lifecycle (emerged, confirmed, failed, invalidated)

5. **Similarity Search**
   - Feature vector generation for market situations
   - Nearest-neighbor search using vector database (Qdrant)
   - Pattern clustering and grouping
   - Historical analog discovery
   - Case-based reasoning for similar market conditions

6. **Probability Engine**
   - Generate statistical probabilities from historical outcomes
   - Trend continuation probability
   - Breakout success probability
   - Reversal probability
   - Expected move probability
   - Risk-reward expectancy
   - Confidence score calculation

7. **Feature Generation**
   - Generate ML-ready features for training
   - ATR, Volume Ratio, Relative Strength
   - Gap Percentage, Distance from VWAP/EMA
   - Body Size Ratio, Wick Ratio
   - Momentum Score, Trend Strength

8. **Historical Dataset Management**
   - Maintain training datasets for ML models
   - Versioned dataset storage
   - Data quality validation
   - Feature store management

9. **Pattern Performance Tracking**
   - Track win rates by pattern type
   - Track failure rates by market condition
   - Track future returns by pattern
   - Track holding period performance
   - Track time-of-day performance

10. **Market Behavior Learning**
    - Volatility regime classification
    - Market condition classification
    - Intraday vs overnight behavior
    - Sector-specific patterns

**MUST NOT DO**:
- ❌ Place orders, modify orders, or cancel orders
- ❌ Directly trigger execution or broker communication
- ❌ Own risk validation (moved to Strategy Service)
- ❌ Manage positions or portfolio state
- ❌ Act as execution authority
- ❌ Make trading decisions (advisory only)

**Data Ownership**:
- Pattern occurrences and outcomes
- Historical market context database
- Feature vectors and embeddings
- Pattern performance metrics
- Probability distributions
- Training datasets
- Model artifacts (future phases)

**Stateless Design**:
- No runtime state persistence for analysis
- All intelligence derived from historical data and real-time market data
- Vector database for similarity search (stateless queries)
- Pattern database for historical learning (append-only writes)

---

### 2.2 Strategy Service - REFACTORED

**Port**: 8006  
**Database**: `smarttrade_strategy_service` (PostgreSQL)  
**Plane**: ASYNC (decision engine and trade recommendations)

#### New Responsibility: Decision Engine

**MUST OWN**:
1. **Intelligence-to-Decision Conversion**
   - Consume AI Market Intelligence outputs (patterns, probabilities, similarity scores)
   - Apply user-defined strategy rules
   - Generate trade recommendations (ENTER, EXIT, HOLD)
   - Calculate position sizing based on risk parameters

2. **Entry Rule Evaluation**
   - Evaluate entry conditions based on AI intelligence
   - Apply user-defined entry filters
   - Validate setup quality criteria
   - Check risk-reward ratios

3. **Exit Rule Evaluation**
   - Evaluate exit conditions based on AI intelligence
   - Apply user-defined exit filters
   - Trailing stop calculation
   - Take-profit level determination

4. **Automation Policies**
   - Manage user automation settings
   - Apply automation rules based on AI intelligence
   - Handle pyramiding policies
   - Manage position scaling rules

5. **Risk-Reward Filters**
   - Apply user risk constraints
   - Validate position sizing
   - Check portfolio-level risk limits
   - Validate leverage constraints

6. **Trade Qualification**
   - Qualify trades based on multi-factor analysis
   - Rank trade opportunities
   - Apply confidence thresholds
   - Filter by market conditions

**Inputs**:
- AI Market Intelligence events (`ai.pattern.detected`, `ai.probability.generated`, `ai.similarity.found`)
- User settings and preferences
- Portfolio state (from Portfolio Service)
- Risk constraints (user-defined)
- Market data (from MDS)

**Outputs**:
- Trade recommendations (`strategy.decision` events)
- Decision events for automation
- Trade qualification scores
- Risk-adjusted position sizes

**MUST NOT DO**:
- ❌ Place orders directly (BAS is sole execution authority)
- ❌ Perform pattern detection (AMIS responsibility)
- ❌ Calculate probabilities (AMIS responsibility)
- ❌ Maintain historical pattern database (AMIS responsibility)

---

### 2.3 Broker Adapter Service - UNCHANGED

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
| Candlestick Patterns | AMIS | PostgreSQL + Qdrant | `ai.pattern.detected` |
| Price Action Analysis | AMIS | PostgreSQL + Qdrant | `ai.pattern.detected` |
| Market Structure | AMIS | PostgreSQL + Qdrant | `ai.pattern.detected` |
| Historical Patterns | AMIS | PostgreSQL + Qdrant | `ai.pattern.stored` |
| Similarity Search | AMIS | Qdrant | `ai.similarity.found` |
| Probability Engine | AMIS | PostgreSQL | `ai.probability.generated` |
| Feature Generation | AMIS | PostgreSQL + Qdrant | Internal (not published) |
| Pattern Performance | AMIS | PostgreSQL | `ai.pattern.performance` |
| Decision Making | Strategy Service | PostgreSQL | `strategy.decision` |
| Entry/Exit Rules | Strategy Service | PostgreSQL | Internal |
| Risk Management | Strategy Service | PostgreSQL | Internal |
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
- ❌ Validate complex risk rules (Strategy Service responsibility)

### 4.2 Execution Plane Isolation

The execution plane remains isolated from intelligence/decision plane:
- BAS has no dependency on AMIS or Strategy Service
- BAS consumes only market data (`market.quote`) for execution freshness
- All trading decisions flow through user/automation → BAS (no direct AI → BAS path)
- Event-driven communication ensures loose coupling

---

## 5. Async Plane Impact

### 5.1 New Event Flows

#### Intelligence Flow (AMIS → Strategy Service)

```
MDS (market.quote) → AMIS (Pattern Detection) → 
Event: ai.pattern.detected → 
Strategy Service (Decision Engine) → 
Event: strategy.decision
```

#### Learning Flow (Journal Service → AMIS)

```
BAS (order.updated, trade.executed) → 
Journal Service (Trade History) → 
Event: trade.executed → 
AMIS (Historical Learning) → 
Event: ai.pattern.performance
```

#### Similarity Search Flow (AMIS Internal)

```
AMIS (Feature Generation) → 
Qdrant (Vector Search) → 
AMIS (Similarity Scoring) → 
Event: ai.similarity.found → 
Strategy Service (Decision Engine)
```

### 5.2 Event Bus Integration

**AMIS Published Events**:
- `ai.pattern.detected` - Real-time pattern detection results
- `ai.probability.generated` - Statistical probabilities from historical outcomes
- `ai.similarity.found` - Similar historical situations discovered
- `ai.pattern.performance` - Pattern performance metrics (periodic)
- `ai.market.condition` - Market regime/volatility classification

**AMIS Consumed Events**:
- `market.quote` - Real-time market data (via Redis Streams)
- `trade.executed` - Trade execution events (from Journal Service via EventBus)
- `market.instrument` - Instrument master updates (from MDS)

**Strategy Service Published Events**:
- `strategy.decision` - Trade recommendations (ENTER, EXIT, HOLD)
- `strategy.automation.triggered` - Automation rule triggers

**Strategy Service Consumed Events**:
- `ai.pattern.detected` - Pattern intelligence from AMIS
- `ai.probability.generated` - Probability intelligence from AMIS
- `ai.similarity.found` - Similarity intelligence from AMIS
- `portfolio.updated` - Portfolio state from Portfolio Service
- `market.quote` - Real-time market data (via Redis Streams)

---

## 6. Data Ownership

### 6.1 Database Per Service Architecture

| Service | Database | Primary Tables | Data Retention |
|---------|----------|----------------|----------------|
| AMIS | `smarttrade_ai_market_intelligence` | patterns, pattern_outcomes, features, probabilities, market_conditions | 2+ years (historical learning) |
| Strategy Service | `smarttrade_strategy_service` | strategies, rules, decisions, automation_settings | Indefinite (user configuration) |
| BAS | `smarttrade_broker_adapter_service` | broker_connections, trading_accounts, instruments | Indefinite (minimal state) |
| MDS | `smarttrade_market_data_service` | instruments, trading_calendar | Indefinite (authoritative source) |
| Journal Service | `smarttrade_journal_service` | trades, orders, journal_entries | Indefinite (audit trail) |
| Portfolio Service | `smarttrade_portfolio_service` | portfolios, positions, holdings | Indefinite (read-only aggregation) |

### 6.2 Vector Database Ownership

**Qdrant Instance**: Dedicated for AMIS similarity search

**Collections**:
- `pattern_vectors` - Feature vectors for pattern similarity search
- `market_context_vectors` - Feature vectors for market situation similarity
- `trade_vectors` - Feature vectors for trade similarity (future phase)

**Data Retention**: 2+ years (aligned with PostgreSQL pattern database)

**Access Pattern**: AMIS only (no direct access from other services)

---

## 7. Event Contracts

### 7.1 New Events: AI Market Intelligence

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
    "pattern_id",
    "probabilities"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.probability.generated"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "instrument_id": { "type": "string" },
    "timeframe": { "type": "string" },
    "pattern_id": { "type": "string" },
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
  "description": "Emitted when AMIS finds historically similar market situations",
  "type": "object",
  "required": [
    "event_id",
    "topic",
    "timestamp",
    "instrument_id",
    "timeframe",
    "query_vector_id",
    "similar_situations"
  ],
  "properties": {
    "event_id": { "type": "string", "format": "uuid" },
    "topic": { "type": "string", "enum": ["ai.similarity.found"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "instrument_id": { "type": "string" },
    "timeframe": { "type": "string" },
    "query_vector_id": { "type": "string" },
    "similar_situations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
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

---

## 8. REST Contracts

### 8.1 AMIS REST API (BaseServiceClient Pattern)

#### Pattern Intelligence Endpoints

```
GET /api/v1/patterns/{instrument_id}/{timeframe}
Description: Get latest detected patterns for instrument and timeframe
Response: PatternDetectionResponse

GET /api/v1/patterns/{pattern_id}
Description: Get pattern details by ID
Response: PatternDetailResponse

GET /api/v1/patterns/{instrument_id}/{timeframe}/history
Description: Get historical pattern occurrences
Query Params: from_date, to_date, pattern_type
Response: PatternHistoryResponse
```

#### Probability Endpoints

```
GET /api/v1/probabilities/{instrument_id}/{timeframe}
Description: Get latest probability estimates
Response: ProbabilityResponse

GET /api/v1/probabilities/{pattern_id}
Description: Get probabilities for specific pattern
Response: ProbabilityDetailResponse
```

#### Similarity Search Endpoints

```
POST /api/v1/similarity/search
Description: Find similar historical situations
Request: SimilaritySearchRequest
Response: SimilaritySearchResponse

GET /api/v1/similarity/{query_id}
Description: Get similarity search results
Response: SimilaritySearchResponse
```

### 8.2 Strategy Service REST API (Enhanced)

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

---

## 9. Database Design

### 9.1 AMIS Database Schema

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

#### probabilities table

```sql
CREATE TABLE probabilities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
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
    INDEX idx_probabilities_pattern (pattern_id),
    INDEX idx_probabilities_instrument (instrument_id, timeframe),
    INDEX idx_probabilities_horizon (time_horizon)
);
```

---

## 10. Training Architecture

### 10.1 Phased Training Roadmap

#### Phase 1: Statistical Learning Engine (P0)

**Objective**: No ML. Store patterns, outcomes, and probabilities using statistical methods.

**Components**:
- Pattern occurrence tracking (PostgreSQL)
- Outcome measurement (future returns)
- Probability calculation (frequentist statistics)
- Pattern performance analytics (aggregation queries)

**Deliverables**:
- Pattern database with 2+ year history
- Outcome tracking for all patterns
- Statistical probability engine
- Pattern performance dashboard
- Historical learning pipeline

**No ML Dependencies**: Pure statistical analysis using SQL and Python data processing.

#### Phase 2: Classical Machine Learning (P1)

**Objective**: Introduce supervised learning for pattern classification and outcome prediction.

**Recommended Models**:
- XGBoost (gradient boosting)
- LightGBM (lightweight gradient boosting)
- Random Forest (ensemble learning)

**Use Cases**:
- Pattern quality classification
- Outcome prediction (win/loss)
- Probability regression
- Feature importance analysis

#### Phase 3: Sequence Learning (Future)

**Objective**: Introduce deep learning for sequential pattern recognition and market prediction.

**Recommended Models**:
- LSTM (Long Short-Term Memory)
- Temporal CNN (Convolutional Neural Networks)
- Transformer (attention-based)

---

## 11. Historical Learning Architecture

### 11.1 Pattern Occurrence Tracking

**Data Flow**:
```
Real-time Pattern Detection → Pattern Database → 
Outcome Tracking (from trade events) → Pattern Performance Analytics
```

### 11.2 Outcome Measurement Strategy

**Time Horizons**:
- 1-hour future return
- 4-hour future return
- 1-day future return
- 1-week future return

**Outcome Classification**:
- SUCCESS: Positive return above threshold
- FAILURE: Negative return below threshold
- NEUTRAL: Return within threshold range

---

## 12. Similarity Search Architecture

### 12.1 Vector Database Integration (Qdrant)

**Deployment**: Dedicated Qdrant instance for AMIS

**Collections**:
1. **pattern_vectors**: Feature vectors for pattern similarity
2. **market_context_vectors**: Feature vectors for market situation similarity
3. **trade_vectors**: Feature vectors for trade similarity (future phase)

### 12.2 Feature Generation Pipeline

**Input Features**:
- Technical indicators (ATR, RSI, MACD, etc.)
- Price action features (body size, wick ratio, etc.)
- Market context (trend, volatility, volume)
- Temporal features (time of day, day of week)

**Vector Dimensionality**: 50-100 dimensions (configurable)

---

## 13. Probability Engine Design

### 13.1 Statistical Probability Calculation

**Approach**: Frequentist statistics based on historical outcomes

**Formula**:
```
P(success) = count(successful_outcomes) / count(total_outcomes)
P(failure) = count(failed_outcomes) / count(total_outcomes)
P(neutral) = count(neutral_outcomes) / count(total_outcomes)

Expected Return = sum(future_returns) / count(total_outcomes)
Risk-Reward Ratio = abs(Expected Return) / Max Drawdown
```

### 13.2 Probability Caching Strategy

**Cache Key**: `{instrument_id}:{timeframe}:{pattern_type}:{market_condition}`

**Cache TTL**: 5 minutes (configurable)

---

## 14. Strategy Service Interaction

### 14.1 Decision Engine Architecture

The Strategy Service is refactored as a Decision Engine that:
1. Consumes AI intelligence from AMIS
2. Applies user-defined strategy rules
3. Generates trade recommendations
4. Publishes `strategy.decision` events

### 14.2 Intelligence Integration

**Pattern Intelligence Consumption**:
- Subscribe to `ai.pattern.detected` events
- Evaluate patterns against strategy rules
- Generate entry/exit signals

**Probability Intelligence Consumption**:
- Subscribe to `ai.probability.generated` events
- Update decision confidence with probability intelligence
- Apply probability thresholds

**Similarity Intelligence Consumption**:
- Subscribe to `ai.similarity.found` events
- Analyze similar historical outcomes
- Adjust decisions based on historical success rates

---

## 15. Future ML Architecture

### 15.1 Extension Points for Advanced ML

**Model Registry Integration**:
- MLflow for experiment tracking
- Model versioning in database
- A/B testing framework
- Model performance monitoring

**Feature Store**:
- Centralized feature storage
- Feature versioning
- Feature lineage tracking
- Real-time feature serving

---

## 16. Migration Plan

### 16.1 Phase 1: AI Market Intelligence Service Setup (Week 1-4)

**Objective**: Stand up AMIS infrastructure and migrate SPS pattern detection capabilities.

**Tasks**:
1. Create AMIS service repository
2. Set up PostgreSQL database (`smarttrade_ai_market_intelligence`)
3. Set up Qdrant vector database
4. Implement pattern detection engines (migrate from SPS)
5. Implement feature generation pipeline
6. Implement historical data ingestion from MDS
7. Set up Redis Stream consumer for `market.quote`
8. Implement event publishing (`ai.pattern.detected`)
9. Set up instrument master replica via InstrumentSyncService
10. Write unit and integration tests

**Deliverables**:
- AMIS service running on port 8014
- Pattern detection operational for all timeframes
- Historical pattern database populated
- Event publishing verified
- Test coverage > 80%

### 16.2 Phase 2: Historical Learning & Probability Engine (Week 5-8)

**Objective**: Implement historical learning and statistical probability calculation.

**Tasks**:
1. Implement outcome tracking from Journal Service events
2. Implement performance calculation pipeline
3. Implement statistical probability engine
4. Set up Qdrant vector database integration
5. Implement feature vector generation
6. Implement similarity search
7. Implement probability caching
8. Set up periodic batch processing
9. Implement performance analytics
10. Write unit and integration tests

**Deliverables**:
- Historical learning pipeline operational
- Probability engine functional
- Similarity search operational
- Performance analytics dashboard
- Test coverage > 80%

### 16.3 Phase 3: Strategy Service Refactoring (Week 9-12)

**Objective**: Refactor Strategy Service as Decision Engine with AI intelligence integration.

**Tasks**:
1. Design decision engine architecture
2. Implement intelligence consumers (pattern, probability, similarity)
3. Implement rule evaluation engine
4. Implement decision generation logic
5. Implement risk validation
6. Implement position sizing
7. Update event contracts (`strategy.decision`)
8. Implement REST API endpoints
9. Migrate existing strategy configurations
10. Write unit and integration tests

**Deliverables**:
- Strategy Service refactored as Decision Engine
- AI intelligence integration complete
- Decision generation operational
- Risk validation functional
- Test coverage > 80%

### 16.4 Phase 4: Signal Processor Service Decommissioning (Week 13-14)

**Objective**: Decommission SPS after validating AMIS capabilities.

**Tasks**:
1. Validate AMIS pattern detection matches SPS
2. Validate all SPS consumers migrated to AMIS events
3. Update frontend to consume AMIS endpoints
4. Update service dependencies (remove SPS references)
5. Stop SPS service
6. Remove SPS from docker-compose
7. Clean up SPS database
8. Update documentation
9. Archive SPS repository
10. Post-migration validation

**Deliverables**:
- SPS service stopped and removed
- All consumers migrated to AMIS
- Frontend updated
- Documentation updated
- Clean removal of SPS artifacts

### 16.5 Phase 5: End-to-End Testing & Optimization (Week 15-16)

**Objective**: Comprehensive testing and performance optimization.

**Tasks**:
1. End-to-end testing of complete flow
2. Performance testing and optimization
3. Load testing for AMIS
4. Failover testing
5. Data validation testing
6. User acceptance testing
7. Documentation finalization
8. Runbook creation
9. Monitoring setup
10. Go-live preparation

**Deliverables**:
- End-to-end testing complete
- Performance optimized
- Monitoring operational
- Documentation complete
- Runbooks ready

---

## 17. Phased Implementation Plan

### 17.1 Phase 1: Statistical Learning Engine (P0) - Week 1-8

**Priority**: P0 (Critical for MVP)

**Objective**: Production-ready pattern intelligence without ML dependencies.

**Deliverables**:
1. ✅ AI Market Intelligence Service operational
2. ✅ Pattern detection for all timeframes
3. ✅ Historical pattern database
4. ✅ Outcome tracking pipeline
5. ✅ Statistical probability engine
6. ✅ Similarity search (Qdrant)
7. ✅ Performance analytics
8. ✅ Strategy Service refactored as Decision Engine
9. ✅ AI intelligence integration
10. ✅ Signal Processor Service decommissioned

**Success Criteria**:
- Pattern detection accuracy > 95%
- Probability calculation error < 5%
- Similarity search latency < 100ms
- Decision generation latency < 200ms
- Test coverage > 80%
- Zero downtime during migration

### 17.2 Phase 2: Classical Machine Learning (P1) - Week 9-16

**Priority**: P1 (Enhancement)

**Objective**: Introduce supervised learning for improved pattern classification and outcome prediction.

**Deliverables**:
1. XGBoost/LightGBM model training pipeline
2. Feature store implementation
3. Model registry (MLflow)
4. Model serving infrastructure
5. A/B testing framework
6. Model performance monitoring

### 17.3 Phase 3: Sequence Learning (Future) - Week 17+

**Priority**: Future (Research)

**Objective**: Introduce deep learning for sequential pattern recognition and market prediction.

**Deliverables**:
1. LSTM/Transformer model training
2. GPU infrastructure
3. Real-time inference pipeline
4. Advanced model serving

---

## 18. Risks and Mitigations

### 18.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Pattern detection accuracy degradation | High | Medium | Comprehensive testing against SPS baseline; gradual rollout with A/B testing |
| Qdrant performance bottlenecks | High | Low | Horizontal scaling; caching layer; fallback to PostgreSQL queries |
| Probability calculation errors | High | Medium | Statistical validation; confidence intervals; minimum sample size checks |
| Strategy Service decision logic errors | High | Medium | Extensive unit testing; integration testing; user acceptance testing |
| Event delivery failures | Medium | Low | EventBus retry logic; DLQ monitoring; circuit breakers |

### 18.2 Operational Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Migration downtime | High | Low | Blue-green deployment; rollback plan; zero-downtime migration strategy |
| Data loss during migration | High | Low | Database backups; point-in-time recovery; data validation scripts |
| Performance degradation | Medium | Medium | Load testing; performance monitoring; auto-scaling |
| Frontend integration issues | Medium | Medium | Frontend testing; API versioning; backward compatibility |

### 18.3 Architectural Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Service boundary violations | High | Low | Strict code review; architectural decision records; service ownership documentation |
| Event contract mismatches | Medium | Medium | Schema validation; contract testing; versioning strategy |
| Database coupling | Medium | Low | Database-per-service enforcement; cross-service query prohibition |

---

## 19. Conclusion

This HLD defines a comprehensive architecture for refactoring SmartTrade's intelligence and decision-making capabilities. The key principles are:

1. **Maintain Execution Authority**: BAS remains the sole execution authority with no changes
2. **Intelligence Centralization**: AMIS becomes the unified intelligence platform for all market learning
3. **Decision Engine Refactoring**: Strategy Service evolves into a Decision Engine consuming AI intelligence
4. **Phased Implementation**: Statistical learning (P0) → Classical ML (P1) → Deep Learning (Future)
5. **Stateless Architecture**: All services maintain stateless design with event-driven communication
6. **Production-Grade Approach**: No shortcuts; all implementations must be production-ready from day one

The migration plan ensures zero disruption to trading operations while delivering enhanced pattern intelligence, historical learning, and probability generation capabilities.

---

## Appendix A: Configuration

### A.1 Environment Variables (AMIS)

```bash
# Core
ENV=local
SERVICE_NAME=ai-market-intelligence-service
SERVICE_PORT=8014
LOG_LEVEL=INFO

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@host:…/smarttrade_ai_market_intelligence

# Redis
REDIS_URL=redis://localhost:6379/0

# Qdrant
QDRANT_URL=http://qdrant:6333
QDRANT_API_KEY=

# Market Data Service
MARKET_DATA_SERVICE_URL=http://market-data-service:8000

# Instrument Master Sync
INSTRUMENT_MASTER_SYNC_INTERVAL_SECONDS=21600  # 6 hours

# Pattern Detection
SUPPORTED_TIMEFRAMES=1m,3m,5m,15m,30m,1h,4h,1D,1W,1M
PATTERN_CONFIDENCE_THRESHOLD=0.7

# Probability Engine
MIN_SAMPLE_SIZE=30
PROBABILITY_CACHE_TTL_SECONDS=300

# Similarity Search
VECTOR_DIMENSIONALITY=100
SIMILARITY_THRESHOLD=0.7
TOP_K_SIMILAR=10

# Historical Learning
OUTCOME_THRESHOLDS_SUCCESS=0.5
OUTCOME_THRESHOLDS_FAILURE=-0.3
DATA_RETENTION_YEARS=2
```

### A.2 Environment Variables (Strategy Service - Enhanced)

```bash
# Core
ENV=local
SERVICE_NAME=strategy-service
SERVICE_PORT=8006
LOG_LEVEL=INFO

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@host:…/smarttrade_strategy_service

# Redis
REDIS_URL=redis://localhost:6379/0

# AI Market Intelligence Service
AI_MARKET_INTELLIGENCE_SERVICE_URL=http://ai-market-intelligence-service:8014

# Portfolio Service
PORTFOLIO_SERVICE_URL=http://portfolio-service:8000

# Decision Engine
DECISION_CONFIDENCE_THRESHOLD=0.7
DECISION_SCORE_WEIGHTS_PATTERN=0.30
DECISION_SCORE_WEIGHTS_PROBABILITY=0.30
DECISION_SCORE_WEIGHTS_SIMILARITY=0.20
DECISION_SCORE_WEIGHTS_RULES=0.20
```

---

## Appendix B: Monitoring

### B.1 Key Metrics (AMIS)

**Pattern Detection**:
- `pattern_detection_total` - Total patterns detected
- `pattern_detection_by_type` - Patterns by type
- `pattern_detection_latency` - Detection latency histogram
- `pattern_detection_errors` - Detection error rate

**Probability Engine**:
- `probability_calculation_total` - Total probability calculations
- `probability_calculation_latency` - Calculation latency histogram
- `probability_cache_hit_rate` - Cache hit rate
- `probability_sample_size_distribution` - Sample size distribution

**Similarity Search**:
- `similarity_search_total` - Total similarity searches
- `similarity_search_latency` - Search latency histogram
- `similarity_score_distribution` - Similarity score distribution
- `qdrant_query_latency` - Qdrant query latency

**Historical Learning**:
- `outcome_tracking_total` - Total outcomes tracked
- `outcome_tracking_latency` - Tracking latency histogram
- `performance_aggregation_latency` - Aggregation latency histogram
- `pattern_performance_win_rate` - Win rate by pattern type

### B.2 Key Metrics (Strategy Service)

**Decision Engine**:
- `decision_generation_total` - Total decisions generated
- `decision_generation_latency` - Generation latency histogram
- `decision_by_action` - Decisions by action (ENTER/EXIT/HOLD)
- `decision_confidence_distribution` - Confidence score distribution

**Intelligence Integration**:
- `intelligence_pattern_consumed_total` - Pattern events consumed
- `intelligence_probability_consumed_total` - Probability events consumed
- `intelligence_similarity_consumed_total` - Similarity events consumed
- `intelligence_integration_latency` - Integration latency histogram

**Rule Evaluation**:
- `rule_evaluation_total` - Total rule evaluations
- `rule_evaluation_latency` - Evaluation latency histogram
- `rule_pass_rate` - Rule pass rate by rule type

---

## Appendix C: Testing Strategy

### C.1 Unit Testing

**AMIS**:
- Pattern detection engines (all pattern types)
- Feature generation (all feature types)
- Probability calculation (statistical formulas)
- Similarity search (vector operations)
- Event publishing (schema validation)

**Strategy Service**:
- Intelligence consumers (all event types)
- Rule evaluation (all rule types)
- Decision generation (all scenarios)
- Risk validation (all risk checks)
- Position sizing (all sizing methods)

### C.2 Integration Testing

**AMIS**:
- Pattern detection → Database persistence
- Feature generation → Qdrant insertion
- Probability calculation → Event publishing
- Similarity search → Qdrant query
- Historical learning → Outcome tracking

**Strategy Service**:
- Intelligence consumption → Decision generation
- Rule evaluation → Decision scoring
- Risk validation → Position sizing
- Decision generation → Event publishing

### C.3 End-to-End Testing

**Complete Flow**:
1. MDS publishes market.quote
2. AMIS detects pattern and publishes ai.pattern.detected
3. AMIS calculates probability and publishes ai.probability.generated
4. AMIS performs similarity search and publishes ai.similarity.found
5. Strategy Service consumes events and generates strategy.decision
6. User/automation places order via BAS
7. BAS executes order and publishes order.updated
8. Journal Service records trade and publishes trade.executed
9. AMIS tracks outcome and updates pattern performance

---

**Document End**
