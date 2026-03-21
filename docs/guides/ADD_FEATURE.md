# How to: Add a New Feature

This guide explains the standard process for implementing features in SmartTrade, from design to production.

---

## Step 0: Choose the Right Mode

**FAST mode** — for bug fixes, small features, refactors
```
planner → executor → reviewer
No design doc required.
```

**FULL mode** — for new API, new schema, new service, event-driven workflow, risk/compliance logic
```
planner → designer → executor → reviewer
Design doc REQUIRED (place in smarttrade-project/design/<service>/).
```

When in doubt, use FULL mode.

---

## Step 1: Design Doc (FULL mode only)

Create `smarttrade-project/design/<service>/<FEATURE_NAME>_DESIGN.md` with:

1. **Overview** — what problem it solves, what it doesn't do
2. **Data Model** — new/modified DB tables (Python model classes)
3. **API Endpoints** — method, path, request/response schemas
4. **Service Logic** — pseudocode for the service layer
5. **Events** — events published/consumed
6. **Test Coverage** — what tests will be written

Get design reviewed before writing code.

---

## Step 2: Implementation Order (MANDATORY)

Always follow this order:

```
Models → Schemas → Repository → Services → Routes → Events
```

Never skip layers. Never put business logic in routes. Never call repositories from routes.

### Models (SQLAlchemy)

```python
# broker-adapter-service/src/broker_adapter_service/models/my_feature.py
from smarttrade_common.database.models import Base
from sqlalchemy import Column, String, Boolean
from sqlalchemy.dialects.postgresql import UUID

class MyFeature(Base):
    __tablename__ = "my_features"
    id = Column(UUID, primary_key=True, default=uuid4)
    user_id = Column(UUID, nullable=False, index=True)   # ALWAYS indexed
    name = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
```

### Schemas (Pydantic v2)

```python
# broker-adapter-service/src/broker_adapter_service/schemas/my_feature.py
from pydantic import BaseModel, UUID4

class MyFeatureCreate(BaseModel):
    name: str
    is_active: bool = True

class MyFeatureResponse(BaseModel):
    id: UUID4
    user_id: UUID4
    name: str
    is_active: bool
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)
```

### Repository

```python
# broker-adapter-service/src/broker_adapter_service/repositories/my_feature_repo.py
from smarttrade_common.database.repository import BaseRepository
from broker_adapter_service.models.my_feature import MyFeature

class MyFeatureRepository(BaseRepository[MyFeature]):
    model = MyFeature

    async def get_active(self, user_id: UUID) -> list[MyFeature]:
        return await self.list(user_id=user_id, filters={"is_active": True})
```

### Service

```python
# broker-adapter-service/src/broker_adapter_service/services/my_feature_service.py
from smarttrade_common.errors import SmartTradeError

class MyFeatureService:
    def __init__(self, repo: MyFeatureRepository, event_bus: EventBus):
        self.repo = repo
        self.event_bus = event_bus

    async def create(self, user_id: UUID, data: MyFeatureCreate) -> MyFeature:
        # Business logic here — NOT in routes
        existing = await self.repo.get_active(user_id)
        if len(existing) >= MAX_FEATURES:
            raise SmartTradeError("VAL_001", "Maximum feature count reached")

        feature = await self.repo.create({
            "user_id": user_id,
            "name": data.name,
            "is_active": data.is_active,
        })
        await self.event_bus.publish("feature.created", FeatureCreatedEvent(
            user_id=user_id,
            feature_id=feature.id
        ))
        return feature
```

### Routes (thin — no business logic)

```python
# broker-adapter-service/src/broker_adapter_service/api/routes_my_feature.py
router = APIRouter(prefix="/api/v1/my_feature", tags=["my_feature"])

@router.post("", response_model=MyFeatureResponse, status_code=201)
async def create_feature(
    data: MyFeatureCreate,
    current_user: User = Depends(get_current_user),
    service: MyFeatureService = Depends(get_my_feature_service),
):
    feature = await service.create(current_user.id, data)
    return MyFeatureResponse.model_validate(feature)
```

### Register Router in main.py

```python
# broker-adapter-service/src/broker_adapter_service/main.py
from broker_adapter_service.api.routes_my_feature import router as my_feature_router

app = create_app(
    ...,
    routers=[..., my_feature_router],
)
```

### Alembic Migration

```bash
cd broker-adapter-service
uv run alembic revision --autogenerate -m "add_my_feature_table"
# Review the generated migration in alembic/versions/
uv run alembic upgrade head
```

---

## Step 3: Write Tests First (TDD Preferred)

### Unit Test (pure business logic)

```python
# tests/unit/test_my_feature_service.py
@pytest.fixture
def mock_repo():
    return AsyncMock(spec=MyFeatureRepository)

@pytest.fixture
def service(mock_repo):
    return MyFeatureService(repo=mock_repo, event_bus=AsyncMock())

async def test_create_feature_success(service, mock_repo):
    mock_repo.get_active.return_value = []
    mock_repo.create.return_value = MyFeature(id=uuid4(), name="test", is_active=True)

    result = await service.create(user_id=uuid4(), data=MyFeatureCreate(name="test"))

    assert result.name == "test"
    mock_repo.create.assert_called_once()

async def test_create_feature_max_limit(service, mock_repo):
    mock_repo.get_active.return_value = [MyFeature() for _ in range(MAX_FEATURES)]

    with pytest.raises(SmartTradeError) as exc:
        await service.create(user_id=uuid4(), data=MyFeatureCreate(name="test"))
    assert exc.value.code == "VAL_001"
```

### Route Test (HTTP layer)

```python
# tests/routes/test_my_feature_routes.py
async def test_create_feature_returns_201(client, auth_headers):
    response = await client.post(
        "/api/v1/my_feature",
        json={"name": "test_feature"},
        headers=auth_headers,
    )
    assert response.status_code == 201
    body = response.json()
    assert body["name"] == "test_feature"
    assert "id" in body

async def test_create_feature_requires_auth(client):
    response = await client.post("/api/v1/my_feature", json={"name": "test"})
    assert response.status_code == 401
```

---

## Step 4: Financial Correctness Checklist

Before any PR that touches money:

- [ ] All monetary values use `Decimal` — never `float` or `int`
- [ ] Decimal literals use strings: `Decimal("100")` not `Decimal(100)` or `100.0`
- [ ] All DB mutations are wrapped in `async with transaction(db)`
- [ ] No partial writes possible (all-or-nothing)
- [ ] Every DB query includes `user_id` filter
- [ ] No cross-user data can be accessed

---

## Step 5: Event Design (if feature emits events)

Add event schema to `smarttrade-common`:

```python
# smarttrade-common/src/smarttrade_common/schemas/services/trading/my_events.py
class MyFeatureCreatedEvent(BaseEvent):
    topic: str = "feature.created"
    feature_id: UUID
    name: str
```

Consumers subscribe in the relevant service's `event_consumers.py`.

---

## Step 6: Frontend Integration (if feature has UI)

Follow the standard pattern:

1. Add API function to `src/api/`
2. Add store state to relevant Zustand store
3. Add hook that combines API call + store update
4. Build component(s) using the hook
5. Register component in `PANEL_MAP` if it's a dashboard panel

---

## Step 7: PR Checklist

Before creating PR:

- [ ] All tests pass: `uv run pytest`
- [ ] No lint errors: `uv run ruff check src/`
- [ ] No type errors: `uv run mypy src/`
- [ ] Migration reviewed (no destructive changes)
- [ ] Design doc updated if implementation diverged from design
- [ ] PR < 500 lines (split if larger)
- [ ] PR title follows: `feat(service): description` or `fix(service): description`

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why | Fix |
|-------------|-----|-----|
| Business logic in route handler | Routes are untestable, not reusable | Move to service layer |
| `float` for money | Floating point errors accumulate | Use `Decimal` |
| Missing `user_id` filter in query | Cross-user data leak | Always filter by user_id |
| `requests.get()` (sync HTTP) | Blocks event loop | Use `httpx.AsyncClient` |
| `db.add(); await db.commit()` without transaction | Race condition + partial writes | Use `async with transaction(db)` |
| Raw `dict` across service boundaries | No validation | Use Pydantic schemas |
| Bare `except Exception:` | Swallows errors silently | Catch specific exceptions + log |
