# smarttrade-common — Shared Library Design

**Version:** 1.0
**Package:** `smarttrade_common`
**Used by:** All backend services

---

## 1. Purpose

`smarttrade-common` is the internal shared library that enforces cross-cutting concerns uniformly across all SmartTrade services. It prevents code duplication and ensures that security, observability, resilience, and financial correctness patterns are implemented consistently.

**Rule:** Any cross-service concern that is implemented differently in two services is a bug. It should be in `smarttrade-common`.

---

## 2. Module Reference

### app_factory.py

Creates a FastAPI application with all standard middleware and exception handlers pre-wired. Every service calls this instead of instantiating `FastAPI()` directly.

```python
def create_app(
    title: str,
    settings: CommonSettings,
    routers: list[APIRouter],
    lifespan: Callable | None = None,
    public_paths: list[str] = [],
) -> FastAPI:
    # Applies: CORS, rate limiting, request_id, Bearer auth, exception handlers
    # Prometheus metrics if PROMETHEUS_ENABLED=true
    # Sentry integration if SENTRY_ENABLED=true
```

### config.py

Base settings class using Pydantic Settings v2. All services extend this:

```python
class CommonSettings(BaseSettings):
    env: str = "local"
    service_name: str
    log_level: str = "INFO"
    database_url: str
    jwt_secret_key: str
    token_encryption_key: str
    event_bus: str = "redis"
    event_bus_url: str = "redis://localhost:6379/0"
    prometheus_enabled: bool = True
    sentry_enabled: bool = False
    sentry_dsn: str = ""
    rate_limit_enabled: bool = True

    model_config = SettingsConfigDict(env_file=".env", extra="allow")
```

### errors.py

Standardized error hierarchy. All services raise `SmartTradeError` subclasses:

```python
class SmartTradeError(Exception):
    code: str           # e.g., "VAL_001", "AUTH_004"
    message: str
    http_status: int    # 400, 401, 403, 404, 409, 422, 500

# Registered codes:
# VAL_001 — Validation error
# AUTH_001 — Invalid credentials
# AUTH_004 — Account suspended
# RISK_001 — Risk limit breach (order rejected)
# RISK_002 — Max positions reached
# BKR_001 — Broker API error
# BKR_002 — Broker session expired
# ORD_001 — Order not found
# ORD_002 — Order not cancellable
# SYS_001 — Internal system error
```

---

### database/

#### session.py

```python
# AsyncSessionFactory — used by all services
engine = create_async_engine(settings.database_url, pool_size=10, max_overflow=20)
AsyncSessionFactory = async_sessionmaker(engine, expire_on_commit=False)

# Dependency for FastAPI routes
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionFactory() as session:
        yield session
```

#### repository.py

Generic async repository. All service repositories extend this:

```python
class BaseRepository(Generic[T]):
    model: type[T]
    session: AsyncSession

    async def get(self, id: UUID, user_id: UUID | None = None) -> T | None
    async def list(self, user_id: UUID, filters: dict = {}, limit=50, offset=0) -> list[T]
    async def create(self, data: dict) -> T
    async def update(self, instance: T) -> T
    async def delete(self, id: UUID, user_id: UUID) -> None
    async def count(self, user_id: UUID, filters: dict = {}) -> int
```

Note: `user_id` parameter is mandatory on all read/write operations to enforce isolation. Passing `user_id=None` raises an assertion error in non-admin contexts.

#### transaction.py

```python
@asynccontextmanager
async def transaction(session: AsyncSession):
    """ACID transaction wrapper. Commits on success, rolls back on exception."""
    async with session.begin():
        yield session

# Usage:
async with transaction(db):
    order = await order_repo.create(...)
    trade = await trade_repo.create(...)
    position = await position_repo.update(...)
    # All-or-nothing
```

#### audit_models.py + audit_repository.py

Append-only audit trail. Records are never updated or deleted.

```python
class AuditRecord(Base):
    __tablename__ = "audit_log"
    id: UUID
    user_id: UUID
    entity_type: str    # "order", "trade", "user", etc.
    entity_id: UUID
    action: str         # "created", "filled", "cancelled"
    payload: dict       # JSON snapshot of entity at time of action
    created_at: datetime
    # NO updated_at — immutable by design
    __table_args__ = (
        # PostgreSQL-level: prevent DELETE/UPDATE via row security policy
    )
```

---

### events/

#### event_bus.py

Redis-backed pub/sub with Pydantic event schema validation:

```python
class EventBus:
    async def publish(self, topic: str, event: BaseEvent) -> None:
        payload = event.model_dump_json()
        await self.redis.publish(topic, payload)

    async def subscribe(
        self,
        topics: list[str],
        handler: Callable[[BaseEvent], Awaitable[None]]
    ) -> None:
        async for message in self.pubsub.listen():
            event = parse_event(message, topic_schema_map)
            await handler(event)
```

All events are Pydantic models inheriting from `BaseEvent`:

```python
class BaseEvent(BaseModel):
    event_id: UUID = Field(default_factory=uuid4)
    topic: str
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    user_id: UUID
```

---

### security/

#### jwt_utils.py

```python
def create_access_token(user_id, role, email) -> str:
    payload = {"sub": str(user_id), "role": role, "email": email,
               "iat": now, "exp": now + timedelta(minutes=15)}
    return jwt.encode(payload, settings.jwt_secret_key, algorithm="HS256")

def verify_token(token: str) -> TokenPayload:
    try:
        payload = jwt.decode(token, settings.jwt_secret_key, algorithms=["HS256"])
        return TokenPayload(**payload)
    except ExpiredSignatureError:
        raise SmartTradeError("AUTH_006", "Token expired")
    except InvalidTokenError:
        raise SmartTradeError("AUTH_006", "Invalid token")
```

#### rbac.py

YAML-driven role-based access control:

```python
class RBACEnforcer:
    def __init__(self, policy_path: str):
        self.policies = load_yaml(policy_path)

    def assert_allowed(self, role: str, route: str, method: str):
        allowed_routes = self.policies.get(role, [])
        if not any(matches(route, pattern) for pattern in allowed_routes):
            raise SmartTradeError("AUTH_004", f"Role {role} cannot access {method} {route}")
```

#### cryptography.py

AES-256-GCM for broker credential encryption:

```python
def encrypt(plaintext: str, key: bytes) -> str:
    # Returns: base64(nonce + ciphertext + tag)

def decrypt(ciphertext: str, key: bytes) -> str:
    # Decodes base64, extracts nonce, decrypts, verifies tag
```

---

### resilience/

#### retry.py

```python
@retry(max_attempts=3, backoff_factor=2, exceptions=(BrokerAPIError, aiohttp.ClientError))
async def call_broker_api():
    ...
# Delays: 1s, 2s, 4s between attempts; full jitter applied
```

#### circuit_breaker.py

```python
class CircuitBreaker:
    state: "CLOSED" | "OPEN" | "HALF_OPEN"
    failure_count: int
    failure_threshold: int = 5
    recovery_timeout: int = 30  # seconds

    async def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time_since_opened > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError()
        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
```

#### distributed_rate_limiter.py

Redis token-bucket rate limiter:

```python
class DistributedRateLimiter:
    async def check_and_consume(self, key: str, limit: int, window_seconds: int) -> bool:
        # Lua script for atomic check-and-decrement
        # Returns True if request allowed, False if rate limited
```

---

### observability/

#### metrics.py

```python
# Prometheus metrics registry
http_requests_total = Counter("http_requests_total", ["method", "path", "status"])
http_request_duration = Histogram("http_request_duration_seconds", ["path"])
orders_placed = Counter("orders_placed_total", ["broker", "status"])
active_websockets = Gauge("websocket_connections_active")

def track_request(method, path, status, duration):
    http_requests_total.labels(method=method, path=path, status=status).inc()
    http_request_duration.labels(path=path).observe(duration)
```

#### tracing.py

```python
# OpenTelemetry setup
def setup_tracing(service_name: str, endpoint: str):
    tracer_provider = TracerProvider(resource=Resource.create({"service.name": service_name}))
    tracer_provider.add_span_processor(BatchSpanProcessor(OTLPExporter(endpoint)))
    trace.set_tracer_provider(tracer_provider)

# Usage:
with tracer.start_as_current_span("place_order") as span:
    span.set_attribute("user.id", str(user_id))
    span.set_attribute("order.symbol", symbol)
    result = await broker.place_order(...)
```

---

### idempotency.py

```python
class IdempotencyChecker:
    async def get(self, key: str) -> dict | None:
        value = await self.redis.get(f"idempotency:{key}")
        return json.loads(value) if value else None

    async def set(self, key: str, result: dict, ttl=86400) -> None:
        await self.redis.setex(f"idempotency:{key}", ttl, json.dumps(result))
```

---

### locking.py

```python
class DistributedLock:
    async def acquire(self, key: str, timeout=10) -> bool:
        # SET NX EX with random token (prevents accidental release)

    async def release(self, key: str) -> None:
        # Lua: if value == our_token: DEL key

# Usage — prevent concurrent position updates for same user:
async with DistributedLock(f"position:update:{user_id}"):
    position = await repo.get(position_id)
    position.net_qty += fill_qty
    await repo.update(position)
```

---

## 3. Usage Pattern in Services

Every service `config.py`:

```python
from smarttrade_common.config import CommonSettings

class BASSettings(CommonSettings):
    service_name: str = "broker-adapter-service"
    fyers_app_id: str
    fyers_app_secret: str
    mock_service_url: str = "http://localhost:8002"
    mds_url: str = "http://localhost:8004"
```

Every service `main.py`:

```python
from smarttrade_common.app_factory import create_app

app = create_app(
    title="Broker Adapter Service",
    settings=settings,
    routers=[orders_router, portfolio_router, ...],
    lifespan=lifespan,
    public_paths=["/oauth/callback"],
)
```

---

## 4. Testing `smarttrade-common`

40+ test files covering every module. Key tests:

| Test | Coverage |
|------|---------|
| `test_security.py` | JWT sign/verify, bcrypt |
| `test_rbac.py` | Policy enforcement |
| `test_event_bus.py` | Publish/subscribe |
| `test_idempotency.py` | Check-and-set semantics |
| `test_resilience.py` | Retry, circuit breaker, timeout |
| `test_locking.py` | Distributed lock acquire/release |
| `test_transactional.py` | ACID transaction rollback |
| `test_audit.py` | Append-only audit records |
| `test_rate_limit.py` | Token bucket enforcement |
| `test_middleware.py` | Request ID injection |
| `test_filtering.py` | Query filter DSL |
