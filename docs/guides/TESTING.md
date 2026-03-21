# How to: Test SmartTrade

---

## Testing Philosophy

| Layer | Coverage Target | Scope |
|-------|----------------|-------|
| Unit | 90%+ on financial logic | Pure business logic, no I/O |
| Integration | All critical paths | Single-service with real DB (SQLite) |
| E2E | All user workflows | Cross-service (in `smarttrade-tests` repo) |

**Rule:** Never mock the database in integration tests. Mocked DB tests pass while real migration failures go undetected (we learned this the hard way).

**Test placement:**
- Per-service `tests/` = unit + single-service integration
- `smarttrade-tests` repo = all E2E, Postman, Playwright

---

## Running Tests

### Run All Tests in a Service

```bash
cd broker-adapter-service
uv run pytest                           # all tests
uv run pytest -m unit                   # unit tests only
uv run pytest -m "not slow"             # skip slow integration tests
uv run pytest --maxfail=1               # stop at first failure (default)
uv run pytest -x -v                     # verbose, stop on first failure
```

### Coverage Report

```bash
uv run pytest --cov src/ --cov-report=html --cov-report=term-missing
open htmlcov/index.html
```

### Single Test or File

```bash
uv run pytest tests/unit/test_order_handler.py
uv run pytest tests/unit/test_order_handler.py::test_place_order_success
uv run pytest tests/unit/test_order_handler.py -k "financial"   # name filter
```

---

## Test Structure

### Unit Test Pattern

Unit tests use mocked repositories and event bus. They test business logic only.

```python
# tests/unit/test_order_handler.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from decimal import Decimal

@pytest.fixture
def mock_repos():
    return {
        "order_repo": AsyncMock(),
        "trade_repo": AsyncMock(),
        "position_repo": AsyncMock(),
    }

@pytest.fixture
def order_handler(mock_repos):
    return OrderHandler(
        order_repo=mock_repos["order_repo"],
        trade_repo=mock_repos["trade_repo"],
        risk_engine=AsyncMock(),
        event_bus=AsyncMock(),
        adapter_manager=AsyncMock(),
        idempotency=AsyncMock(get=AsyncMock(return_value=None)),
    )

@pytest.mark.asyncio
async def test_place_order_success(order_handler, mock_repos):
    # Arrange
    mock_repos["order_repo"].create.return_value = Order(
        id=uuid4(), status=OrderStatus.OPEN
    )
    order_handler.adapter_manager.get_plugin.return_value.place_order.return_value = \
        BrokerOrderResponse(broker_order_id="BR123", status="OPEN")

    # Act
    result = await order_handler.place_order(
        user_id=uuid4(),
        request=PlaceOrderRequest(
            symbol="NSE:RELIANCE-EQ",
            side="BUY",
            qty=10,
            order_type="MARKET",
            product_type="INTRADAY"
        )
    )

    # Assert
    assert result.status == OrderStatus.OPEN
    mock_repos["order_repo"].create.assert_called_once()

@pytest.mark.asyncio
async def test_place_order_decimal_correctness(order_handler, mock_repos):
    """Financial calculations must use Decimal, never float."""
    # This test will fail if any float is introduced in price calculations
    order = PlaceOrderRequest(
        symbol="NSE:RELIANCE-EQ",
        side="BUY",
        qty=10,
        price=Decimal("2850.75"),
        order_type="LIMIT"
    )
    # Verify Decimal is preserved through the handler
    captured_args = mock_repos["order_repo"].create.call_args
    assert isinstance(captured_args[1]["price"], Decimal)
```

### Integration Test Pattern

Integration tests use real SQLite (or PostgreSQL) and test the full service layer.

```python
# tests/integration/test_order_settlement_workflow.py
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

@pytest.fixture(scope="session")
async def db_engine():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()

@pytest.fixture
async def db(db_engine):
    async with AsyncSession(db_engine) as session:
        yield session
        await session.rollback()

@pytest.mark.asyncio
async def test_order_creates_settlement_record(db):
    # Place an order
    order = await order_handler.place_order(user_id=test_user_id, request=sample_order())

    # Simulate fill
    await order_handler.on_fill(FillEvent(
        broker_order_id=order.broker_order_id,
        fill_qty=10,
        fill_price=Decimal("2855.00")
    ))

    # Verify settlement record created
    settlements = await settlement_repo.get_pending(test_user_id)
    assert len(settlements) == 1
    assert settlements[0].trade_date == date.today()
    assert settlements[0].status == "pending"
```

### Route Test Pattern

Route tests verify HTTP request/response format and auth enforcement.

```python
# tests/routes/test_order_routes.py
@pytest.fixture
async def client(app):
    async with AsyncClient(app=app, base_url="http://test") as c:
        yield c

@pytest.fixture
def auth_headers(test_user):
    token = create_access_token(test_user.id, test_user.role, test_user.email)
    return {"Authorization": f"Bearer {token}"}

async def test_post_order_returns_201(client, auth_headers, mock_order_handler):
    mock_order_handler.place_order.return_value = sample_order()

    response = await client.post(
        "/api/v1/orders",
        json={
            "symbol": "NSE:RELIANCE-EQ",
            "side": "BUY",
            "qty": 10,
            "order_type": "MARKET",
            "product_type": "INTRADAY"
        },
        headers=auth_headers,
    )

    assert response.status_code == 201
    body = response.json()
    assert "id" in body
    assert body["status"] == "OPEN"

async def test_post_order_requires_auth(client):
    response = await client.post("/api/v1/orders", json={})
    assert response.status_code == 401

async def test_post_order_validates_request(client, auth_headers):
    response = await client.post(
        "/api/v1/orders",
        json={"symbol": "INVALID"},  # missing required fields
        headers=auth_headers,
    )
    assert response.status_code == 422
```

---

## Fixtures and Conftest

Each service has a `tests/conftest.py` with shared fixtures:

```python
# tests/conftest.py
@pytest.fixture(scope="session")
def event_loop():
    """Use a single event loop for the test session."""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture
def test_user_id() -> UUID:
    return UUID("12345678-1234-5678-1234-567812345678")

@pytest.fixture
def sample_order(test_user_id) -> Order:
    return Order(
        id=uuid4(),
        user_id=test_user_id,
        symbol="NSE:RELIANCE-EQ",
        side=OrderSide.BUY,
        qty=10,
        price=Decimal("2850.00"),
        status=OrderStatus.OPEN,
    )
```

---

## Testing Financial Logic

### Decimal Tests

```python
def test_pnl_calculation_uses_decimal():
    """P&L must never use float arithmetic."""
    buy_price = Decimal("2850.75")
    sell_price = Decimal("2900.25")
    qty = 10

    pnl = (sell_price - buy_price) * qty
    assert pnl == Decimal("495.00")
    assert isinstance(pnl, Decimal)
    # If this were float: 2900.25 - 2850.75 = 49.49999999999977 (wrong)

def test_division_uses_decimal_100():
    """Division by 100 must use Decimal("100") not 100.0."""
    risk_pct = Decimal("2.5")
    capital = Decimal("100000")

    # CORRECT
    risk_amount = capital * risk_pct / Decimal("100")
    assert risk_amount == Decimal("2500.0")

    # This would fail if using 100.0:
    # risk_amount = capital * risk_pct / 100.0  → mixed types
```

### ACID Tests

```python
async def test_order_and_trade_atomic(db):
    """If trade creation fails, order should not be created."""
    with pytest.raises(SomeError):
        async with transaction(db):
            order = await order_repo.create(sample_order_data())
            # Force failure:
            raise SomeError("simulated failure")

    # Verify rollback — order should not exist
    result = await order_repo.get(order.id)
    assert result is None
```

---

## Testing PIE / Risk Engine

### Immutable RuntimeRiskInput

`RuntimeRiskInput` is a frozen dataclass — tests must create new instances, not mutate fixtures:

```python
# CORRECT
def make_risk_input(**overrides):
    defaults = {
        "user_id": uuid4(),
        "realized_pnl": Decimal("-1000"),
        "unrealized_pnl": Decimal("-200"),
        "open_position_count": 3,
        "account_capital": Decimal("100000"),
        "pending_order": None,
        "positions": [],
        "evaluated_at": datetime.utcnow(),
    }
    return RuntimeRiskInput(**{**defaults, **overrides})

async def test_daily_loss_breach():
    rule = MaxDailyLossRule(params={"default_limit": -5000.0})
    input = make_risk_input(realized_pnl=Decimal("-4000"), unrealized_pnl=Decimal("-2000"))
    result = rule.evaluate(input)
    assert not result.passed

# WRONG — frozen dataclasses cannot be modified
async def test_wrong_way(risk_input_fixture):
    risk_input_fixture.realized_pnl = Decimal("-9000")  # FrozenInstanceError!
```

---

## Test Markers

```ini
# pytest.ini
[pytest]
asyncio_mode = auto
markers =
    unit: Pure unit tests (no I/O)
    integration: Integration tests (real DB, may be slow)
    slow: Very slow tests (skip in dev loop)
    e2e: Cross-service E2E (run in smarttrade-tests only)
```

Usage:
```bash
uv run pytest -m unit            # fast feedback loop
uv run pytest -m "not slow"      # skip slow tests
uv run pytest -m integration     # full integration suite
```

---

## CI Pipeline

Tests run automatically on every PR:

```yaml
# .github/workflows/ci.yml (per service)
- name: Run tests
  run: |
    uv sync --extra dev
    uv run pytest --cov src/ --cov-fail-under=80
```

Coverage requirement: 80% minimum (90%+ target for financial logic).
