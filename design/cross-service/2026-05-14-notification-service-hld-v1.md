# Notification Service High-Level Design (HLD)

**Version**: 2.0
**Date**: 2026-05-14
**Status**: Design (Refactored)
**Services Affected**: notification-service (new), broker-adapter-service (event publisher), market-data-service (event publisher), authentication-service (user context)

---

## Executive Summary

### What
The Notification Service is a new microservice that provides asynchronous, event-driven notification delivery to SmartTrade users. It consumes domain events from the event bus, transforms them into user notifications based on user preferences, and delivers them through multiple channels (WebSocket UI, with future support for SMS, Email, and Mobile Push).

### Why
- **User Engagement**: Real-time notifications improve trading experience by keeping users informed about order fills, position updates, risk events, and system alerts
- **Flexibility**: Channel abstraction allows adding new delivery mechanisms (SMS, Email, Push) without core service changes
- **Scalability**: Asynchronous event-driven architecture handles high notification volumes without blocking trading operations
- **Reliability**: Replay support ensures users never miss notifications, even during disconnections
- **Separation of Concerns**: Notification logic is isolated from trading logic, following microservice best practices
- **Architectural Alignment**: Unified event ingestion, publisher-owned severity, wildcard subscriptions, and template-based messaging align with SmartTrade platform standards

### Which Services
- **notification-service** (NEW): Event consumer, notification processor, WebSocket delivery, preference management
- **broker-adapter-service**: Continues publishing existing domain events with severity in EventEnvelope
- **market-data-service**: Publishes broker.* and system.* events with severity in EventEnvelope
- **authentication-service**: Provides user context for JWT validation and user preferences

### Success Criteria
1. Users can subscribe to domain event patterns (wildcards supported) via REST API
2. Notifications are delivered in real-time via WebSocket with < 100ms latency
3. Users can replay missed notifications from any sequence number (automatic on reconnect)
4. Notification preferences are persisted and enforced with per-channel configuration
5. Severity classification is publisher-owned in EventEnvelope (single source of truth)
6. Channel abstraction allows adding SMS/Email/Push without core changes
7. At-least-once delivery guarantee with idempotency handling
8. Support for 10,000+ concurrent WebSocket connections
9. 99.9% notification delivery success rate for UI channel
10. Comprehensive monitoring and alerting for delivery failures
11. Rate limiting prevents event storms (100/min, 1000/hour per user/event type)
12. Template-based messaging enables message changes without code deployment
13. Centralized event schemas in smarttrade-common (single source of truth)
14. Subscription patterns validated against EventCatalog
15. Event catalog provides centralized event discovery and documentation

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    smarttrade-common (Event Schemas)                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Centralized Event Catalog                          │  │
│  │  events/schemas/domain/                                               │  │
│  │  ├── trading/         (OrderPlacedV1, OrderFilledV1, ...)             │  │
│  │  ├── risk/            (RiskLimitReachedV1, ...)                       │  │
│  │  ├── broker/          (BrokerConnectionFailedV1, ...)                 │  │
│  │  ├── system/          (SystemEmergencyV1, ...)                         │  │
│  │  ├── ai/              (AISignalGeneratedV1, ...)                       │  │
│  │  └── notification/    (NotificationCreatedV1, ...)                    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Import schemas
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SmartTrade Event Bus (Redis)                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    All Domain Events (with severity)                  │  │
│  │  order.filled (severity: SUCCESS)                                  │  │
│  │  trade.executed (severity: SUCCESS)                                │  │
│  │  position.updated (severity: INFO)                                │  │
│  │  risk.limit_reached.v1 (severity: CRITICAL)                           │  │
│  │  broker.connection.failed.v1 (severity: ERROR)                       │  │
│  │  system.emergency.v1 (severity: CRITICAL)                             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ @subscribe('*')
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Notification Service                                 │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Unified Event Consumer                           │  │
│  │  notification_event_consumer.py                                      │  │
│  │  @subscribe('*') - Single consumer for all events                    │  │
│  │  Internal delegation to event handlers                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │              Rate Limiter (user_id, event_type)                      │  │
│  │  100/min, 1000/hour per user per event type                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Notification Processing Pipeline                   │  │
│  │  1. Extract user_id, severity from event envelope                    │  │
│  │  2. Check user subscription (wildcard pattern matching)               │  │
│  │  3. Determine category (TRADING, RISK, SYSTEM, BROKER, AI)            │  │
│  │  4. Generate notification message (template registry)                │  │
│  │  5. Create notification record (BIGSERIAL sequence_number)            │  │
│  │  6. Publish notification.created.v1 event                            │  │
│  │  7. Trigger delivery for each subscribed channel                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         Channel Abstraction                          │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │  │
│  │  │ UIChannel    │  │ SMSChannel   │  │ EmailChannel │              │  │
│  │  │ (WebSocket)  │  │ (Future)     │  │ (Future)     │              │  │
│  │  │ No log       │  │ Log delivery │  │ Log delivery │              │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │  │
│  │  ┌──────────────┐                                                   │  │
│  │  │ PushChannel  │                                                   │  │
│  │  │ (Future)     │                                                   │  │
│  │  │ Log delivery │                                                   │  │
│  │  └──────────────┘                                                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      WebSocket Manager                                │  │
│  │  - Connection management per user_id                                   │  │
│  │  - Automatic replay on connect (?last_seq=12345)                      │  │
│  │  - Heartbeat (5s interval)                                           │  │
│  │  - Live stream after replay                                          │  │
│  │  - Idempotent connection handling                                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         Database (PostgreSQL)                         │  │
│  │  ┌──────────────────┐  ┌──────────────────────────────────┐          │  │
│  │  │ notification_     │  │ notification_subscription_channels│          │  │
│  │  │ subscriptions    │  │ (subscription_id, channel, config) │          │  │
│  │  │ (event_pattern)  │  └──────────────────────────────────┘          │  │
│  │  └──────────────────┘  ┌──────────────────┐                         │  │
│  │  ┌──────────────────┐  │ notification_    │                         │  │
│  │  │ notification_     │  │ delivery_log     │                         │  │
│  │  │ messages         │  │ (SMS/Email/Push) │                         │  │
│  │  │ (BIGSERIAL seq)  │  │ UI not logged    │                         │  │
│  │  └──────────────────┘  └──────────────────┘                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         Template Registry                             │  │
│  │  templates/                                                              │  │
│  │  - order_filled.jinja2                                                 │  │
│  │  - risk_limit.jinja2                                                   │  │
│  │  - broker_failure.jinja2                                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         REST API                                       │  │
│  │  POST   /api/v1/notifications/subscriptions                         │  │
│  │  GET    /api/v1/notifications/subscriptions                         │  │
│  │  PUT    /api/v1/notifications/subscriptions/{id}                    │  │
│  │  DELETE /api/v1/notifications/subscriptions/{id}                    │  │
│  │  GET    /api/v1/notifications                                       │  │
│  │  GET    /api/v1/notifications/{id}                                  │  │
│  │  PUT    /api/v1/notifications/{id}/read                             │  │
│  │  GET    /api/v1/notifications/replay                                 │  │
│  │  WS     /api/v1/ws/notifications?last_seq=12345                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Frontend (React)                                  │
│  - WebSocket connection with automatic replay                             │
│  - Notification center UI with category filtering                        │
│  - Subscription preferences UI (wildcard patterns)                       │
│  - Per-channel configuration (phone, email, device tokens)                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Centralized Event Definitions in smarttrade-common

### Overview

All event schema definitions are centralized in `smarttrade-common` to provide:
- **Single source of truth** for all event contracts across the platform
- **Notification service** access to event schemas for subscription validation and template generation
- **Consistency** across all services (no duplicate event definitions)
- **Centralized event registry** for discovery and documentation
- **Type safety** through shared Pydantic models

### Directory Structure

```
smarttrade-common/
  src/smarttrade_common/
    events/
      __init__.py
      event_bus.py              # Existing: EventBus implementation
      publisher.py              # Existing: DomainEventPublisher
      schemas/
        __init__.py             # Existing: SchemaRegistry
        registry.py             # Existing: SchemaRegistry implementation
        domain/                 # NEW: Centralized domain event schemas
          __init__.py
          trading/
            __init__.py
            order_events.py     # OrderPlacedV1, OrderFilledV1, etc.
            trade_events.py     # TradeExecutedV1, etc.
            position_events.py  # PositionUpdatedV1, etc.
          risk/
            __init__.py
            risk_events.py      # RiskWarningV1, RiskLimitReachedV1, etc.
          broker/
            __init__.py
            broker_events.py    # BrokerConnectionFailedV1, etc.
          system/
            __init__.py
            system_events.py    # SystemEmergencyV1, etc.
          ai/
            __init__.py
            ai_events.py        # AISignalGeneratedV1, etc.
          notification/
            __init__.py
            notification_events.py  # NotificationCreatedV1, etc.
        event_catalog.py        # NEW: Central event catalog and metadata
```

### Event Schema Definition Pattern

**Event Schema Example** (`order_events.py`):
```python
from pydantic import BaseModel, Field
from smarttrade_common.events.schemas.domain.event_base import DomainEventBase

class OrderPlacedV1(DomainEventBase):
    """Schema for order.placed event."""
    
    event_type: str = "order.placed"
    version: str = "v1"
    
    # Event-specific fields
    broker_id: str
    account_id: str
    client_order_id: str
    instrument_id: str
    side: str  # BUY|SELL
    quantity: int
    price: str | None  # String for precision
    order_type: str
```

**Event Base Class** (`event_base.py`):
```python
from pydantic import BaseModel, Field
from datetime import datetime
from uuid import UUID

class DomainEventBase(BaseModel):
    """Base class for all domain events with common envelope fields."""
    
    # Envelope fields (auto-populated by DomainEventPublisher)
    event_id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    event_name: str
    event_version: str = "1.0"
    user_id: str
    trace_id: str
    request_id: str
    idempotency_key: str
    timestamp: str = Field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
    
    # Publisher-provided metadata
    severity: str = Field(default="INFO")  # INFO, SUCCESS, WARNING, ERROR, CRITICAL
    category: str = Field(default="SYSTEM")  # TRADING, RISK, SYSTEM, BROKER, AI
    
    # Event-specific payload
    payload: dict
```

### Event Catalog

**Event Catalog** (`event_catalog.py`):
```python
from typing import Dict, List
from smarttrade_common.events.schemas.domain.trading.order_events import OrderPlacedV1, OrderFilledV1
from smarttrade_common.events.schemas.domain.risk.risk_events import RiskLimitReachedV1
# ... import all event schemas

class EventCatalog:
    """Central catalog of all domain events with metadata."""
    
    EVENTS: Dict[str, dict] = {
        "order.placed": {
            "schema": OrderPlacedV1,
            "owner": "broker-adapter-service",
            "category": "TRADING",
            "default_severity": "INFO",
            "description": "Emitted when an order is placed with the broker",
        },
        "order.filled": {
            "schema": OrderFilledV1,
            "owner": "broker-adapter-service",
            "category": "TRADING",
            "default_severity": "SUCCESS",
            "description": "Emitted when an order is filled (full or partial)",
        },
        "risk.limit_reached.v1": {
            "schema": RiskLimitReachedV1,
            "owner": "broker-adapter-service",
            "category": "RISK",
            "default_severity": "CRITICAL",
            "description": "Emitted when a risk limit is reached",
        },
        # ... all events
    }
    
    @classmethod
    def get_event_schema(cls, event_name: str):
        """Get event schema class by name."""
        if event_name not in cls.EVENTS:
            raise ValueError(f"Unknown event: {event_name}")
        return cls.EVENTS[event_name]["schema"]
    
    @classmethod
    def get_all_events(cls) -> List[str]:
        """Get list of all registered event names."""
        return list(cls.EVENTS.keys())
    
    @classmethod
    def get_events_by_category(cls, category: str) -> List[str]:
        """Get list of events by category."""
        return [
            event_name for event_name, meta in cls.EVENTS.items()
            if meta["category"] == category
        ]
    
    @classmethod
    def get_events_by_owner(cls, owner: str) -> List[str]:
        """Get list of events by owner service."""
        return [
            event_name for event_name, meta in cls.EVENTS.items()
            if meta["owner"] == owner
        ]
    
    @classmethod
    def validate_event_pattern(cls, pattern: str) -> bool:
        """Validate that event pattern matches at least one known event."""
        from fnmatch import fnmatch
        return any(fnmatch(event_name, pattern) for event_name in cls.EVENTS.keys())
```

### Service Usage Patterns

**Publisher Services** (e.g., broker-adapter-service):
```python
from smarttrade_common.events.publisher import DomainEventPublisher
from smarttrade_common.events.schemas.domain.trading.order_events import OrderPlacedV1

publisher = DomainEventPublisher.get_instance()

# Create event data using schema
event_data = OrderPlacedV1(
    broker_id="fyers",
    account_id="ACC123",
    client_order_id="CLIENT-001",
    instrument_id="NIFTY50-INDEX",
    side="BUY",
    quantity=10,
    price="21000.50",
    order_type="LIMIT",
).model_dump()

# Prepare and publish
prepared = publisher.prepare_event(
    event_name="order.placed",
    event_data=event_data,
    idempotency_key=f"order-{client_order_id}",
    metadata={
        "severity": "INFO",  # Publisher can override default
    }
)
await event_bus.publish(prepared.event_name, prepared.to_dict())
```

**Notification Service**:
```python
from smarttrade_common.events.schemas.event_catalog import EventCatalog
from fnmatch import fnmatch

# Validate subscription pattern
if not EventCatalog.validate_event_pattern("order.*"):
    raise ValueError("Invalid event pattern")

# Get event metadata
event_meta = EventCatalog.EVENTS.get("order.filled")
category = event_meta["category"]
default_severity = event_meta["default_severity"]

# Match subscription patterns
subscriptions = await subscription_repo.get_user_subscriptions(user_id)
for sub in subscriptions:
    if fnmatch(event_name, sub.event_pattern):
        # Process notification
        pass
```

### Schema Registration

**Automatic Registration at Startup**:
```python
# In smarttrade-common lifespan.py
from smarttrade_common.events.schemas.registry import SchemaRegistry
from smarttrade_common.events.schemas.event_catalog import EventCatalog

def register_all_event_schemas(registry: SchemaRegistry):
    """Register all event schemas from catalog."""
    for event_name, meta in EventCatalog.EVENTS.items():
        registry.register(
            event_name=event_name,
            schema=meta["schema"],
            owner_service=meta["owner"],
        )
    registry.freeze()
```

### Subscription Validation

**Notification Service Subscription Validation**:
```python
from smarttrade_common.events.schemas.event_catalog import EventCatalog

async def create_subscription(user_id: str, event_pattern: str, channels: list):
    # Validate pattern matches at least one known event
    if not EventCatalog.validate_event_pattern(event_pattern):
        raise ValueError(f"Invalid event pattern: {event_pattern}")
    
    # Validate channels
    for channel in channels:
        if channel not in ["UI", "SMS", "EMAIL", "MOBILE_PUSH"]:
            raise ValueError(f"Invalid channel: {channel}")
    
    # Create subscription
    await subscription_repo.create(...)
```

### Template Integration

**Template Registry with Event Catalog**:
```python
from smarttrade_common.events.schemas.event_catalog import EventCatalog

class TemplateService:
    def __init__(self):
        # Auto-discover templates based on event catalog
        for event_name in EventCatalog.get_all_events():
            template_file = f"templates/{event_name.replace('.', '_')}.jinja2"
            self.load_template(template_file)
```

### Migration Path

**Phase 1: Add Event Catalog to smarttrade-common** (Week 1)
1. Create `smarttrade-common/src/smarttrade_common/events/schemas/domain/` directory structure
2. Move existing event schemas from services to smarttrade-common
3. Create `EventCatalog` class
4. Update `SchemaRegistry` to auto-register from catalog
5. Add validation utilities

**Phase 2: Migrate Publisher Services** (Week 2)
1. Update broker-adapter-service to import event schemas from smarttrade-common
2. Update market-data-service to import event schemas from smarttrade-common
3. Remove local event schema definitions
4. Test event publishing with imported schemas
5. Deploy publisher updates

**Phase 3: Update Notification Service** (Week 3)
1. Import event schemas from smarttrade-common
2. Use EventCatalog for subscription validation
3. Use EventCatalog for template discovery
4. Use EventCatalog for category mapping
5. Test subscription validation with catalog

**Phase 4: Remove Local Schemas** (Week 4)
1. Remove local event schema files from publisher services
2. Remove local event schema files from notification service
3. Update all imports to use smarttrade-common
4. Clean up deprecated code

**Backward Compatibility**:
- During migration, services can use both local and centralized schemas
- SchemaRegistry supports registration from both sources
- Gradual migration to avoid breaking changes
- Deprecation warnings for local schema usage

### Benefits

1. **Single Source of Truth**: All event definitions in one place
2. **Type Safety**: Shared Pydantic models ensure consistency
3. **Discovery**: EventCatalog provides centralized event discovery
4. **Validation**: Subscription patterns validated against known events
5. **Documentation**: Event catalog includes descriptions and metadata
6. **Reduced Duplication**: No duplicate event definitions across services
7. **Easier Onboarding**: New services can import event schemas directly
8. **Versioning**: Centralized versioning of event schemas
9. **Testing**: Shared schemas enable cross-service event testing
10. **Refactoring**: Event schema changes propagate to all services

---

## Service-Specific Sections

### Notification Service (NEW)

**Port**: 8003
**Database**: `smarttrade_notification_service` (PostgreSQL)
**Status**: Greenfield implementation

#### Responsibilities
1. **Unified Event Consumption**: Single `@subscribe('*')` consumer receives all events, delegates internally
2. **Subscription Management**: Allow users to subscribe to event patterns (wildcards supported) per channel
3. **Notification Processing**: Transform events into notifications using template registry
4. **Severity Inheritance**: Read severity from EventEnvelope (publisher-owned)
5. **Channel Delivery**: Deliver notifications via WebSocket (current), SMS/Email/Push (future)
6. **Preference Storage**: Persist user notification preferences with per-channel configuration
7. **Notification History**: Store all notifications with read/unread state and category
8. **Replay Support**: Automatic replay on WebSocket reconnect via query parameter
9. **Rate Limiting**: Prevent event storms (100/min, 1000/hour per user/event type)
10. **Retention Policy**: Hot storage 30d, warm storage 90d, delete after 180d

#### Non-Responsibilities (Strict Boundaries)
- ❌ **No order execution** - Does not execute, modify, or cancel orders
- ❌ **No strategy logic** - Does not make trading decisions or run strategies
- ❌ **No risk calculations** - Does not compute risk metrics or enforce limits
- ❌ **No market data ownership** - Does not generate or own market data
- ❌ **No business decision making** - Does not determine if notifications should be sent (user preferences only)
- ❌ **No synchronous calls to BAS** - Does not call broker-adapter-service synchronously
- ❌ **No severity classification** - Severity is publisher-owned in EventEnvelope

#### Directory Structure
```
notification-service/
  src/notification_service/
    main.py                      # FastAPI app + router registration
    config.py                    # NotificationServiceSettings
    lifespan.py                  # Startup/shutdown (event consumer registration)
    
    models/                      # SQLAlchemy ORM models
      notification_subscription.py
      notification_subscription_channel.py
      notification_message.py
      notification_delivery_log.py
    
    schemas/                     # Pydantic I/O schemas
      notification_schemas.py
      subscription_schemas.py
      event_schemas.py           # Event schemas for published events
    
    repositories/               # Data access layer
      notification_repository.py
      subscription_repository.py
      delivery_log_repository.py
    
    events/                      # Unified event consumer
      __init__.py
      notification_event_consumer.py  # @subscribe('*') - single consumer
    
    services/                    # Business logic
      notification_service.py   # Event → notification transformation
      subscription_service.py   # Preference management
      delivery_service.py        # Channel delivery orchestration
      rate_limiter_service.py   # Rate limiting logic
      template_service.py       # Template registry management
    
    channels/                    # Channel implementations
      __init__.py
      base_channel.py           # Abstract base class
      ui_channel.py             # WebSocket delivery (no log)
      # sms_channel.py          # Future (with log)
      # email_channel.py        # Future (with log)
      # push_channel.py         # Future (with log)
    
    websocket/                   # WebSocket management
      __init__.py
      notification_ws_manager.py # Connection & delivery management
      routes_ws.py               # WebSocket endpoint
    
    api/                         # REST endpoints
      __init__.py
      routes_subscriptions.py   # Subscription CRUD
      routes_notifications.py   # Notification query & read status
      routes_replay.py          # Replay endpoint
    
    templates/                   # Jinja2 templates
      order_filled.jinja2
      trade_executed.jinja2
      position_updated.jinja2
      risk_limit_reached.jinja2
      risk_kill_switch.jinja2
      broker_connection_failed.jinja2
      system_emergency.jinja2
      ai_signal_generated.jinja2
  
  migrations/versions/
    001_initial_schema.py      # All notification tables
  rbac_policies.yaml           # RBAC policies
```

#### Key Components

**NotificationEventConsumer**: Unified event ingestion
- `@subscribe('*')` - Single consumer for all events
- `process_event()` - Delegate to appropriate handler based on event type
- `handle_trading_event()` - Process order, trade, position events
- `handle_risk_event()` - Process risk events
- `handle_broker_event()` - Process broker events
- `handle_system_event()` - Process system events
- `handle_ai_event()` - Process AI events

**NotificationService**: Core business logic
- `process_event()` - Transform domain event into notification
- `check_subscription()` - Verify user has subscription (wildcard pattern matching)
- `validate_event_pattern()` - Validate pattern against EventCatalog
- `determine_category()` - Map event to category from EventCatalog
- `generate_message()` - Render message from template registry
- `read_severity_from_envelope()` - Extract severity from EventEnvelope

**SubscriptionService**: Preference management
- `create_subscription()` - Add user subscription for event pattern
- `update_subscription()` - Modify channels, enabled status, channel config
- `delete_subscription()` - Remove subscription
- `get_user_subscriptions()` - Get all subscriptions for user
- `match_subscription()` - Wildcard pattern matching (fnmatch)

**DeliveryService**: Channel orchestration
- `deliver_notification()` - Send notification via all subscribed channels
- `retry_failed_delivery()` - Async retry with exponential backoff (external channels only)
- `log_delivery_attempt()` - Track delivery in notification_delivery_log (external channels only)
- `should_log_delivery()` - Skip UI channel logging

**RateLimiterService**: Rate limiting
- `check_rate_limit()` - Enforce 100/min, 1000/hour per user/event type
- `get_key()` - Generate rate limit key (user_id, event_type)
- `is_allowed()` - Return True if within limits

**TemplateService**: Template registry
- `render()` - Render message from template with payload variables
- `get_template()` - Load Jinja2 template for event type
- `register_templates()` - Load all templates at startup

**NotificationWSManager**: WebSocket management
- `register_connection()` - Register user WebSocket connection with last_seq param
- `auto_replay()` - Automatic replay on connect if last_seq provided
- `send_notification()` - Real-time delivery to connected user
- `heartbeat_loop()` - 5-second heartbeat for connection health

#### Database Schema

**notification_subscriptions**
```sql
CREATE TABLE notification_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    event_pattern VARCHAR(255) NOT NULL,  -- Supports wildcards: order.*, risk.*
    enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, event_pattern)
);

CREATE INDEX idx_notification_subscriptions_user_id ON notification_subscriptions(user_id);
CREATE INDEX idx_notification_subscriptions_event_pattern ON notification_subscriptions(event_pattern);
```

**notification_subscription_channels**
```sql
CREATE TABLE notification_subscription_channels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES notification_subscriptions(id) ON DELETE CASCADE,
    channel VARCHAR(50) NOT NULL CHECK (channel IN ('UI', 'SMS', 'EMAIL', 'MOBILE_PUSH')),
    config_json JSONB,  -- Per-channel config: {"phone": "+91xxx", "rate_limit": "10/min"}
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    UNIQUE (subscription_id, channel)
);

CREATE INDEX idx_notification_subscription_channels_subscription_id ON notification_subscription_channels(subscription_id);
CREATE INDEX idx_notification_subscription_channels_channel ON notification_subscription_channels(channel);
```

**notification_messages**
```sql
CREATE TABLE notification_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID NOT NULL UNIQUE,
    user_id UUID NOT NULL,
    event_name VARCHAR(255) NOT NULL,
    event_id VARCHAR(255) NOT NULL UNIQUE,  -- From event envelope
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('INFO', 'SUCCESS', 'WARNING', 'ERROR', 'CRITICAL')),
    category VARCHAR(20) NOT NULL CHECK (category IN ('TRADING', 'RISK', 'SYSTEM', 'BROKER', 'AI')),
    title VARCHAR(500) NOT NULL,
    message TEXT NOT NULL,
    payload JSONB,
    read BOOLEAN NOT NULL DEFAULT false,
    sequence_number BIGSERIAL NOT NULL,  -- Auto-increment, no separate table needed
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notification_messages_user_id ON notification_messages(user_id);
CREATE INDEX idx_notification_messages_user_id_created_at ON notification_messages(user_id, created_at DESC);
CREATE INDEX idx_notification_messages_user_id_read ON notification_messages(user_id, read);
CREATE INDEX idx_notification_messages_user_id_severity ON notification_messages(user_id, severity);
CREATE INDEX idx_notification_messages_user_id_category ON notification_messages(user_id, category);
CREATE INDEX idx_notification_messages_event_id ON notification_messages(event_id);
CREATE INDEX idx_notification_messages_sequence_number ON notification_messages(sequence_number);
```

**notification_delivery_log**
```sql
CREATE TABLE notification_delivery_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID NOT NULL,
    user_id UUID NOT NULL,
    channel VARCHAR(50) NOT NULL CHECK (channel IN ('SMS', 'EMAIL', 'MOBILE_PUSH')),  -- UI not logged
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'DELIVERED', 'FAILED')),
    delivered_at TIMESTAMP WITH TIME ZONE,
    failure_reason TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notification_delivery_log_notification_id ON notification_delivery_log(notification_id);
CREATE INDEX idx_notification_delivery_log_notification_id_channel ON notification_delivery_log(notification_id, channel);
CREATE INDEX idx_notification_delivery_log_user_id ON notification_delivery_log(user_id);
CREATE INDEX idx_notification_delivery_log_user_id_status ON notification_delivery_log(user_id, status);
```

**Note**: `notification_sequence` table removed - using BIGSERIAL sequence_number directly.

#### Event Schemas (Published)

**notification.created.v1**
```python
class NotificationCreatedV1(BaseModel):
    event_id: str
    event_name: str = "notification.created.v1"
    event_version: str = "1.0"
    user_id: str
    trace_id: str
    request_id: str
    idempotency_key: str
    timestamp: str
    
    notification_id: str
    source_event_name: str  # Original domain event name
    severity: str  # From EventEnvelope
    category: str  # TRADING, RISK, SYSTEM, BROKER, AI
    channels: list[str]
    sequence_number: int
```

**notification.delivered.v1**
```python
class NotificationDeliveredV1(BaseModel):
    event_id: str
    event_name: str = "notification.delivered.v1"
    event_version: str = "1.0"
    user_id: str
    trace_id: str
    request_id: str
    idempotency_key: str
    timestamp: str
    
    notification_id: str
    channel: str
    delivered_at: str
```

**notification.read.v1**
```python
class NotificationReadV1(BaseModel):
    event_id: str
    event_name: str = "notification.read.v1"
    event_version: str = "1.0"
    user_id: str
    trace_id: str
    request_id: str
    idempotency_key: str
    timestamp: str
    
    notification_id: str
    read_at: str
```

**notification.subscription.updated.v1**
```python
class NotificationSubscriptionUpdatedV1(BaseModel):
    event_id: str
    event_name: str = "notification.subscription.updated.v1"
    event_version: str = "1.0"
    user_id: str
    trace_id: str
    request_id: str
    idempotency_key: str
    timestamp: str
    
    subscription_id: str
    event_pattern: str  # Supports wildcards
    channels: list[str]
    enabled: bool
    action: str  # CREATED, UPDATED, DELETED
```

**notification.delivery.failed.v1**
```python
class NotificationDeliveryFailedV1(BaseModel):
    event_id: str
    event_name: str = "notification.delivery.failed.v1"
    event_version: str = "1.0"
    user_id: str
    trace_id: str
    request_id: str
    idempotency_key: str
    timestamp: str
    
    notification_id: str
    channel: str
    failure_reason: str
    retry_count: int
```

#### REST API Endpoints

**POST /api/v1/notifications/subscriptions**
Create or update notification subscription
```json
Request:
{
  "event_pattern": "order.*",
  "channels": [
    {
      "channel": "UI",
      "config": {}
    },
    {
      "channel": "SMS",
      "config": {
        "phone": "+91xxxxxxxxxx",
        "rate_limit": "10/min"
      }
    }
  ],
  "enabled": true
}

Response:
{
  "id": "uuid",
  "user_id": "uuid",
  "event_pattern": "order.*",
  "channels": [...],
  "enabled": true,
  "created_at": "2026-05-14T10:00:00Z",
  "updated_at": "2026-05-14T10:00:00Z"
}
```

**GET /api/v1/notifications/subscriptions**
Get all subscriptions for current user
```json
Response:
{
  "total": 5,
  "items": [...]
}
```

**PUT /api/v1/notifications/subscriptions/{id}**
Update subscription
```json
Request:
{
  "channels": [...],
  "enabled": false
}
```

**DELETE /api/v1/notifications/subscriptions/{id}**
Delete subscription

**GET /api/v1/notifications**
Query notifications for current user
```json
Query Parameters:
- category: TRADING|RISK|SYSTEM|BROKER|AI (optional)
- severity: INFO|SUCCESS|WARNING|ERROR|CRITICAL (optional)
- read: true|false (optional)
- from_date: ISO8601 timestamp (optional)
- to_date: ISO8601 timestamp (optional)
- limit: int (default 50, max 1000)
- offset: int (default 0)

Response:
{
  "total": 150,
  "items": [
    {
      "notification_id": "uuid",
      "user_id": "uuid",
      "event_name": "order.filled",
      "severity": "SUCCESS",
      "category": "TRADING",
      "title": "Order Filled",
      "message": "Your order for 10 shares of NIFTY50-INDEX was filled at 21000.50",
      "payload": {...},
      "read": false,
      "sequence_number": 12345,
      "created_at": "2026-05-14T10:00:00Z"
    }
  ],
  "limit": 50,
  "offset": 0
}
```

**GET /api/v1/notifications/{id}**
Get specific notification

**PUT /api/v1/notifications/{id}/read**
Mark notification as read
```json
Response:
{
  "notification_id": "uuid",
  "read": true,
  "read_at": "2026-05-14T10:05:00Z"
}
```

**GET /api/v1/notifications/replay**
Replay notifications from sequence_number
```json
Query Parameters:
- since_sequence: int (required)
- limit: int (default 50, max 1000)
- category: TRADING|RISK|SYSTEM|BROKER|AI (optional)
- severity: INFO|SUCCESS|WARNING|ERROR|CRITICAL (optional)

Response:
{
  "total": 10,
  "items": [...],
  "last_sequence_number": 12355
}
```

**WebSocket /api/v1/ws/notifications?last_seq=12345**
Real-time notification delivery with automatic replay
```
Connection:
- JWT token via query param: ?token=xxx&last_seq=12345
- User ID extracted from token
- Automatic replay if last_seq provided
- Live stream after replay

Server → Client (during replay):
{
  "type": "notification",
  "data": {
    "notification_id": "uuid",
    "event_name": "order.filled",
    "severity": "SUCCESS",
    "category": "TRADING",
    "title": "Order Filled",
    "message": "...",
    "sequence_number": 12346,
    "created_at": "2026-05-14T10:00:00Z"
  }
}

Server → Client (replay complete):
{
  "type": "replay_complete",
  "last_sequence_number": 12355,
  "count": 10
}

Server → Client (live notification):
{
  "type": "notification",
  "data": {...}
}

Server → Client (heartbeat):
{
  "type": "ping",
  "timestamp": "2026-05-14T10:00:05Z"
}

Client → Server (pong):
{
  "type": "pong",
  "timestamp": "2026-05-14T10:00:05Z"
}
```

#### Severity Classification

**Publisher-Owned Severity** (in EventEnvelope)
Severity is now the responsibility of event publishers, not the notification service. Each event publisher includes severity in the EventEnvelope when publishing events.

**Examples**:
- BAS publishes `risk.limit_reached.v1` with `severity: CRITICAL`
- MDS publishes `broker.connection.failed.v1` with `severity: ERROR`
- BAS publishes `order.filled` with `severity: SUCCESS`

**Notification Service Behavior**:
- Read severity directly from event envelope
- No local severity mapping or classification logic
- Single source of truth prevents duplicated mappings

**Removed Components**:
- ❌ SeverityService
- ❌ SEVERITY_MAPPING constant
- ❌ Local severity classification logic

#### Category Classification

**Platform-Defined Categories** (for frontend filtering)
```python
CATEGORY_MAPPING = {
    # Trading events
    "order.*": "TRADING",
    "trade.*": "TRADING",
    "position.*": "TRADING",
    
    # Risk events
    "risk.*": "RISK",
    
    # Broker events
    "broker.*": "BROKER",
    
    # System events
    "system.*": "SYSTEM",
    
    # AI events
    "ai.*": "AI",
}
```

**Purpose**: Enable frontend filtering and grouped views without parsing event names.

#### Rate Limiting

**Rules**:
- 100 notifications per minute per (user_id, event_type)
- 1000 notifications per hour per (user_id, event_type)

**Implementation**:
```python
class RateLimiterService:
    async def check_rate_limit(self, user_id: str, event_type: str) -> bool:
        key = f"notification_rate:{user_id}:{event_type}"
        # Check Redis for current count
        # Enforce limits
        return is_allowed
```

**Purpose**: Prevent event storms and UI flooding.

#### Template Registry

**Jinja2 Templates** (enable message changes without code deployment)
```
templates/
  order_filled.jinja2:
    Title: Order Filled
    Message: Your order for {{ quantity }} shares of {{ instrument_id }} was filled at {{ price }}
  
  risk_limit_reached.jinja2:
    Title: Risk Limit Reached
    Message: Risk limit reached for {{ instrument_id }}. Current exposure: {{ exposure }}
  
  broker_connection_failed.jinja2:
    Title: Broker Connection Failed
    Message: Connection to {{ broker_id }} failed. Error: {{ error_message }}
```

**Implementation**:
```python
class TemplateService:
    def render(self, event_name: str, payload: dict) -> tuple[str, str]:
        template = self.get_template(event_name)
        title = template.render_title(payload)
        message = template.render_message(payload)
        return title, message
```

**Benefits**:
- Message changes without code deployment
- Support for multiple languages (future)
- Template versioning for backward compatibility

#### Retention Policy

**Lifecycle**:
- **Hot Storage**: 30 days (frequent queries, full indexing)
- **Warm Storage**: 90 days (archived to slower storage, reduced indexing)
- **Delete After**: 180 days (automatic deletion)

**Implementation**:
- PostgreSQL partitioning by created_at
- Scheduled job for archiving and deletion
- Configurable retention periods

**Purpose**: Prevent unbounded database growth and control storage costs.

#### RBAC Policy

```yaml
notifications:
  read:
    self: true
    services: []
  
subscriptions:
  create:
    self: true
    services: []
  read:
    self: true
    services: []
  update:
    self: true
    services: []
  delete:
    self: true
    services: []
```

Users can only manage their own notifications and subscriptions. No service impersonation needed.

---

### Broker Adapter Service (Existing)

**Changes Required**: 
1. Import event schemas from smarttrade-common
2. Add severity to EventEnvelope when publishing events
3. Remove local event schema definitions

**Event Publishing Standards** (Mandatory):
- Import event schemas from `smarttrade_common.events.schemas.domain`
- Use `DomainEventPublisher` (singleton)
- Use `prepare_event()` for validation
- Use `filter_data(..., safe=True)` for financial fields
- Include `severity` in event metadata
- No direct Redis publishing
- No raw Kafka publishing

**Example**:
```python
from smarttrade_common.events.publisher import DomainEventPublisher

publisher = DomainEventPublisher.get_instance()
prepared = publisher.prepare_event(
    event_name="risk.limit_reached.v1",
    event_data={
        "user_id": str(user_id),
        "instrument_id": instrument_id,
        "exposure": str(exposure),
    },
    idempotency_key=f"risk-limit-{user_id}-{instrument_id}",
    metadata={
        "severity": "CRITICAL",  # Publisher-owned
    }
)
await event_bus.publish(prepared.event_name, prepared.to_dict())
```

**Events Published** (with severity):
- `order.placed` (severity: INFO)
- `order.accepted.v1` (severity: SUCCESS)
- `order.filled` (severity: SUCCESS)
- `order.cancelled` (severity: INFO)
- `order.rejected` (severity: ERROR)
- `trade.executed` (severity: SUCCESS)
- `position.updated` (severity: INFO)
- `risk.warning.v1` (severity: WARNING)
- `risk.limit_reached.v1` (severity: CRITICAL)
- `risk.kill_switch_triggered.v1` (severity: CRITICAL)

---

### Market Data Service (Existing)

**Changes Required**: 
1. Import event schemas from smarttrade-common
2. Add severity to EventEnvelope when publishing events
3. Remove local event schema definitions

**Event Publishing Standards** (Mandatory):
- Import event schemas from `smarttrade_common.events.schemas.domain`
- Use `DomainEventPublisher` (singleton)
- Use `prepare_event()` for validation
- Include `severity` in event metadata
- No direct Redis publishing

**Events Published** (with severity):
- `broker.connection.failed.v1` (severity: ERROR)
- `broker.connection.restored.v1` (severity: SUCCESS)
- `system.maintenance_scheduled.v1` (severity: INFO)
- `system.emergency.v1` (severity: CRITICAL)

---

### Authentication Service (Existing)

**Changes Required**: None (provides JWT validation)

**Integration**: Notification service uses existing JWT middleware from smarttrade-common for WebSocket authentication and REST API authorization.

---

## Detailed Specifications

### Event Contracts

#### Events Consumed

**Unified Consumer Pattern**: Single `@subscribe('*')` consumer receives all events

**Event Types** (examples):
- **Trading**: order.*, trade.*, position.*
- **Risk**: risk.*
- **Broker**: broker.*
- **System**: system.*
- **AI**: ai.*

**Event Envelope Structure** (with severity):
```python
{
    "event_id": "uuid",
    "event_name": "order.filled",
    "event_version": "1.0",
    "user_id": "uuid",
    "trace_id": "uuid",
    "request_id": "uuid",
    "idempotency_key": "uuid",
    "timestamp": "ISO8601",
    "severity": "SUCCESS",  # Publisher-owned
    "payload": {
        # Event-specific data
    }
}
```

#### Events Published

**notification.created.v1**
Published when a notification is created and persisted.

**notification.delivered.v1**
Published when a notification is successfully delivered via an external channel (SMS, Email, Push).

**notification.read.v1**
Published when a user marks a notification as read.

**notification.subscription.updated.v1**
Published when a user creates, updates, or deletes a subscription.

**notification.delivery.failed.v1**
Published when notification delivery fails after max retries (external channels only).

### Data Models

**NotificationSubscription**
```python
class NotificationSubscription(SQLModel, UUIDMixin, TimestampMixin, table=True):
    user_id: uuid.UUID = Field(foreign_key="users.id", index=True)
    event_pattern: str = Field(index=True)  # Supports wildcards
    enabled: bool = Field(default=True)
    
    __table_args__ = (
        UniqueConstraint('user_id', 'event_pattern', name='uq_user_event_pattern'),
    )
```

**NotificationSubscriptionChannel**
```python
class NotificationSubscriptionChannel(SQLModel, UUIDMixin, TimestampMixin, table=True):
    subscription_id: uuid.UUID = Field(foreign_key="notification_subscriptions.id", ondelete="CASCADE")
    channel: ChannelEnum = Field(index=True)
    config_json: dict = Field(sa_column=Column(JSONB), default={})
    
    __table_args__ = (
        UniqueConstraint('subscription_id', 'channel', name='uq_subscription_channel'),
    )
```

**NotificationMessage**
```python
class NotificationMessage(SQLModel, UUIDMixin, TimestampMixin, table=True):
    notification_id: uuid.UUID = Field(unique=True, index=True)
    user_id: uuid.UUID = Field(foreign_key="users.id", index=True)
    event_name: str = Field(index=True)
    event_id: str = Field(unique=True, index=True)  # From event envelope
    severity: SeverityEnum = Field(index=True)
    category: CategoryEnum = Field(index=True)
    title: str = Field(max_length=500)
    message: str
    payload: dict = Field(sa_column=Column(JSONB))
    read: bool = Field(default=True, index=True)
    sequence_number: int = Field(default_factory=next_sequence, index=True)  # BIGSERIAL
```

**NotificationDeliveryLog**
```python
class NotificationDeliveryLog(SQLModel, UUIDMixin, TimestampMixin, table=True):
    notification_id: uuid.UUID = Field(foreign_key="notification_messages.id")
    user_id: uuid.UUID = Field(index=True)
    channel: ChannelEnum = Field(index=True)  # SMS, EMAIL, MOBILE_PUSH only (UI not logged)
    status: DeliveryStatusEnum = Field(default=DeliveryStatus.PENDING, index=True)
    delivered_at: Optional[datetime] = None
    failure_reason: Optional[str] = None
    retry_count: int = Field(default=0)
```

**Enums**:
```python
class SeverityEnum(str, Enum):
    INFO = "INFO"
    SUCCESS = "SUCCESS"
    WARNING = "WARNING"
    ERROR = "ERROR"
    CRITICAL = "CRITICAL"

class CategoryEnum(str, Enum):
    TRADING = "TRADING"
    RISK = "RISK"
    SYSTEM = "SYSTEM"
    BROKER = "BROKER"
    AI = "AI"

class ChannelEnum(str, Enum):
    UI = "UI"
    SMS = "SMS"
    EMAIL = "EMAIL"
    MOBILE_PUSH = "MOBILE_PUSH"

class DeliveryStatusEnum(str, Enum):
    PENDING = "PENDING"
    DELIVERED = "DELIVERED"
    FAILED = "FAILED"
```

### Channel Interface

```python
from abc import ABC, abstractmethod
from enum import Enum

class ChannelType(str, Enum):
    UI = "UI"
    SMS = "SMS"
    EMAIL = "EMAIL"
    MOBILE_PUSH = "MOBILE_PUSH"

class DeliveryResult(BaseModel):
    success: bool
    delivered_at: Optional[datetime] = None
    failure_reason: Optional[str] = None
    should_retry: bool = False
    should_log: bool = True  # UI channel returns False

class NotificationChannel(ABC):
    @abstractmethod
    async def send(self, notification: NotificationMessage, user_id: uuid.UUID, config: dict) -> DeliveryResult:
        """Send notification via this channel."""
        pass
    
    @abstractmethod
    def validate_recipient(self, recipient: str) -> bool:
        """Validate recipient format for this channel."""
        pass
    
    @abstractmethod
    def get_channel_type(self) -> ChannelType:
        """Return channel type enum."""
        pass
    
    @abstractmethod
    async def get_recipient(self, user_id: uuid.UUID) -> Optional[str]:
        """Get recipient address for user (phone, email, device token)."""
        pass
    
    @abstractmethod
    def should_log_delivery(self) -> bool:
        """Return True if delivery should be logged (UI returns False)."""
        pass
```

### Consumer Flow

```
Event Received via @subscribe('*')
↓
Extract event_id, user_id, event_name, severity, payload from envelope
↓
Check idempotency: Does notification with event_id already exist?
  If yes: Skip (exactly-once guarantee)
  If no: Continue
↓
Check rate limit: (user_id, event_name) within 100/min, 1000/hour?
  If exceeded: Skip, log warning
  If within limits: Continue
↓
Query notification_subscriptions for user_id
↓
Match event_name against subscription.event_pattern (fnmatch wildcard)
  If no match: Skip notification
  If match: Continue
↓
Determine category from CATEGORY_MAPPING
↓
Read severity from event envelope (publisher-owned)
↓
Generate notification message:
  - Load Jinja2 template for event_name
  - Render title and message with payload variables
  - Fallback to default message if template fails
↓
Create NotificationMessage record (BIGSERIAL sequence_number auto-generated)
↓
Publish notification.created.v1 event
↓
For each channel in subscription channels:
  - Get channel config from notification_subscription_channels
  - Instantiate channel handler
  - Call channel.send()
  - If channel.should_log_delivery(): Log in notification_delivery_log
  - If failure and channel is external: Schedule retry with exponential backoff
↓
If UI channel and user connected via WebSocket:
  - Send via NotificationWSManager.send_notification()
  - No delivery log (UI channel not logged)
↓
Return
```

### Subscription Matching Logic

**Wildcard Pattern Matching**:
```python
from fnmatch import fnmatch

def match_subscription(event_name: str, event_pattern: str) -> bool:
    """Match event name against subscription pattern (supports wildcards)."""
    return fnmatch(event_name, event_pattern)

# Examples:
match_subscription("order.filled", "order.*")  # True
match_subscription("risk.limit_reached.v1", "risk.*")  # True
match_subscription("order.filled", "order.filled")  # True
match_subscription("order.filled", "trade.*")  # False
```

**Subscription Examples**:
- `"order.*"` - All order events
- `"risk.*"` - All risk events
- `"broker.connection.*"` - All broker connection events
- `"trade.executed"` - Specific event only

### Replay Architecture

**Automatic WebSocket Replay**:
```
Client connects to /api/v1/ws/notifications?last_seq=12345
↓
Server extracts last_seq from query parameter
↓
If last_seq provided:
  Server queries notification_messages:
    WHERE user_id = ? AND sequence_number > 12345
    ORDER BY sequence_number ASC
    LIMIT 100
  Server sends each notification in sequence
  Server sends replay_complete message
↓
After replay (or if no last_seq), live notifications resume
```

**REST API Replay**:
```
Client calls GET /api/v1/notifications/replay?since_sequence=12300
↓
Server queries notification_messages with pagination
↓
Returns paginated list with last_sequence_number
↓
Client can request next page using last_sequence_number
```

**Sequence Tracking**:
- Client stores last received sequence_number locally
- On reconnect, sends last_seq via query parameter
- Ensures no gaps in notification history
- Supports multiple devices (each tracks own sequence)

**Benefits**:
- Reduced client complexity (no replay message roundtrip)
- Automatic replay on reconnect
- Seamless transition from replay to live stream

### Failure Handling

**Event Processing Failures**
- Event validation failure: Log warning, skip, send to DLQ
- Rate limit exceeded: Log warning, skip
- Subscription lookup failure: Log warning, skip (assume no subscription)
- Template rendering failure: Use default message, log warning
- Database insert failure: Log error, send to DLQ

**Delivery Failures (UI Channel)**
- User not connected: Log info, no retry (UI is ephemeral)
- Send failure: Log warning, retry once immediately
- Retry failure: Log as warning, no further retry
- No delivery log (UI channel not logged)

**Delivery Failures (External Channels)**
- Initial failure: Mark as PENDING in delivery_log
- Retry with exponential backoff: 1s, 5s, 30s, 5min
- Max retries: 3 (configurable)
- After max retries: Mark as FAILED, publish notification.delivery.failed.v1
- Delivery log updated for each attempt

**Dead Letter Queue**
- Events that fail processing go to notification.dlq topic
- DLQ payload includes original event + error context
- Monitoring alerts on DLQ buildup
- Manual replay from DLQ for investigation

**Circuit Breaker**
- Per-channel circuit breaker for external channels only
- Threshold: 5 consecutive failures
- Timeout: 30 seconds before retry
- Half-open: Test with single delivery before full recovery

### Extensibility for Future Channels

**Adding a New Channel**

1. **Implement Channel Interface**
```python
class SMSChannel(NotificationChannel):
    async def send(self, notification: NotificationMessage, user_id: uuid.UUID, config: dict) -> DeliveryResult:
        # Integrate with SMS gateway (Twilio, AWS SNS, etc.)
        # Use config: {"phone": "+91xxx", "rate_limit": "10/min"}
        pass
    
    def validate_recipient(self, recipient: str) -> bool:
        # Validate phone number format
        pass
    
    def get_channel_type(self) -> ChannelType:
        return ChannelType.SMS
    
    async def get_recipient(self, user_id: uuid.UUID) -> Optional[str]:
        # Query notification_subscription_channels config for phone
        pass
    
    def should_log_delivery(self) -> bool:
        return True  # External channels log delivery
```

2. **Register Channel**
```python
# In lifespan.py
from notification_service.channels.sms_channel import SMSChannel

channel_registry.register(SMSChannel())
```

3. **Add Channel Configuration**
```python
# In config.py
class NotificationServiceSettings(CommonSettings):
    SMS_ENABLED: bool = True
    SMS_GATEWAY_API_KEY: SecretStr
    SMS_RATE_LIMIT_PER_MINUTE: int = 10
```

4. **Channel Config in Subscription**
```json
{
  "channel": "SMS",
  "config": {
    "phone": "+91xxxxxxxxxx",
    "rate_limit": "10/min"
  }
}
```

**Channel Selection Logic**
```python
for channel_config in subscription.channels:
    channel = channel_config.channel
    config = channel_config.config
    
    if not channel_registry.is_enabled(channel):
        continue
    
    channel_handler = channel_registry.get_handler(channel)
    
    result = await channel_handler.send(notification, user_id, config)
    
    if result.should_log_delivery:
        await delivery_log_repository.log_attempt(notification.id, channel, result)
```

### Performance Requirements

- **WebSocket Latency**: < 100ms from event to delivery
- **Concurrent Connections**: Support 10,000+ simultaneous WebSocket connections
- **Event Processing**: < 50ms per event (rate limit check + subscription match + template render)
- **Database Queries**: < 20ms for subscription lookup, < 50ms for notification query
- **Replay Performance**: < 500ms for 100 notification replay
- **Delivery Success Rate**: 99.9% for UI channel, 95% for external channels
- **Rate Limiting**: 100/min, 1000/hour per user/event type (enforced in < 5ms)

### Monitoring & Observability

**Metrics**
- `notification_events_total` - Events consumed by type
- `notification_rate_limit_exceeded_total` - Rate limit violations by user/event
- `notification_created_total` - Notifications created by severity and category
- `notification_delivered_total` - Notifications delivered by channel
- `notification_delivery_failed_total` - Failed deliveries by channel
- `notification_delivery_latency_seconds` - Delivery latency by channel
- `websocket_connections_active` - Active WebSocket connections
- `websocket_messages_sent_total` - WebSocket messages sent
- `websocket_replay_requests_total` - Replay requests
- `template_render_errors_total` - Template rendering failures

**Logging**
- Event processing logs (rate limit check, subscription match, severity, category)
- Template rendering logs (template used, render time, errors)
- Delivery logs (channel, status, latency) - external channels only
- WebSocket logs (connect with last_seq, disconnect, replay, heartbeat)
- Error logs (processing failures, delivery failures, rate limit violations)

**Alerts**
- DLQ size > 100 events
- Delivery success rate < 95% for any external channel
- WebSocket connection failures > 5% in 5 minutes
- Event processing latency > 1s for 10 consecutive events
- Database query latency > 100ms for 10 consecutive queries
- Rate limit violations > 10% of events for a user

---

## Dependencies & Sequencing

### Dependencies

**External Dependencies**
- Redis (event bus) - Already in place
- PostgreSQL (database) - Already in place
- smarttrade-common (shared library) - Requires update for event schemas
- Jinja2 (template engine) - New dependency

**Service Dependencies**
- smarttrade-common (event catalog) - Requires update (Phase 0)
- broker-adapter-service (event publisher with severity, centralized schemas) - Requires update (Phase 0)
- market-data-service (event publisher with severity, centralized schemas) - Requires update (Phase 0)
- authentication-service (JWT validation) - Already in place

**Blocking Dependencies**
- smarttrade-common must add event catalog and centralized schemas (Phase 0)
- broker-adapter-service and market-data-service must migrate to centralized schemas and add severity to EventEnvelope (Phase 0)
- Notification service depends on centralized event schemas for subscription validation
- This is a coordinated change across services

### Implementation Sequence

**Phase 0: Centralized Event Schemas** (Week 1-2) - **BLOCKING**
1. Add event catalog to smarttrade-common
   - Create `events/schemas/domain/` directory structure
   - Move existing event schemas from services to smarttrade-common
   - Create EventCatalog class with metadata
   - Update SchemaRegistry to auto-register from catalog
2. Migrate broker-adapter-service
   - Import event schemas from smarttrade-common
   - Add severity to EventEnvelope when publishing
   - Remove local event schema definitions
   - Test event publishing with imported schemas
3. Migrate market-data-service
   - Import event schemas from smarttrade-common
   - Add severity to EventEnvelope when publishing
   - Remove local event schema definitions
   - Test event publishing with imported schemas
4. Deploy smarttrade-common and publisher updates
5. Validate event schemas are registered correctly

**Phase 1: Core Infrastructure** (Week 2-3)

**Phase 1: Core Infrastructure** (Week 2)
1. Create notification-service directory structure
2. Set up FastAPI app with smarttrade-common
3. Configure database and migrations
4. Implement BaseRepository pattern
5. Set up event bus integration
6. Add Jinja2 template engine dependency

**Phase 2: Database & Models** (Week 2)
1. Create database models (notification_subscriptions, notification_subscription_channels, notification_messages, notification_delivery_log)
2. Create Alembic migration
3. Implement repositories
4. Write unit tests for repositories

**Phase 3: Unified Event Consumption** (Week 3)
1. Implement NotificationEventConsumer with @subscribe('*')
2. Implement internal event delegation (trading, risk, broker, system, ai)
3. Implement NotificationService (event → notification transformation)
4. Implement RateLimiterService
5. Implement idempotency handling (event_id check)
6. Write unit tests for consumer and services

**Phase 4: Subscription Management** (Week 3-4)
1. Implement SubscriptionService with wildcard pattern matching
2. Create REST API endpoints for subscriptions
3. Implement per-channel configuration
4. Write unit and integration tests
5. Test subscription enforcement in event processing

**Phase 5: Template Registry** (Week 4)
1. Create Jinja2 templates for common events
2. Implement TemplateService
3. Implement template loading and rendering
4. Add fallback to default message
5. Write unit tests for template rendering

**Phase 6: WebSocket Delivery** (Week 4)
1. Implement NotificationWSManager
2. Create WebSocket endpoint with last_seq query parameter
3. Implement automatic replay on connect
4. Implement heartbeat mechanism
5. Implement real-time notification delivery
6. Write integration tests for WebSocket

**Phase 7: Category Classification** (Week 4-5)
1. Implement CATEGORY_MAPPING
2. Add category to notification creation
3. Update API to support category filtering
4. Write unit tests for category mapping

**Phase 8: Channel Abstraction** (Week 5)
1. Implement NotificationChannel interface
2. Implement UIChannel (WebSocket, no log)
3. Implement channel registry
4. Update delivery service to use channel abstraction
5. Implement should_log_delivery logic
6. Write unit tests for channel interface

**Phase 9: Failure Handling & Retry** (Week 5-6)
1. Implement delivery logging (external channels only)
2. Implement retry mechanism with exponential backoff
3. Implement DLQ handling
4. Implement circuit breaker (external channels only)
5. Write tests for failure scenarios

**Phase 10: Retention Policy** (Week 6)
1. Implement database partitioning by created_at
2. Create scheduled job for archiving (30d → warm)
3. Create scheduled job for deletion (180d)
4. Test retention lifecycle
5. Write unit tests for retention logic

**Phase 11: Monitoring & Observability** (Week 6)
1. Add Prometheus metrics (including rate limit metrics)
2. Add structured logging
3. Configure alerting rules
4. Test monitoring dashboards

**Phase 12: Testing & Hardening** (Week 6-7)
1. Write comprehensive unit tests (80%+ coverage)
2. Write integration tests (event flow, WebSocket, replay, rate limiting)
3. Load testing (WebSocket connections, event processing, rate limiting)
4. Chaos testing (delivery failures, network issues)
5. Security audit (JWT validation, RBAC, wildcard injection)

**Phase 13: Documentation** (Week 7)
1. Update CLAUDE.md for notification-service
2. Create API documentation
3. Create operator runbooks
4. Create troubleshooting guides

**Phase 14: Deployment** (Week 8)
1. Docker image build
2. Kubernetes deployment configuration
3. Database migration in staging
4. Smoke testing in staging
5. Production deployment
6. Production verification

### What Blocks What

**Blocking Dependencies**:
- Phase 0 (Publisher Updates) must complete before Phase 3 (Event Consumption)
- Notification service depends on severity in EventEnvelope

**No Other Blocking**:
- Notification service can be developed independently after Phase 0
- No synchronous dependencies on other services
- Can be developed and deployed in parallel after publisher updates

**Deployment Order**:
1. smarttrade-common (add event catalog and centralized schemas)
2. broker-adapter-service (migrate to centralized schemas, add severity to EventEnvelope)
3. market-data-service (migrate to centralized schemas, add severity to EventEnvelope)
4. notification-service (new)
5. Frontend integration - After notification-service is deployed
6. Future channels (SMS, Email, Push) - After core is stable

---

## Testing Strategy

### Unit Tests

**Target Coverage**: 80%+ for core logic

**Test Areas**
- **Repositories**: CRUD operations, filtering, pagination
- **Services**: NotificationService, SubscriptionService, DeliveryService, RateLimiterService, TemplateService
- **Event Consumer**: Unified consumer, internal delegation, idempotency
- **Channel Interface**: Mock implementations, validation logic, should_log_delivery
- **WebSocket Manager**: Connection management, automatic replay, heartbeat
- **Subscription Matching**: Wildcard pattern matching (fnmatch)
- **Template Rendering**: Jinja2 template rendering, fallback logic
- **Category Mapping**: Event to category mapping

**Example Tests**
```python
async def test_unified_event_consumer():
    # Test @subscribe('*') receives all events
    consumer = NotificationEventConsumer()
    await consumer.handle_order_event(test_order_event)
    await consumer.handle_risk_event(test_risk_event)
    # Verify both notifications created

async def test_subscription_wildcard_matching():
    # Test wildcard pattern matching
    await subscription_service.create_subscription(user_id, "order.*", ["UI"])
    has_subscription = await subscription_service.match_subscription(user_id, "order.filled")
    assert has_subscription is True

async def test_rate_limiting():
    # Test rate limit enforcement
    for i in range(105):
        await rate_limiter.check_rate_limit(user_id, "order.filled")
    # 101st should be blocked
    allowed = await rate_limiter.check_rate_limit(user_id, "order.filled")
    assert allowed is False

async def test_template_rendering():
    # Test Jinja2 template rendering
    title, message = template_service.render("order.filled", {
        "quantity": 10,
        "instrument_id": "NIFTY50-INDEX",
        "price": "21000.50"
    })
    assert "Order Filled" in title
    assert "10 shares" in message
```

### Integration Tests

**Test Areas**
- **Event Flow**: End-to-end event consumption → notification creation → delivery
- **WebSocket**: Connection with last_seq, automatic replay, heartbeat, message delivery
- **REST API**: Subscription CRUD (with wildcards), notification query (with category), replay endpoint
- **Database**: Transaction handling, concurrent updates, BIGSERIAL sequence generation
- **Rate Limiting**: Redis-based rate limiting enforcement
- **Template Registry**: Template loading, rendering, fallback

**Example Tests**
```python
async def test_unified_event_flow():
    # Publish test event to event bus
    await event_bus.publish("order.filled", test_event_payload)
    # Wait for processing
    await asyncio.sleep(0.5)
    # Verify notification created with correct severity from envelope
    notifications = await notification_repo.list(user_id=test_user_id)
    assert len(notifications) == 1
    assert notifications[0].severity == "SUCCESS"  # From envelope
    assert notifications[0].category == "TRADING"

async def test_websocket_auto_replay():
    # Create test notifications
    await create_test_notifications(user_id, count=10)
    # Connect WebSocket with last_seq
    ws_client = await websocket_client.connect(user_id, last_seq=5)
    # Verify automatic replay
    messages = await ws_client.receive_messages(count=5)
    assert all(msg["sequence_number"] > 5 for msg in messages)
    # Verify replay_complete message
    replay_complete = await ws_client.receive_message()
    assert replay_complete["type"] == "replay_complete"
```

### Load Tests

**Test Areas**
- **WebSocket Connections**: 10,000 concurrent connections
- **Event Processing**: 10,000 events/second
- **Database Queries**: 1,000 queries/second
- **Replay Performance**: 1,000 notification replay
- **Rate Limiting**: 10,000 concurrent users, rate limit enforcement

**Tools**: Locust, k6

### Chaos Tests

**Test Areas**
- **Event Bus Failure**: Redis connection loss, event delivery failure
- **Database Failure**: Connection loss, slow queries, deadlocks
- **WebSocket Failure**: Network issues, client disconnects
- **Delivery Failures**: Channel failures, retry logic, circuit breaker
- **Rate Limiting**: Redis failure, rate limit bypass attempts

**Tools**: Chaos Mesh, custom failure injection

### E2E Tests

**Location**: smarttrade-tests repository

**Test Areas**
- **Cross-Service Flows**: BAS → notification-service → frontend (with severity)
- **User Journeys**: Subscribe to wildcard patterns, receive notifications, automatic replay
- **Integration with Frontend**: WebSocket connection with last_seq, notification center UI with category filtering

**Not in notification-service repo**: Per project guidelines, E2E tests belong in smarttrade-tests

---

## Success Criteria Verification

### Functional Requirements
- [ ] Users can subscribe to domain event patterns (wildcards supported) via REST API
- [ ] Notifications are delivered in real-time via WebSocket with < 100ms latency
- [ ] Users can replay missed notifications from any sequence number (automatic on reconnect)
- [ ] Notification preferences are persisted and enforced with per-channel configuration
- [ ] Severity classification is publisher-owned in EventEnvelope (single source of truth)
- [ ] Channel abstraction allows adding SMS/Email/Push without core changes
- [ ] At-least-once delivery guarantee with idempotency handling
- [ ] Support for 10,000+ concurrent WebSocket connections
- [ ] 99.9% notification delivery success rate for UI channel
- [ ] Comprehensive monitoring and alerting for delivery failures
- [ ] Rate limiting prevents event storms (100/min, 1000/hour)
- [ ] Template-based messaging enables message changes without code deployment
- [ ] Category classification enables frontend filtering
- [ ] Centralized event schemas in smarttrade-common (single source of truth)
- [ ] Subscription patterns validated against EventCatalog
- [ ] Event catalog provides centralized event discovery

### Non-Functional Requirements
- [ ] 80%+ unit test coverage for core logic
- [ ] Integration tests for event flow, WebSocket, replay, rate limiting
- [ ] Load tests pass (10k connections, 10k events/sec, rate limiting)
- [ ] Chaos tests pass (failure scenarios)
- [ ] Monitoring metrics exposed (Prometheus)
- [ ] Alerting rules configured
- [ ] Documentation complete (CLAUDE.md, API docs, runbooks)

### Architecture Requirements
- [ ] Strict boundaries respected (no order execution, no risk calculations)
- [ ] Event-driven architecture (no synchronous calls to BAS)
- [ ] Unified event consumption (@subscribe('*'))
- [ ] Publisher-owned severity (in EventEnvelope)
- [ ] Channel abstraction implemented (extensible for future channels)
- [ ] Idempotency handling (event_id check)
- [ ] Replay support (automatic via last_seq query param)
- [ ] Failure handling (retry, DLQ, circuit breaker)
- [ ] Rate limiting (per user/event type)
- [ ] Template registry (Jinja2)
- [ ] Category classification (TRADING, RISK, SYSTEM, BROKER, AI)
- [ ] Retention policy (30d hot, 90d warm, 180d delete)
- [ ] UI delivery logging skipped (external channels only)
- [ ] BIGSERIAL sequence_number (no separate sequence table)
- [ ] Centralized event schemas in smarttrade-common (EventCatalog)
- [ ] Subscription pattern validation against EventCatalog

### SmartTrade Alignment
- [ ] Event publishing standards enforced (DomainEventPublisher, PreparedEvent)
- [ ] No direct Redis publishing
- [ ] No raw Kafka publishing
- [ ] filter_data(..., safe=True) for financial fields
- [ ] Severity in EventEnvelope (publisher-owned)
- [ ] Unified event consumer pattern (@subscribe('*'))
- [ ] Centralized event schemas in smarttrade-common (EventCatalog)
- [ ] Event schemas imported from smarttrade-common (no local definitions)

---

## Risks & Mitigations

### Risk 1: Publisher Coordination
**Risk**: broker-adapter-service and market-data-service must add severity to EventEnvelope before notification service can use it
**Mitigation**: 
- Phase 0 dedicated to publisher updates
- Coordinated deployment across services
- Backward compatibility check (default to INFO if severity missing)
- Monitoring for events without severity

### Risk 2: Event Volume Overload
**Risk**: High event volume (e.g., market data events) could overwhelm notification service
**Mitigation**: 
- Rate limiting per user/event type (100/min, 1000/hour)
- Subscription filtering (users only receive subscribed events)
- Backpressure handling in event consumers
- Separate consumer groups for high-volume events

### Risk 3: WebSocket Connection Limits
**Risk**: 10,000+ concurrent connections may exceed resource limits
**Mitigation**:
- Horizontal scaling (multiple notification-service instances)
- Connection pooling and load balancing
- Resource monitoring and auto-scaling
- Connection timeout and cleanup

### Risk 4: Database Performance
**Risk**: High notification volume could slow database queries
**Mitigation**:
- Proper indexing (user_id, sequence_number, created_at, category)
- Database connection pooling
- Query optimization and caching
- Partitioning by created_at for retention lifecycle
- BIGSERIAL sequence_number reduces write contention

### Risk 5: Template Rendering Complexity
**Risk**: Jinja2 template rendering may fail for complex events
**Mitigation**:
- Fallback to default message on template failure
- Template validation at startup
- Template versioning for backward compatibility
- Monitoring of template rendering errors

### Risk 6: Replay Performance
**Risk**: Large replay requests (e.g., 1000+ notifications) could be slow
**Mitigation**:
- Pagination support (max 100 notifications per replay request)
- Efficient query with proper indexing on sequence_number
- Async streaming for large replay sets
- Client-side pagination for large histories

### Risk 7: Channel Integration Complexity
**Risk**: Future channels (SMS, Email, Push) may have complex integration requirements
**Mitigation**:
- Channel abstraction isolates complexity
- Per-channel configuration in JSON
- Channel-specific retry logic
- Per-channel monitoring and alerting
- Delivery logging only for external channels

### Risk 8: Wildcard Subscription Abuse
**Risk**: Users may subscribe to "*" causing excessive notifications
**Mitigation**:
- Rate limiting per user/event type
- Maximum number of subscriptions per user
- Admin override for abusive subscriptions
- Monitoring for unusual subscription patterns

### Risk 9: Security Concerns
**Risk**: WebSocket authentication, wildcard injection, notification content exposure
**Mitigation**:
- JWT validation for all WebSocket connections
- User-scoped queries (users only see own notifications)
- RBAC enforcement on all endpoints
- Wildcard pattern validation (prevent regex injection)
- Audit logging for subscription changes

---

## Future Enhancements

### Phase 2: SMS Channel (Q3 2026)
- Integrate with SMS gateway (Twilio, AWS SNS)
- Phone number validation and management
- SMS-specific templates (160 char limit)
- Per-user rate limiting (via channel config)
- Delivery tracking via gateway callbacks
- Delivery logging in notification_delivery_log

### Phase 3: Email Channel (Q3 2026)
- Integrate with email service (AWS SES, SendGrid)
- Email validation and management
- HTML email templates with rich formatting
- Attachment support
- Bounce handling and unsubscribe
- Delivery logging in notification_delivery_log

### Phase 4: Mobile Push Channel (Q4 2026)
- Integrate with FCM (Android) and APNs (iOS)
- Device token management
- Platform-specific payloads
- Batch delivery support
- Action buttons and deep links
- Delivery logging in notification_delivery_log

### Phase 5: Notification Batching (Q4 2026)
- Batch multiple notifications into single delivery
- Digest mode (hourly/daily summaries)
- User-configurable batching preferences
- Reduced delivery costs for external channels

### Phase 6: Notification Analytics (Q1 2027)
- Per-user engagement metrics
- Channel effectiveness analysis
- A/B testing for message content
- Notification preference optimization
- Template performance analytics

---

## Appendix

### A. Event Severity Examples (Publisher-Owned)

**Broker Adapter Service**:
```python
# Order events
publish_event("order.placed", severity="INFO")
publish_event("order.accepted.v1", severity="SUCCESS")
publish_event("order.filled", severity="SUCCESS")
publish_event("order.cancelled", severity="INFO")
publish_event("order.rejected", severity="ERROR")

# Trade events
publish_event("trade.executed", severity="SUCCESS")

# Position events
publish_event("position.updated", severity="INFO")

# Risk events
publish_event("risk.warning.v1", severity="WARNING")
publish_event("risk.limit_reached.v1", severity="CRITICAL")
publish_event("risk.kill_switch_triggered.v1", severity="CRITICAL")
```

**Market Data Service**:
```python
# Broker events
publish_event("broker.connection.failed.v1", severity="ERROR")
publish_event("broker.connection.restored.v1", severity="SUCCESS")

# System events
publish_event("system.maintenance_scheduled.v1", severity="INFO")
publish_event("system.emergency.v1", severity="CRITICAL")
```

### B. Notification Category Mapping

```python
CATEGORY_MAPPING = {
    # Trading events
    "order.*": "TRADING",
    "trade.*": "TRADING",
    "position.*": "TRADING",
    
    # Risk events
    "risk.*": "RISK",
    
    # Broker events
    "broker.*": "BROKER",
    
    # System events
    "system.*": "SYSTEM",
    
    # AI events
    "ai.*": "AI",
}
```

### C. Jinja2 Template Examples

**templates/order_filled.jinja2**
```jinja2
Title: Order Filled
Message: Your order for {{ quantity }} shares of {{ instrument_id }} was filled at {{ price }}
```

**templates/risk_limit_reached.jinja2**
```jinja2
Title: Risk Limit Reached
Message: Risk limit reached for {{ instrument_id }}. Current exposure: {{ exposure }}
```

**templates/broker_connection_failed.jinja2**
```jinja2
Title: Broker Connection Failed
Message: Connection to {{ broker_id }} failed. Error: {{ error_message }}
```

### D. WebSocket Message Schema

**Notification Message**
```json
{
  "type": "notification",
  "data": {
    "notification_id": "uuid",
    "event_name": "order.filled",
    "severity": "SUCCESS",
    "category": "TRADING",
    "title": "Order Filled",
    "message": "Your order for 10 shares of NIFTY50-INDEX was filled at 21000.50",
    "sequence_number": 12345,
    "created_at": "2026-05-14T10:00:00Z"
  }
}
```

**Heartbeat (Server → Client)**
```json
{
  "type": "ping",
  "timestamp": "2026-05-14T10:00:05Z"
}
```

**Heartbeat (Client → Server)**
```json
{
  "type": "pong",
  "timestamp": "2026-05-14T10:00:05Z"
}
```

**Replay Complete (Server → Client)**
```json
{
  "type": "replay_complete",
  "last_sequence_number": 12355,
  "count": 10
}
```

### E. Configuration Reference

**Environment Variables**
```bash
# Core
SERVICE_NAME=notification-service
ENV=local
LOG_LEVEL=INFO

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/smarttrade_notification_service
DB_POOL_SIZE=20
DB_MAX_OVERFLOW=50

# Event Bus
REDIS_URL=redis://localhost:6379/0

# JWT
JWT_SECRET_KEY=<base64>
TOKEN_ENCRYPTION_KEY=<base64>

# WebSocket
WS_HEARTBEAT_INTERVAL=5
WS_MAX_CONNECTIONS=10000
WS_REPLAY_LIMIT=100

# Rate Limiting
RATE_LIMIT_PER_MINUTE=100
RATE_LIMIT_PER_HOUR=1000

# Notification
NOTIFICATION_RETENTION_HOT_DAYS=30
NOTIFICATION_RETENTION_WARM_DAYS=90
NOTIFICATION_RETENTION_DELETE_DAYS=180

# Future Channels (Phase 2+)
SMS_ENABLED=false
SMS_GATEWAY_API_KEY=<key>
SMS_RATE_LIMIT_PER_MINUTE=10

EMAIL_ENABLED=false
EMAIL_GATEWAY_API_KEY=<key>
EMAIL_FROM_ADDRESS=noreply@smarttrade.asia

PUSH_ENABLED=false
FCM_API_KEY=<key>
APNS_KEY_FILE=/path/to/key.p8
```

### F. SmartTrade Event Publishing Standards

**Mandatory Rules**:
1. Use `DomainEventPublisher` singleton (from smarttrade_common.events.publisher)
2. Use `prepare_event()` for validation and envelope creation
3. Use `filter_data(..., safe=True)` for financial fields (no floats)
4. Include `severity` in event metadata (publisher-owned)
5. No direct Redis publishing (use EventBus)
6. No raw Kafka publishing (use EventBus)

**Example**:
```python
from smarttrade_common.events.publisher import DomainEventPublisher
from smarttrade_common.financial.validation import filter_data

publisher = DomainEventPublisher.get_instance()

# Filter financial data
safe_payload = filter_data({
    "user_id": str(user_id),
    "instrument_id": instrument_id,
    "quantity": quantity,
    "price": str(price),  # String for precision
}, safe=True)

# Prepare event with severity
prepared = publisher.prepare_event(
    event_name="risk.limit_reached.v1",
    event_data=safe_payload,
    idempotency_key=f"risk-limit-{user_id}-{instrument_id}-{timestamp}",
    metadata={
        "severity": "CRITICAL",  # Publisher-owned
    }
)

# Publish via EventBus
await event_bus.publish(prepared.event_name, prepared.to_dict())
```

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-05-14 | Devin | Initial HLD for Notification Service |
| 2.0 | 2026-05-14 | Devin | Refactored HLD with 12 architectural improvements (unified consumer, publisher-owned severity, wildcard subscriptions, template registry, rate limiting, category classification, retention policy, BIGSERIAL sequence, automatic replay, UI log skip, channel config normalization, SmartTrade publishing standards) |
| 2.1 | 2026-05-14 | Devin | Added centralized event definitions in smarttrade-common (event catalog, schema registry, subscription validation, migration path) |