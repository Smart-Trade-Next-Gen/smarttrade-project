> **DEPRECATED**: This document describes the legacy `ai-service` (port 8014), which has been deprecated. Its functionality has been migrated to `amis-core-service` (port 8000), `amis-lab-service` (port 8016), and `signal-processor-service` (port 8012). See `ai-service-deprecation-plan.md` for details.

# Low Level Design (LLD): AI Market Intelligence Service (AMIS)

## Document Information
- **Feature**: AI Market Intelligence Service
- **Phase**: V1 (Statistical Learning Engine)
- **Service**: AI Market Intelligence Service (AMIS)
- **Port**: 8014
- **Database**: smarttrade_ai_market_intelligence (PostgreSQL + pgvector)
- **Date**: 2026-06-05
- **Based on**: HLD v2.0

---

## 1. Folder Structure

```
ai-market-intelligence-service/
├── src/
│   └── ai_market_intelligence_service/
│       ├── __init__.py
│       ├── main.py                              # FastAPI app + router registration
│       ├── lifespan.py                          # Startup/shutdown hooks
│       ├── config.py                            # Pydantic settings
│       ├── models.py                            # SQLAlchemy database entities
│       ├── schemas.py                           # Pydantic request/response models
│       ├── repositories.py                      # Data access layer
│       │
│       ├── pattern_detection/                  # Pattern Detection Module
│       │   ├── __init__.py
│       │   ├── candlestick_patterns.py          # Candlestick pattern detection logic
│       │   ├── price_action_analysis.py         # Price action pattern detection
│       │   ├── market_structure.py              # Market structure analysis
│       │   ├── pattern_detector.py             # Unified pattern detection orchestrator
│       │   └── schemas.py                      # Pattern-specific Pydantic models
│       │
│       ├── setup_intelligence/                  # Setup Intelligence Module
│       │   ├── __init__.py
│       │   ├── setup_detector.py               # Multi-factor setup detection
│       │   ├── setup_scorer.py                 # Setup quality scoring engine
│       │   ├── setup_catalog.py                # Setup catalog management
│       │   ├── setup_lifecycle.py              # Setup lifecycle management
│       │   └── schemas.py                      # Setup-specific Pydantic models
│       │
│       ├── outcome_evaluation/                  # Outcome Evaluation Module
│       │   ├── __init__.py
│       │   ├── outcome_tracker.py               # Track setup outcomes from trade events
│       │   ├── return_calculator.py             # Calculate future returns at horizons
│       │   ├── outcome_classifier.py           # Classify outcomes (SUCCESS/FAILURE/NEUTRAL)
│       │   └── schemas.py                      # Outcome-specific Pydantic models
│       │
│       ├── probability_engine/                  # Probability Engine Module
│       │   ├── __init__.py
│       │   ├── statistical_calculator.py       # Statistical probability calculations
│       │   ├── probability_cache.py            # Probability caching and management
│       │   ├── confidence_calculator.py         # Confidence score calculation
│       │   └── schemas.py                      # Probability-specific Pydantic models
│       │
│       ├── market_regime/                       # Market Regime Module
│       │   ├── __init__.py
│       │   ├── regime_classifier.py            # Market regime classification logic
│       │   ├── volatility_analyzer.py           # Volatility regime analysis
│       │   ├── trend_analyzer.py               # Trend strength analysis
│       │   └── schemas.py                      # Regime-specific Pydantic models
│       │
│       ├── watchlist_intelligence/             # Watchlist Intelligence Module
│       │   ├── __init__.py
│       │   ├── pre_market_generator.py         # Pre-market watchlist generation
│       │   ├── opportunity_ranker.py           # Opportunity ranking logic
│       │   ├── unusual_activity_detector.py    # Unusual activity detection
│       │   ├── watchlist_manager.py            # Watchlist CRUD operations
│       │   └── schemas.py                      # Watchlist-specific Pydantic models
│       │
│       ├── similarity_search/                  # Similarity Search Module (pgvector)
│       │   ├── __init__.py
│       │   ├── vector_generator.py             # Feature vector generation
│       │   ├── embedding_service.py            # Embedding generation and storage
│       │   ├── similarity_searcher.py          # Vector similarity search logic
│       │   ├── vector_types.py                # Vector type definitions
│       │   └── schemas.py                      # Similarity-specific Pydantic models
│       │
│       ├── feature_generation/                  # Feature Generation Module
│       │   ├── __init__.py
│       │   ├── technical_indicators.py         # ATR, Volume Ratio, Relative Strength
│       │   ├── candlestick_features.py         # Body size, wick ratios, shadows
│       │   ├── momentum_features.py            # Momentum score, trend strength
│       │   ├── context_features.py             # Time of day, day of week
│       │   └── schemas.py                      # Feature-specific Pydantic models
│       │
│       ├── historical_learning/                # Historical Learning Module
│       │   ├── __init__.py
│       │   ├── setup_collector.py              # Ingest setup detection events
│       │   ├── performance_calculator.py       # Aggregate setup performance metrics
│       │   ├── learning_scheduler.py          # Periodic batch processing
│       │   ├── pattern_performance_tracker.py  # Track pattern performance metrics
│       │   └── schemas.py                      # Learning-specific Pydantic models
│       │
│       ├── events/                              # Event Consumption
│       │   ├── __init__.py
│       │   ├── market_data_consumer.py         # BaseStreamConsumer for market.quote
│       │   ├── market_candle_consumer.py       # BaseStreamConsumer for market.candle
│       │   ├── trade_consumer.py               # EventBus consumer for trade.executed
│       │   ├── order_consumer.py               # EventBus consumer for order events
│       │   └── event_publisher.py              # Publish AI events
│       │
│       ├── services/                            # Business Logic Services
│       │   ├── __init__.py
│       │   ├── pattern_service.py              # Pattern detection orchestration
│       │   ├── setup_service.py                # Setup intelligence orchestration
│       │   ├── outcome_service.py              # Outcome evaluation orchestration
│       │   ├── probability_service.py          # Probability calculation orchestration
│       │   ├── regime_service.py               # Market regime orchestration
│       │   ├── watchlist_service.py            # Watchlist intelligence orchestration
│       │   ├── similarity_service.py           # Similarity search orchestration
│       │   └── learning_service.py             # Historical learning orchestration
│       │
│       ├── api/                                 # REST API Endpoints
│       │   ├── __init__.py
│       │   ├── routes_setups.py                # Setup intelligence endpoints
│       │   ├── routes_patterns.py               # Pattern intelligence endpoints
│       │   ├── routes_probabilities.py          # Probability endpoints
│       │   ├── routes_similarity.py            # Similarity search endpoints
│       │   ├── routes_regime.py                # Market regime endpoints
│       │   └── routes_watchlist.py             # Watchlist intelligence endpoints
│       │
│       └── utils/                               # Utilities
│           ├── __init__.py
│           ├── date_utils.py                   # Date/time utilities
│           ├── decimal_utils.py                # Decimal precision handling
│           └── validation_utils.py             # Input validation utilities
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── migrations/
│   └── versions/                               # Alembic migrations
│
├── pyproject.toml                              # UV project configuration
├── README.md
└── CLAUDE.md
```

---

## 2. Module Responsibilities

### main.py
- FastAPI application factory using `smarttrade-common.app_factory.create_app`
- Router registration for all API modules
- Middleware configuration (CORS, rate limiting, request ID)
- Exception handler registration
- Health check endpoints (`/`, `/ready`)

### lifespan.py
- Startup: Initialize database connection, event bus connection, pgvector extension
- Startup: Register event consumers (BaseStreamConsumer, EventBus)
- Startup: Initialize background tasks (learning scheduler, cleanup jobs)
- Shutdown: Graceful connection cleanup, consumer deregistration

### config.py
- Extend `CommonSettings` from `smarttrade-common`
- AMIS-specific configuration (detection thresholds, scoring weights, retention periods)
- pgvector configuration (vector dimensions, similarity metrics)
- Learning pipeline configuration (batch sizes, scheduling intervals)

### models.py
- SQLAlchemy ORM models for all AMIS database tables
- Extend `UUIDMixin`, `TimestampMixin` from `smarttrade-common`
- Use `Decimal` for all monetary values with string serialization
- Define relationships and indexes

### schemas.py
- Pydantic models for REST API requests/responses
- Pydantic models for event schemas (AI events)
- Input validation logic
- Response formatting

### repositories.py
- Data access layer extending `BaseRepository` from `smarttrade-common`
- Custom queries for pattern/setup lookup
- Bulk operations for historical data ingestion
- Vector similarity queries using pgvector

### pattern_detection/
- **candlestick_patterns.py**: Detect candlestick patterns (Doji, Hammer, Engulfing, etc.)
- **price_action_analysis.py**: Detect price action patterns (breakouts, pullbacks, retests)
- **market_structure.py**: Analyze market structure (HH/HL/LH/LL, support/resistance)
- **pattern_detector.py**: Orchestrate all pattern detection pipelines
- **schemas.py**: Pattern-specific Pydantic models

### setup_intelligence/
- **setup_detector.py**: Detect multi-factor setups (pattern + structure + volume + trend)
- **setup_scorer.py**: Calculate setup quality scores (pattern, structure, volume, trend, context)
- **setup_catalog.py**: Manage setup catalog (CRUD, versioning)
- **setup_lifecycle.py**: Manage setup lifecycle (detected → active → expired → evaluated)
- **schemas.py**: Setup-specific Pydantic models

### outcome_evaluation/
- **outcome_tracker.py**: Track setup outcomes from trade execution events
- **return_calculator.py**: Calculate future returns at multiple horizons (1h, 4h, 1d, 1w, 1m)
- **outcome_classifier.py**: Classify outcomes (SUCCESS/FAILURE/NEUTRAL) based on thresholds
- **schemas.py**: Outcome-specific Pydantic models

### probability_engine/
- **statistical_calculator.py**: Calculate statistical probabilities from historical outcomes
- **probability_cache.py**: Cache and manage probability calculations
- **confidence_calculator.py**: Calculate confidence scores based on sample size
- **schemas.py**: Probability-specific Pydantic models

### market_regime/
- **regime_classifier.py**: Classify market regimes using statistical rules
- **volatility_analyzer.py**: Analyze volatility regimes (high/low)
- **trend_analyzer.py**: Analyze trend strength and direction
- **schemas.py**: Regime-specific Pydantic models

### watchlist_intelligence/
- **pre_market_generator.py**: Generate pre-market watchlists based on overnight gaps
- **opportunity_ranker.py**: Rank opportunities based on setup quality and probabilities
- **unusual_activity_detector.py**: Detect unusual volume/price activity
- **watchlist_manager.py**: CRUD operations for watchlists
- **schemas.py**: Watchlist-specific Pydantic models

### similarity_search/
- **vector_generator.py**: Generate feature vectors for patterns/setups/regimes
- **embedding_service.py**: Generate and store embeddings in pgvector
- **similarity_searcher.py**: Perform vector similarity search using pgvector
- **vector_types.py**: Define vector types and dimensions
- **schemas.py**: Similarity-specific Pydantic models

### feature_generation/
- **technical_indicators.py**: Calculate ATR, Volume Ratio, Relative Strength, Gap %, Distance from VWAP/EMA
- **candlestick_features.py**: Calculate body size ratio, wick ratio, shadow ratios
- **momentum_features.py**: Calculate momentum score, trend strength
- **context_features.py**: Extract time of day, day of week
- **schemas.py**: Feature-specific Pydantic models

### historical_learning/
- **setup_collector.py**: Ingest setup detection events into database
- **performance_calculator.py**: Aggregate setup performance metrics (win rates, returns)
- **learning_scheduler.py**: Schedule periodic batch processing jobs
- **pattern_performance_tracker.py**: Track pattern performance by type, market condition, time
- **schemas.py**: Learning-specific Pydantic models

### events/
- **market_data_consumer.py**: BaseStreamConsumer for `market.quote` stream
- **market_candle_consumer.py**: BaseStreamConsumer for `market.candle` stream
- **trade_consumer.py**: EventBus consumer for `trade.executed` events
- **order_consumer.py**: EventBus consumer for order events
- **event_publisher.py**: Publish AI events using DomainEventPublisher

### services/
- **pattern_service.py**: Orchestrate pattern detection and storage
- **setup_service.py**: Orchestrate setup detection, scoring, and lifecycle
- **outcome_service.py**: Orchestrate outcome evaluation and tracking
- **probability_service.py**: Orchestrate probability calculation and caching
- **regime_service.py**: Orchestrate market regime classification
- **watchlist_service.py**: Orchestrate watchlist generation and management
- **similarity_service.py**: Orchestrate similarity search operations
- **learning_service.py**: Orchestrate historical learning pipeline

### api/
- **routes_setups.py**: Setup intelligence REST endpoints
- **routes_patterns.py**: Pattern intelligence REST endpoints
- **routes_probabilities.py**: Probability REST endpoints
- **routes_similarity.py**: Similarity search REST endpoints
- **routes_regime.py**: Market regime REST endpoints
- **routes_watchlist.py**: Watchlist intelligence REST endpoints

---

## 3. Database Entities

### Pattern Table

```python
class Pattern(UUIDMixin, TimestampMixin, table=True):
    """Detected candlestick and price action patterns."""
    __tablename__ = "patterns"
    
    # Identification
    pattern_id: str = Field(unique=True, index=True, nullable=False)
    instrument_id: str = Field(index=True, nullable=False)
    timeframe: str = Field(index=True, nullable=False)
    
    # Pattern classification
    pattern_type: str = Field(index=True, nullable=False)  # DOJI, HAMMER, ENGULFING, etc.
    pattern_direction: str = Field(nullable=False)  # BULLISH, BEARISH, NEUTRAL
    pattern_category: str = Field(index=True, nullable=False)  # CANDLESTICK, PRICE_ACTION, STRUCTURE
    
    # Confidence and quality
    confidence_score: Decimal = Field(max_digits=5, decimal_places=4)
    quality_score: Decimal = Field(max_digits=5, decimal_places=4)
    
    # Pattern parameters (JSON for flexibility)
    parameters: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Market context
    candle_timestamp: datetime = Field(sa_type=DateTime(timezone=True), index=True)
    close_price: Decimal = Field(max_digits=15, decimal_places=4)
    volume: int | None = Field(default=None)
    
    # Metadata
    detection_method: str = Field(default="STATISTICAL")  # STATISTICAL, ML (future)
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_patterns_instrument_timeframe", "instrument_id", "timeframe"),
        Index("ix_patterns_type_direction", "pattern_type", "pattern_direction"),
        Index("ix_patterns_timestamp", "candle_timestamp"),
    )
```

### Setup Table

```python
class Setup(UUIDMixin, TimestampMixin, table=True):
    """Multi-factor setup detections (pattern + structure + volume + trend)."""
    __tablename__ = "setups"
    
    # Identification
    setup_id: str = Field(unique=True, index=True, nullable=False)
    instrument_id: str = Field(index=True, nullable=False)
    timeframe: str = Field(index=True, nullable=False)
    
    # Setup classification
    setup_type: str = Field(index=True, nullable=False)  # BULLISH_CONTINUATION, BREAKOUT_SETUP, etc.
    setup_direction: str = Field(nullable=False)  # LONG, SHORT, NEUTRAL
    
    # Quality scores
    setup_score: Decimal = Field(max_digits=5, decimal_places=4)
    confidence_level: Decimal = Field(max_digits=5, decimal_places=4)
    ranking_score: Decimal = Field(max_digits=5, decimal_places=4)
    
    # Component scores
    pattern_quality_score: Decimal = Field(max_digits=5, decimal_places=4)
    structure_quality_score: Decimal = Field(max_digits=5, decimal_places=4)
    volume_quality_score: Decimal = Field(max_digits=5, decimal_places=4)
    trend_quality_score: Decimal = Field(max_digits=5, decimal_places=4)
    context_quality_score: Decimal = Field(max_digits=5, decimal_places=4)
    
    # Component references (JSON array of pattern IDs)
    components: list[dict] = Field(default_factory=list, sa_column=Column(JSON))
    
    # Market context
    candle_timestamp: datetime = Field(sa_type=DateTime(timezone=True), index=True)
    close_price: Decimal = Field(max_digits=15, decimal_places=4)
    volume: int | None = Field(default=None)
    
    # Market regime at detection
    market_regime: str | None = Field(default=None)
    volatility_regime: str | None = Field(default=None)
    
    # Lifecycle
    lifecycle_status: str = Field(default="ACTIVE", index=True)  # ACTIVE, EXPIRED, EVALUATED
    expiry_timestamp: datetime | None = Field(default=None, sa_type=DateTime(timezone=True))
    
    # Metadata
    detection_method: str = Field(default="STATISTICAL")
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_setups_instrument_timeframe", "instrument_id", "timeframe"),
        Index("ix_setups_type_direction", "setup_type", "setup_direction"),
        Index("ix_setups_lifecycle", "lifecycle_status"),
        Index("ix_setups_timestamp", "candle_timestamp"),
    )
```

### Setup Outcome Table

```python
class SetupOutcome(UUIDMixin, TimestampMixin, table=True):
    """Outcome evaluation for setups."""
    __tablename__ = "setup_outcomes"
    
    # References
    setup_id: str = Field(index=True, nullable=False)
    pattern_id: str | None = Field(default=None, index=True)
    
    # Outcome classification
    outcome_type: str = Field(index=True, nullable=False)  # SUCCESS, FAILURE, NEUTRAL
    outcome_confidence: Decimal = Field(max_digits=5, decimal_places=4)
    
    # Performance metrics (future returns)
    future_return_1h: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    future_return_4h: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    future_return_1d: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    future_return_1w: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    future_return_1m: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    
    # Risk metrics
    drawdown: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    reward_achieved: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    
    # Holding period
    holding_period_minutes: int | None = Field(default=None)
    exit_reason: str | None = Field(default=None)
    
    # Market context at outcome
    outcome_trend: str | None = Field(default=None)
    outcome_volatility: str | None = Field(default=None)
    outcome_regime: str | None = Field(default=None)
    
    # Timestamps
    outcome_timestamp: datetime = Field(sa_type=DateTime(timezone=True), index=True)
    evaluation_horizon: str = Field(default="1d")  # 1h, 4h, 1d, 1w, 1m
    
    # Metadata
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_setup_outcomes_setup", "setup_id"),
        Index("ix_setup_outcomes_type", "outcome_type"),
        Index("ix_setup_outcomes_timestamp", "outcome_timestamp"),
    )
```

### Pattern Outcome Table

```python
class PatternOutcome(UUIDMixin, TimestampMixin, table=True):
    """Outcome evaluation for patterns."""
    __tablename__ = "pattern_outcomes"
    
    # References
    pattern_id: str = Field(index=True, nullable=False)
    
    # Outcome classification
    outcome_type: str = Field(index=True, nullable=False)  # SUCCESS, FAILURE, NEUTRAL
    outcome_confidence: Decimal = Field(max_digits=5, decimal_places=4)
    
    # Performance metrics
    future_return_1h: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    future_return_4h: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    future_return_1d: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    future_return_1w: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    
    # Holding period
    holding_period_minutes: int | None = Field(default=None)
    exit_reason: str | None = Field(default=None)
    
    # Market context at outcome
    outcome_trend: str | None = Field(default=None)
    outcome_volatility: str | None = Field(default=None)
    
    # Timestamps
    outcome_timestamp: datetime = Field(sa_type=DateTime(timezone=True), index=True)
    evaluation_horizon: str = Field(default="1d")
    
    # Metadata
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_pattern_outcomes_pattern", "pattern_id"),
        Index("ix_pattern_outcomes_type", "outcome_type"),
        Index("ix_pattern_outcomes_timestamp", "outcome_timestamp"),
    )
```

### Market Regime Table

```python
class MarketRegime(UUIDMixin, TimestampMixin, table=True):
    """Market regime classifications."""
    __tablename__ = "market_regimes"
    
    # Identification
    instrument_id: str = Field(index=True, nullable=False)
    
    # Regime classification
    regime_type: str = Field(index=True, nullable=False)  # TRENDING_UPTREND, RANGE_BOUND, etc.
    regime_confidence: Decimal = Field(max_digits=5, decimal_places=4, nullable=False)
    
    # Regime metrics
    volatility_level: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    trend_strength: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    range_width: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    
    # Timestamps
    regime_timestamp: datetime = Field(sa_type=DateTime(timezone=True), index=True)
    
    # Metadata
    classification_method: str = Field(default="STATISTICAL")
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_regimes_instrument", "instrument_id"),
        Index("ix_regimes_type", "regime_type"),
        Index("ix_regimes_timestamp", "regime_timestamp"),
    )
```

### Watchlist Table

```python
class Watchlist(UUIDMixin, TimestampMixin, table=True):
    """User watchlists for intelligence-driven monitoring."""
    __tablename__ = "watchlists"
    
    # Ownership
    user_id: UUID = Field(index=True, nullable=False)
    
    # Watchlist classification
    watchlist_type: str = Field(index=True, nullable=False)  # PRE_MARKET, OPPORTUNITIES, UNUSUAL_ACTIVITY
    watchlist_name: str = Field(nullable=False)
    
    # Status
    is_active: bool = Field(default=True, index=True)
    
    # Metadata
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_watchlists_user", "user_id"),
        Index("ix_watchlists_type", "watchlist_type"),
        Index("ix_watchlists_active", "is_active"),
    )
```

### Watchlist Item Table

```python
class WatchlistItem(UUIDMixin, TimestampMixin, table=True):
    """Items within a watchlist."""
    __tablename__ = "watchlist_items"
    
    # References
    watchlist_id: UUID = Field(foreign_key="watchlists.id", index=True, nullable=False)
    instrument_id: str = Field(index=True, nullable=False)
    
    # Ranking and scoring
    score: Decimal = Field(max_digits=5, decimal_places=4, nullable=False)
    rank: int = Field(nullable=False)
    reason: str | None = Field(default=None, max_length=500)
    
    # Metadata
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_watchlist_items_watchlist", "watchlist_id"),
        Index("ix_watchlist_items_instrument", "instrument_id"),
        Index("ix_watchlist_items_score", "score"),
    )
```

### Features Table

```python
class Feature(UUIDMixin, TimestampMixin, table=True):
    """Generated features for patterns and setups."""
    __tablename__ = "features"
    
    # References
    pattern_id: str | None = Field(default=None, index=True)
    setup_id: str | None = Field(default=None, index=True)
    instrument_id: str = Field(index=True, nullable=False)
    timeframe: str = Field(nullable=False)
    
    # Technical indicators
    atr: Decimal | None = Field(default=None, max_digits=15, decimal_places=4)
    volume_ratio: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    relative_strength: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    gap_percentage: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    distance_from_vwap: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    distance_from_ema: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    
    # Candlestick features
    body_size_ratio: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    wick_ratio: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    upper_shadow_ratio: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    lower_shadow_ratio: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    
    # Momentum features
    momentum_score: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    trend_strength: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    
    # Context features
    time_of_day: str | None = Field(default=None)
    day_of_week: str | None = Field(default=None)
    
    # Timestamps
    feature_timestamp: datetime = Field(sa_type=DateTime(timezone=True), index=True)
    
    # Metadata
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_features_pattern", "pattern_id"),
        Index("ix_features_setup", "setup_id"),
        Index("ix_features_instrument_timeframe", "instrument_id", "timeframe"),
    )
```

### Probabilities Table

```python
class Probability(UUIDMixin, TimestampMixin, table=True):
    """Statistical probabilities for setups and patterns."""
    __tablename__ = "probabilities"
    
    # References
    setup_id: str | None = Field(default=None, index=True)
    pattern_id: str | None = Field(default=None, index=True)
    instrument_id: str = Field(index=True, nullable=False)
    timeframe: str = Field(nullable=False)
    
    # Probability estimates
    trend_continuation_prob: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    breakout_success_prob: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    reversal_prob: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    expected_move_prob: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    
    # Risk-reward metrics
    risk_reward_ratio: Decimal | None = Field(default=None, max_digits=10, decimal_places=4)
    confidence_score: Decimal | None = Field(default=None, max_digits=5, decimal_places=4)
    
    # Sample information
    sample_size: int = Field(nullable=False)
    historical_win_rate: Decimal = Field(max_digits=5, decimal_places=4)
    
    # Time horizon
    time_horizon: str = Field(index=True, nullable=False)  # 1h, 4h, 1d, 1w, 1m
    
    # Timestamps
    calculated_at: datetime = Field(sa_type=DateTime(timezone=True), default=func.now())
    valid_until: datetime = Field(sa_type=DateTime(timezone=True), nullable=False)
    
    # Metadata
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_probabilities_setup", "setup_id"),
        Index("ix_probabilities_pattern", "pattern_id"),
        Index("ix_probabilities_instrument_timeframe", "instrument_id", "timeframe"),
        Index("ix_probabilities_horizon", "time_horizon"),
    )
```

### Vector Table (pgvector)

```python
class Vector(UUIDMixin, TimestampMixin, table=True):
    """Vector embeddings for similarity search using pgvector."""
    __tablename__ = "vectors"
    
    # Identification
    vector_id: str = Field(unique=True, index=True, nullable=False)
    vector_type: str = Field(index=True, nullable=False)  # PATTERN, STRUCTURE, SETUP, REGIME
    
    # Vector embedding (pgvector)
    embedding: vector = Field()  # vector(100) - will be configured in migration
    
    # References
    pattern_id: str | None = Field(default=None, index=True)
    setup_id: str | None = Field(default=None, index=True)
    instrument_id: str = Field(index=True, nullable=False)
    timeframe: str | None = Field(default=None)
    
    # Metadata
    metadata: dict = Field(default_factory=dict, sa_column=Column(JSON))
    
    # Indexes
    __table_args__ = (
        Index("ix_vectors_type", "vector_type"),
        Index("ix_vectors_instrument", "instrument_id"),
    )
```

---

## 4. Event Schemas

### ai.setup.detected Event

```python
from pydantic import BaseModel, Field
from datetime import datetime
from uuid import UUID
from decimal import Decimal
from typing import List, Optional

class SetupComponent(BaseModel):
    component_type: str
    component_value: str
    component_score: Decimal

class SetupDetectedEvent(BaseModel):
    event_id: UUID
    topic: str = Field(default="ai.setup.detected")
    timestamp: datetime
    
    instrument_id: str
    timeframe: str  # 1m, 3m, 5m, 15m, 30m, 1h, 4h, 1D, 1W, 1M
    
    setup_type: str  # BULLISH_CONTINUATION, BEARISH_CONTINUATION, etc.
    setup_direction: str  # LONG, SHORT, NEUTRAL
    
    setup_score: Decimal
    confidence_level: Optional[Decimal] = None
    ranking_score: Optional[Decimal] = None
    
    components: List[SetupComponent]
    
    pattern_quality_score: Optional[Decimal] = None
    structure_quality_score: Optional[Decimal] = None
    volume_quality_score: Optional[Decimal] = None
    trend_quality_score: Optional[Decimal] = None
    context_quality_score: Optional[Decimal] = None
    
    setup_id: str
    candle_timestamp: datetime
```

### ai.setup.scored Event

```python
class QualityBreakdown(BaseModel):
    pattern_quality: Decimal
    structure_quality: Decimal
    volume_quality: Decimal
    trend_quality: Decimal
    context_quality: Decimal

class SetupScoredEvent(BaseModel):
    event_id: UUID
    topic: str = Field(default="ai.setup.scored")
    timestamp: datetime
    
    setup_id: str
    setup_score: Decimal
    quality_breakdown: QualityBreakdown
    
    confidence_level: Optional[Decimal] = None
    ranking_score: Optional[Decimal] = None
```

### ai.outcome.evaluated Event

```python
class FutureReturns(BaseModel):
    return_1h: Optional[Decimal] = None
    return_4h: Optional[Decimal] = None
    return_1d: Optional[Decimal] = None
    return_1w: Optional[Decimal] = None
    return_1m: Optional[Decimal] = None

class OutcomeEvaluatedEvent(BaseModel):
    event_id: UUID
    topic: str = Field(default="ai.outcome.evaluated")
    timestamp: datetime
    
    setup_id: str
    pattern_id: Optional[str] = None
    
    outcome_type: str  # SUCCESS, FAILURE, NEUTRAL
    evaluation_horizon: str  # 1h, 4h, 1d, 1w, 1m
    
    future_returns: FutureReturns
    drawdown: Optional[Decimal] = None
    reward_achieved: Optional[Decimal] = None
    
    holding_period_minutes: Optional[int] = None
    exit_reason: Optional[str] = None
```

### ai.regime.classified Event

```python
class RegimeClassifiedEvent(BaseModel):
    event_id: UUID
    topic: str = Field(default="ai.regime.classified")
    timestamp: datetime
    
    instrument_id: str
    regime_type: str  # TRENDING_UPTREND, RANGE_BOUND, BREAKOUT_DAY, etc.
    regime_confidence: Decimal
    
    volatility_level: Optional[Decimal] = None
    trend_strength: Optional[Decimal] = None
```

### ai.watchlist.generated Event

```python
class WatchlistItem(BaseModel):
    instrument_id: str
    score: Decimal
    rank: int
    reason: Optional[str] = None

class WatchlistGeneratedEvent(BaseModel):
    event_id: UUID
    topic: str = Field(default="ai.watchlist.generated")
    timestamp: datetime
    
    user_id: UUID
    watchlist_type: str  # PRE_MARKET, OPPORTUNITIES, UNUSUAL_ACTIVITY
    watchlist_name: str
    
    items: List[WatchlistItem]
```

### ai.probability.generated Event

```python
class ProbabilityGeneratedEvent(BaseModel):
    event_id: UUID
    topic: str = Field(default="ai.probability.generated")
    timestamp: datetime
    
    setup_id: Optional[str] = None
    pattern_id: Optional[str] = None
    instrument_id: str
    timeframe: str
    
    trend_continuation_prob: Optional[Decimal] = None
    breakout_success_prob: Optional[Decimal] = None
    reversal_prob: Optional[Decimal] = None
    expected_move_prob: Optional[Decimal] = None
    
    risk_reward_ratio: Optional[Decimal] = None
    confidence_score: Optional[Decimal] = None
    
    sample_size: int
    historical_win_rate: Decimal
    
    time_horizon: str
```

### ai.similarity.found Event

```python
class SimilarityResult(BaseModel):
    setup_id: str
    pattern_id: Optional[str] = None
    similarity_score: Decimal
    outcome_type: Optional[str] = None
    future_return: Optional[Decimal] = None

class SimilarityFoundEvent(BaseModel):
    event_id: UUID
    topic: str = Field(default="ai.similarity.found")
    timestamp: datetime
    
    query_id: str
    query_setup_id: Optional[str] = None
    query_pattern_id: Optional[str] = None
    
    results: List[SimilarityResult]
    total_results: int
```

---

## 5. Configuration

### config.py

```python
from pydantic import Field
from smarttrade_common.config import CommonSettings

class AMISSettings(CommonSettings):
    """AI Market Intelligence Service configuration."""
    
    # Service identification
    service_name: str = "ai-market-intelligence-service"
    
    # Database
    database_url: str = Field(..., description="PostgreSQL database URL with pgvector")
    
    # Pattern detection thresholds
    pattern_confidence_threshold: float = Field(default=0.6, ge=0.0, le=1.0)
    pattern_quality_threshold: float = Field(default=0.5, ge=0.0, le=1.0)
    
    # Setup detection thresholds
    setup_score_threshold: float = Field(default=0.7, ge=0.0, le=1.0)
    setup_confidence_threshold: float = Field(default=0.6, ge=0.0, le=1.0)
    
    # Setup scoring weights
    pattern_quality_weight: float = Field(default=0.3, ge=0.0, le=1.0)
    structure_quality_weight: float = Field(default=0.25, ge=0.0, le=1.0)
    volume_quality_weight: float = Field(default=0.2, ge=0.0, le=1.0)
    trend_quality_weight: float = Field(default=0.15, ge=0.0, le=1.0)
    context_quality_weight: float = Field(default=0.1, ge=0.0, le=1.0)
    
    # Outcome evaluation thresholds
    success_threshold: float = Field(default=0.5, ge=0.0)  # % return for SUCCESS
    failure_threshold: float = Field(default=-0.3, le=0.0)  # % return for FAILURE
    
    # Probability calculation
    min_sample_size: int = Field(default=30, ge=1)
    probability_cache_ttl_hours: int = Field(default=24, ge=1)
    
    # Similarity search
    vector_dimensions: int = Field(default=100, ge=1)
    similarity_threshold: float = Field(default=0.7, ge=0.0, le=1.0)
    max_similar_results: int = Field(default=10, ge=1, le=100)
    
    # Historical learning
    data_retention_days: int = Field(default=730, ge=1)  # 2 years
    performance_aggregation_interval_hours: int = Field(default=24, ge=1)
    
    # Watchlist intelligence
    watchlist_size_limit: int = Field(default=50, ge=1, le=200)
    unusual_activity_volume_threshold: float = Field(default=2.0, ge=1.0)  # 2x average
    
    # Market regime classification
    regime_classification_interval_minutes: int = Field(default=15, ge=1)
    volatility_high_threshold: float = Field(default=2.0, ge=0.0)  # % of price
    volatility_low_threshold: float = Field(default=0.5, ge=0.0)  # % of price
    
    # Background tasks
    learning_job_interval_hours: int = Field(default=6, ge=1)
    cleanup_job_interval_days: int = Field(default=1, ge=1)
    
    # pgvector configuration
    pgvector_index_type: str = Field(default="ivfflat")
    pgvector_similarity_metric: str = Field(default="cosine")
```

---

## 6. Implementation Phases

### Phase 1: Foundation (Week 1-2)
- Service skeleton with FastAPI + smarttrade-common
- Database schema (PostgreSQL + pgvector extension)
- Event bus integration (Redis)
- Configuration management
- Basic API structure

### Phase 2: Pattern Detection (Week 3-4)
- Candlestick pattern detection
- Price action pattern detection
- Market structure analysis
- Pattern database storage
- Pattern event publishing

### Phase 3: Setup Intelligence (Week 5-6)
- Setup detection logic
- Setup quality scoring
- Setup catalog management
- Setup lifecycle management
- Setup event publishing

### Phase 4: Outcome Evaluation (Week 7)
- Outcome tracking from trade events
- Future return calculation
- Outcome classification
- Outcome event publishing

### Phase 5: Probability Engine (Week 8)
- Statistical probability calculation
- Confidence score calculation
- Probability caching
- Probability event publishing

### Phase 6: Market Regime (Week 9)
- Market regime classification
- Volatility analysis
- Trend analysis
- Regime event publishing

### Phase 7: Similarity Search (Week 10-11)
- Feature vector generation
- pgvector integration
- Similarity search logic
- Similarity event publishing

### Phase 8: Watchlist Intelligence (Week 12)
- Pre-market watchlist generation
- Opportunity ranking
- Unusual activity detection
- Watchlist event publishing

### Phase 9: Historical Learning (Week 13-14)
- Setup collection pipeline
- Performance aggregation
- Learning scheduler
- Pattern performance tracking

---

## 7. Success Criteria

### Phase 1 (Statistical Learning Engine) Success Criteria

1. **Pattern Detection**: Detect candlestick, price action, and market structure patterns with >80% accuracy
2. **Setup Intelligence**: Detect multi-factor setups with quality scoring and lifecycle management
3. **Outcome Evaluation**: Track setup outcomes from trade events with >95% accuracy
4. **Probability Engine**: Calculate statistical probabilities from historical data with confidence intervals
5. **Market Regime**: Classify market regimes using statistical rules with >85% accuracy
6. **Historical Learning**: Aggregate setup performance metrics with 2+ year data retention
7. **Event-Driven Architecture**: All intelligence published via events for downstream consumption
8. **Stateless Design**: No runtime state persistence, all intelligence derived from historical data
9. **pgvector Integration**: Vector similarity search operational for pattern/setup/regime similarity
10. **API Availability**: All REST endpoints operational with proper authentication and authorization

---

## 8. Dependencies

### Required Dependencies (pyproject.toml)

```toml
[project]
name = "ai-market-intelligence-service"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "smarttrade-common",
    "fastapi",
    "uvicorn[standard]",
    "sqlalchemy[asyncio]",
    "asyncpg",
    "alembic",
    "pydantic",
    "pydantic-settings",
    "redis",
    "numpy",
    "pandas",
    "pgvector",  # PostgreSQL vector extension
]

[project.optional-dependencies]
dev = [
    "pytest",
    "pytest-asyncio",
    "pytest-cov",
    "ruff",
    "mypy",
]
```

### smarttrade-common Dependencies

| Component | Import |
|---|---|
| Event subscription | `from smarttrade_common.events import consume_event, BaseStreamConsumer, get_event_payload` |
| Event binding | `from smarttrade_common.events.event_bus import bind_registered_subscribers` |
| Event publishing | `from smarttrade_common.events.event_bus import DomainEventPublisher` |
| DB session | `from smarttrade_common.database.session import get_session, get_database_session` |
| Repository base | `from smarttrade_common.database.repository import BaseRepository` |
| Model mixins | `from smarttrade_common.database.models import UUIDMixin, TimestampMixin` |
| App factory | `from smarttrade_common.app_factory import create_app` |
| Lifespan | `from smarttrade_common.lifespan import lifespan as common_lifespan` |
| Auth | `from smarttrade_common.auth import get_current_user` |
| Logging | `from smarttrade_common.logging import init_logging` |
| HTTP client | `from smarttrade_common.http_client import BaseServiceClient` |

---

## 9. Error Handling

### Error Codes

| Error Code | Description | HTTP Status |
|------------|-------------|-------------|
| AMIS_001 | Pattern detection failed | 500 |
| AMIS_002 | Setup detection failed | 500 |
| AMIS_003 | Outcome tracking failed | 500 |
| AMIS_004 | Probability calculation failed | 500 |
| AMIS_005 | Regime classification failed | 500 |
| AMIS_006 | Similarity search failed | 500 |
| AMIS_007 | Vector generation failed | 500 |
| AMIS_008 | Watchlist generation failed | 500 |
| AMIS_009 | Database operation failed | 500 |
| AMIS_010 | Event publishing failed | 500 |
| AMIS_011 | Invalid input parameters | 400 |
| AMIS_012 | Resource not found | 404 |
| AMIS_013 | Unauthorized access | 401 |
| AMIS_014 | Rate limit exceeded | 429 |

### Error Handling Strategy

- Use `SmartTradeError` from `smarttrade-common` for all business errors
- Implement retry logic for transient failures (database, event bus)
- Log all errors with context (instrument_id, timeframe, setup_id)
- Return user-friendly error messages via API
- Implement circuit breaker for external service calls

---

## 10. Monitoring and Observability

### Metrics

- Pattern detection rate (by type, instrument, timeframe)
- Setup detection rate (by type, instrument, timeframe)
- Outcome tracking success rate
- Probability calculation latency
- Similarity search latency
- Vector database query latency
- Event publishing success rate
- API endpoint latency and error rates

### Logging

- Structured logging with JSON format
- Log all pattern detections with metadata
- Log all setup detections with quality scores
- Log all outcome evaluations with results
- Log all probability calculations with sample sizes
- Log all similarity searches with results
- Log all errors with stack traces and context

### Tracing

- OpenTelemetry integration for distributed tracing
- Trace pattern detection pipeline
- Trace setup detection pipeline
- Trace outcome evaluation pipeline
- Trace probability calculation pipeline
- Trace similarity search pipeline

---

## 11. Security Considerations

### Authentication and Authorization

- Use JWT authentication via `smarttrade-common.auth`
- RBAC for API endpoints (read/write permissions)
- Service-to-service authentication via service tokens

### Data Protection

- Encrypt sensitive data at rest (database encryption)
- Use TLS for all network communication
- Implement data retention policies
- Anonymize user data in analytics

### Input Validation

- Validate all API inputs using Pydantic schemas
- Sanitize database queries to prevent SQL injection
- Validate event payloads before processing
- Implement rate limiting to prevent abuse

---

## 12. Performance Considerations

### Database Optimization

- Use appropriate indexes for frequent queries
- Partition historical data by date
- Use connection pooling for database connections
- Implement query result caching where appropriate

### Vector Database Optimization

- Use ivfflat index for pgvector
- Tune vector dimensions for performance vs accuracy
- Implement vector result caching
- Batch vector operations where possible

### Event Processing

- Use batch processing for high-volume events
- Implement event throttling to prevent overload
- Use async processing for all event handlers
- Implement backpressure handling

### API Performance

- Implement response caching for read-heavy endpoints
- Use pagination for list endpoints
- Implement request timeout handling
- Use connection pooling for HTTP clients

---

## 13. Deployment Considerations

### Database Migration

- Use Alembic for database schema migrations
- Test migrations on staging environment first
- Implement rollback procedures for migrations
- Backup database before migrations

### Service Deployment

- Containerize service using Docker
- Use environment variables for configuration
- Implement health checks for container orchestration
- Use rolling deployments for zero-downtime updates

### pgvector Setup

- Install pgvector extension in PostgreSQL
- Configure vector index type and similarity metric
- Monitor vector database performance
- Plan for future migration to Qdrant if needed

---

## 14. Testing Strategy

### Unit Tests

**Pattern Detection Module**:
- Test candlestick pattern detection logic
- Test price action pattern detection logic
- Test market structure analysis logic
- Test pattern scoring and quality calculation

**Setup Intelligence Module**:
- Test setup detection orchestration
- Test setup quality scoring
- Test setup lifecycle management
- Test setup catalog operations

**Outcome Evaluation Module**:
- Test future return calculation
- Test outcome classification
- Test outcome tracking from trade events

**Probability Engine Module**:
- Test statistical probability calculations
- Test confidence score calculation
- Test probability caching logic

**Market Regime Module**:
- Test regime classification logic
- Test volatility analysis
- Test trend analysis

**Similarity Search Module**:
- Test vector generation
- Test pgvector similarity search
- Test embedding storage and retrieval

### Integration Tests

**Event Consumption**:
- Test market data quote consumption
- Test market data candle consumption
- Test trade event consumption
- Test event publishing

**Database Operations**:
- Test pattern storage and retrieval
- Test setup storage and retrieval
- Test outcome tracking
- Test probability caching
- Test vector operations with pgvector

**Service Integration**:
- Test pattern service orchestration
- Test setup service orchestration
- Test outcome service orchestration
- Test probability service orchestration

### E2E Tests

**End-to-End Workflows**:
- Test market data → pattern detection → setup detection → outcome tracking
- Test setup detection → probability calculation → similarity search
- Test market regime classification → watchlist generation
- Test historical learning pipeline

---

## 15. Conclusion

This LLD provides a comprehensive technical specification for the AI Market Intelligence Service (AMIS) Phase 1 implementation, focusing on statistical learning capabilities without ML dependencies. The design follows SmartTrade architectural principles:

- **Stateless Architecture**: No runtime state persistence, all intelligence derived from historical data
- **Event-Driven Communication**: All intelligence published via events for downstream consumption
- **smarttrade-common Integration**: Leverages shared library for consistency across services
- **Production-Grade Quality**: Follows SOLID principles, proper error handling, and comprehensive testing
- **Future-Proof Design**: Architecture supports future ML phases (XGBoost, LSTM, Transformer models)

The service is designed to be the central intelligence and learning engine for SmartTrade, providing setup intelligence, pattern detection, probability generation, and market regime classification to support downstream decision-making in the Strategy Service and other consuming services.
