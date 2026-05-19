# E2E Test Execution Plan — Implementation Roadmap

**Status:** Ready for Execution  
**Date:** 2026-05-16  
**Scope:** E2E test framework rework execution plan  
**Related Design:** [2026-05-16-e2e-test-rework-v1.md](./2026-05-16-e2e-test-rework-v1.md)

---

## Executive Summary

This execution plan provides a detailed roadmap for implementing the E2E test rework to align with the v4.0 stateless architecture. The plan is organized into 7 phases over 10 weeks, with clear deliverables, dependencies, and success criteria for each phase.

**Overall Timeline:** 10 weeks  
**Total Effort:** ~200 hours (20 hours/week)  
**Team Size:** 1-2 engineers  
**Risk Level:** Medium (depends on service implementation completion)

---

## Prerequisites

### Must Complete Before Starting

1. **Service Implementations**
   - ✅ BAS stateless implementation complete
   - ✅ Consolidated `order.updated` event schema implemented
   - ✅ Outbox pattern implemented in BAS
   - ✅ InstrumentSyncService implemented in all services
   - ✅ PBS synchronous API implemented
   - ✅ Strategy advisory-only behavior implemented

2. **Infrastructure**
   - ✅ Redis Streams configured for event bus
   - ✅ Outbox table created in PostgreSQL
   - ✅ Service-scoped consumer groups configured
   - ✅ Broker APIs accessible for E2E tests

3. **Environment**
   - ✅ E2E test environment provisioned
   - ✅ Test broker credentials configured
   - ✅ Network connectivity between services
   - ✅ Docker compose for local testing

### Verification Checklist

Before starting Phase 1, verify:
- [ ] All services are running and healthy
- [ ] Event schemas are finalized and documented
- [ ] Outbox processor is running
- [ ] Instrument sync is working
- [ ] Broker APIs are accessible from test environment
- [ ] Redis Streams are writable and readable
- [ ] Test accounts are provisioned

---

## Phase 1: Infrastructure Updates (Week 1-2)

**Objective:** Replace obsolete components and add new infrastructure for stateless architecture

**Effort:** 40 hours (2 weeks)

### 1.1 Remove bas_ws_client.py (Day 1)

**File:** `smarttrade-tests/e2e/clients/bas_ws_client.py`

**Action:** DELETE file

**Steps:**
```bash
cd smarttrade-tests/e2e/clients
rm bas_ws_client.py
```

**Verification:**
- File deleted
- No imports reference `bas_ws_client` in codebase
- Update `clients/__init__.py` to remove export

**Dependencies:** None

**Deliverable:** `bas_ws_client.py` removed

---

### 1.2 Add broker_state_client.py (Day 2-3)

**File:** `smarttrade-tests/e2e/clients/broker_state_client.py` (NEW)

**Purpose:** Query broker directly for order/position/trade state

**Implementation:**

```python
"""
Broker state client for direct broker queries.

Supports multiple broker types (Fyers, PBS) with unified interface.
Broker is the single source of truth for order/position/trade state.
"""

import logging
from abc import ABC, abstractmethod
from typing import Optional
import httpx

log = logging.getLogger(__name__)


class BrokerStateClient(ABC):
    """Abstract base class for broker state clients."""
    
    @abstractmethod
    async def get_order_state(self, broker_id: str, account_id: str, order_id: str) -> dict:
        """Get order state from broker."""
        pass
    
    @abstractmethod
    async def get_position_state(self, broker_id: str, account_id: str, instrument_id: str) -> dict:
        """Get position state from broker."""
        pass
    
    @abstractmethod
    async def get_trade_state(self, broker_id: str, account_id: str, order_id: str) -> dict:
        """Get trade state from broker."""
        pass
    
    @abstractmethod
    async def get_account_state(self, broker_id: str, account_id: str) -> dict:
        """Get account state from broker."""
        pass


class FyersStateClient(BrokerStateClient):
    """Fyers broker state client."""
    
    def __init__(self, api_url: str, token: str, timeout: float = 10.0):
        self.api_url = api_url.rstrip("/")
        self.token = token
        self.timeout = timeout
        self.client: Optional[httpx.AsyncClient] = None
    
    async def __aenter__(self):
        self.client = httpx.AsyncClient(timeout=self.timeout)
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.client:
            await self.client.aclose()
    
    async def get_order_state(self, broker_id: str, account_id: str, order_id: str) -> dict:
        """Query Fyers API for order state."""
        if not self.client:
            raise RuntimeError("Client not connected. Use async context manager.")
        
        # Implementation depends on Fyers API specifics
        # This is a placeholder - actual API calls need to be implemented
        url = f"{self.api_url}/orders/{order_id}"
        headers = {"Authorization": f"Bearer {self.token}"}
        
        response = await self.client.get(url, headers=headers)
        response.raise_for_status()
        return response.json()
    
    async def get_position_state(self, broker_id: str, account_id: str, instrument_id: str) -> dict:
        """Query Fyers API for position state."""
        if not self.client:
            raise RuntimeError("Client not connected. Use async context manager.")
        
        url = f"{self.api_url}/positions"
        params = {"instrument_id": instrument_id}
        headers = {"Authorization": f"Bearer {self.token}"}
        
        response = await self.client.get(url, params=params, headers=headers)
        response.raise_for_status()
        
        positions = response.json()
        # Find matching position
        for pos in positions:
            if pos.get("instrument_id") == instrument_id:
                return pos
        return {}
    
    async def get_trade_state(self, broker_id: str, account_id: str, order_id: str) -> dict:
        """Query Fyers API for trade state."""
        if not self.client:
            raise RuntimeError("Client not connected. Use async context manager.")
        
        url = f"{self.api_url}/trades"
        params = {"order_id": order_id}
        headers = {"Authorization": f"Bearer {self.token}"}
        
        response = await self.client.get(url, params=params, headers=headers)
        response.raise_for_status()
        
        trades = response.json()
        for trade in trades:
            if trade.get("order_id") == order_id:
                return trade
        return {}
    
    async def get_account_state(self, broker_id: str, account_id: str) -> dict:
        """Query Fyers API for account state."""
        if not self.client:
            raise RuntimeError("Client not connected. Use async context manager.")
        
        url = f"{self.api_url}/account"
        headers = {"Authorization": f"Bearer {self.token}"}
        
        response = await self.client.get(url, headers=headers)
        response.raise_for_status()
        return response.json()


class PBSStateClient(BrokerStateClient):
    """Paper Broker Service state client."""
    
    def __init__(self, base_url: str, token: str, timeout: float = 10.0):
        self.base_url = base_url.rstrip("/")
        self.token = token
        self.timeout = timeout
        self.client: Optional[httpx.AsyncClient] = None
    
    async def __aenter__(self):
        self.client = httpx.AsyncClient(timeout=self.timeout)
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.client:
            await self.client.aclose()
    
    async def get_order_state(self, broker_id: str, account_id: str, order_id: str) -> dict:
        """Query PBS internal API for order state."""
        if not self.client:
            raise RuntimeError("Client not connected. Use async context manager.")
        
        url = f"{self.base_url}/api/v1/orders/{order_id}"
        headers = {"Authorization": f"Bearer {self.token}"}
        
        response = await self.client.get(url, headers=headers)
        response.raise_for_status()
        return response.json()
    
    async def get_position_state(self, broker_id: str, account_id: str, instrument_id: str) -> dict:
        """Query PBS internal API for position state."""
        if not self.client:
            raise RuntimeError("Client not connected. Use async context manager.")
        
        url = f"{self.base_url}/api/v1/positions"
        params = {"instrument_id": instrument_id}
        headers = {"Authorization": f"Bearer {self.token}"}
        
        response = await self.client.get(url, params=params, headers=headers)
        response.raise_for_status()
        
        positions = response.json()
        for pos in positions:
            if pos.get("instrument_id") == instrument_id:
                return pos
        return {}
    
    async def get_trade_state(self, broker_id: str, account_id: str, order_id: str) -> dict:
        """Query PBS internal API for trade state."""
        if not self.client:
            raise RuntimeError("Client not connected. Use async context manager.")
        
        url = f"{self.base_url}/api/v1/trades"
        params = {"order_id": order_id}
        headers = {"Authorization": f"Bearer {self.token}"}
        
        response = await self.client.get(url, params=params, headers=headers)
        response.raise_for_status()
        
        trades = response.json()
        for trade in trades:
            if trade.get("order_id") == order_id:
                return trade
        return {}
    
    async def get_account_state(self, broker_id: str, account_id: str) -> dict:
        """Query PBS internal API for account state."""
        if not self.client:
            raise RuntimeError("Client not connected. Use async context manager.")
        
        url = f"{self.base_url}/api/v1/accounts/{account_id}"
        headers = {"Authorization": f"Bearer {self.token}"}
        
        response = await self.client.get(url, headers=headers)
        response.raise_for_status()
        return response.json()


def create_broker_state_client(broker_type: str, base_url: str, token: str, timeout: float = 10.0) -> BrokerStateClient:
    """
    Factory function to create broker-specific state client.
    
    Args:
        broker_type: Type of broker ("fyers" or "pbs")
        base_url: Base URL for broker API
        token: Authentication token
        timeout: Request timeout in seconds
    
    Returns:
        BrokerStateClient instance
    
    Raises:
        ValueError: If broker_type is not supported
    """
    broker_type = broker_type.lower()
    
    if broker_type == "fyers":
        return FyersStateClient(base_url, token, timeout)
    elif broker_type == "pbs":
        return PBSStateClient(base_url, token, timeout)
    else:
        raise ValueError(f"Unsupported broker type: {broker_type}")
```

**Verification:**
- File created with correct structure
- Unit tests for both Fyers and PBS clients
- Factory function works correctly

**Dependencies:** None

**Deliverable:** `broker_state_client.py` with tests

---

### 1.3 Add redis_event_collector.py (Day 4-5)

**File:** `smarttrade-tests/e2e/harness/redis_event_collector.py` (NEW)

**Purpose:** Collect events directly from Redis Streams

**Implementation:**

```python
"""
Redis stream event collector for E2E testing.

Collects events directly from Redis Streams (bypassing service WebSockets).
Uses consumer groups for reliable event consumption with idempotency.
"""

import asyncio
import json
import logging
from typing import Optional, List
import redis.asyncio as redis

log = logging.getLogger(__name__)


class RedisEventCollector:
    """
    Async event collector using Redis Streams.
    
    Collects events from Redis Streams using consumer groups for reliable
    consumption. Supports per-order event collection with timeout support.
    """
    
    def __init__(
        self,
        redis_url: str,
        consumer_group: str,
        consumer_name: Optional[str] = None,
        timeout: float = 30.0,
    ):
        """
        Initialize RedisEventCollector.
        
        Args:
            redis_url: Redis connection URL (e.g., "redis://localhost:6379")
            consumer_group: Consumer group name for stream consumption
            consumer_name: Consumer name (auto-generated if None)
            timeout: Default timeout for wait operations
        """
        self.redis_url = redis_url
        self.consumer_group = consumer_group
        self.consumer_name = consumer_name or f"consumer-{asyncio.current_task().get_name()}"
        self.timeout = timeout
        
        self.redis_client: Optional[redis.Redis] = None
        self.queues: dict[str, asyncio.Queue] = {}
        self.events: dict[str, List[dict]] = {}
        self._subscribed_streams: set[str] = set()
        self._running = False
        self._reader_task: Optional[asyncio.Task] = None
    
    async def __aenter__(self):
        """Async context manager entry."""
        await self.connect()
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Async context manager exit."""
        await self.disconnect()
    
    async def connect(self) -> None:
        """Connect to Redis and initialize consumer groups."""
        self.redis_client = redis.from_url(self.redis_url, decode_responses=True)
        
        # Test connection
        await self.redis_client.ping()
        log.info(f"Connected to Redis at {self.redis_url}")
    
    async def disconnect(self) -> None:
        """Disconnect from Redis and cleanup."""
        self._running = False
        
        if self._reader_task:
            self._reader_task.cancel()
            try:
                await self._reader_task
            except asyncio.CancelledError:
                pass
        
        if self.redis_client:
            await self.redis_client.close()
            self.redis_client = None
        
        log.info("Disconnected from Redis")
    
    async def subscribe_to_streams(self, stream_patterns: List[str]) -> None:
        """
        Subscribe to Redis Streams and create consumer groups if needed.
        
        Args:
            stream_patterns: List of stream names/patterns to subscribe
        """
        if not self.redis_client:
            raise RuntimeError("Not connected. Call connect() first.")
        
        for stream in stream_patterns:
            try:
                # Try to create consumer group (ignore if exists)
                await self.redis_client.xgroup_create(
                    stream,
                    self.consumer_group,
                    id="0",
                    mkstream=True,
                )
                log.info(f"Created consumer group {self.consumer_group} for stream {stream}")
            except redis.ResponseError as e:
                if "BUSYGROUP" not in str(e):
                    raise
                log.debug(f"Consumer group already exists for stream {stream}")
            
            self._subscribed_streams.add(stream)
        
        # Start reader task
        self._running = True
        self._reader_task = asyncio.create_task(self._reader_loop())
        log.info(f"Subscribed to streams: {stream_patterns}")
    
    async def _reader_loop(self) -> None:
        """Background task to read events from Redis Streams."""
        while self._running:
            try:
                if not self._subscribed_streams:
                    await asyncio.sleep(0.1)
                    continue
                
                # Read from all subscribed streams
                streams_dict = {stream: ">" for stream in self._subscribed_streams}
                
                messages = await self.redis_client.xreadgroup(
                    groupname=self.consumer_group,
                    consumername=self.consumer_name,
                    streams=streams_dict,
                    count=10,
                    block=1000,  # 1 second block
                )
                
                for stream, stream_messages in messages:
                    for message_id, fields in stream_messages:
                        await self._process_message(stream, message_id, fields)
                
            except asyncio.CancelledError:
                break
            except Exception as e:
                log.error(f"Error in reader loop: {e}")
                await asyncio.sleep(1)
    
    async def _process_message(self, stream: str, message_id: str, fields: dict) -> None:
        """
        Process a single message from Redis Stream.
        
        Args:
            stream: Stream name
            message_id: Message ID
            fields: Message fields (event data)
        """
        try:
            # Parse event data
            event_data = json.loads(fields.get("data", "{}"))
            event_type = event_data.get("type", stream)
            order_id = event_data.get("order_id")
            
            if not order_id:
                log.debug(f"Message without order_id: {message_id}")
                await self.redis_client.xack(stream, self.consumer_group, message_id)
                return
            
            # Add to event log
            if order_id not in self.events:
                self.events[order_id] = []
            
            event = {
                "stream": stream,
                "message_id": message_id,
                "type": event_type,
                "data": event_data,
                "timestamp": fields.get("timestamp"),
            }
            self.events[order_id].append(event)
            
            # Add to queue for waiting
            if order_id not in self.queues:
                self.queues[order_id] = asyncio.Queue()
            
            try:
                self.queues[order_id].put_nowait(event)
            except asyncio.QueueFull:
                log.warning(f"Queue full for order_id={order_id}")
            
            # Acknowledge message
            await self.redis_client.xack(stream, self.consumer_group, message_id)
            
            log.debug(f"Processed event | stream={stream} | order_id={order_id} | type={event_type}")
        
        except Exception as e:
            log.error(f"Error processing message: {e}")
    
    async def wait_for_event(
        self,
        order_id: str,
        event_type: str,
        timeout: Optional[float] = None,
    ) -> dict:
        """
        Wait for a specific event type for an order.
        
        Args:
            order_id: Order ID to wait for
            event_type: Event type to wait for
            timeout: Maximum wait time in seconds (uses default if None)
        
        Returns:
            Event dictionary when found
        
        Raises:
            TimeoutError: If event not found within timeout
        """
        timeout = timeout or self.timeout
        start_time = asyncio.get_event_loop().time()
        
        # Check existing events
        if order_id in self.events:
            for event in reversed(self.events[order_id]):
                if event.get("type") == event_type:
                    return event
        
        # Wait for new event
        if order_id not in self.queues:
            self.queues[order_id] = asyncio.Queue()
        
        while True:
            try:
                remaining_timeout = timeout - (asyncio.get_event_loop().time() - start_time)
                if remaining_timeout <= 0:
                    raise TimeoutError(f"Event {event_type} not found for order_id={order_id}")
                
                event = await asyncio.wait_for(
                    self.queues[order_id].get(),
                    timeout=remaining_timeout,
                )
                
                if event.get("type") == event_type:
                    return event
            
            except asyncio.TimeoutError:
                raise TimeoutError(f"Event {event_type} not found for order_id={order_id}")
    
    async def wait_for_completion(
        self,
        order_id: str,
        timeout: Optional[float] = None,
    ) -> List[dict]:
        """
        Wait for order to reach terminal status.
        
        Terminal statuses: FILLED, CANCELLED, REJECTED, EXPIRED
        
        Args:
            order_id: Order ID to wait for
            timeout: Maximum wait time in seconds (uses default if None)
        
        Returns:
            List of all events for the order
        
        Raises:
            TimeoutError: If terminal status not reached within timeout
        """
        timeout = timeout or self.timeout
        start_time = asyncio.get_event_loop().time()
        
        terminal_statuses = {"FILLED", "CANCELLED", "REJECTED", "EXPIRED"}
        
        while True:
            # Check current events
            if order_id in self.events:
                for event in reversed(self.events[order_id]):
                    status = event.get("data", {}).get("status")
                    if status in terminal_statuses:
                        log.info(f"Order reached terminal status | order_id={order_id} | status={status}")
                        return self.events[order_id]
            
            # Check timeout
            elapsed = asyncio.get_event_loop().time() - start_time
            if elapsed > timeout:
                raise TimeoutError(
                    f"Terminal status not reached for order_id={order_id} within {timeout}s"
                )
            
            # Wait for next event
            if order_id not in self.queues:
                self.queues[order_id] = asyncio.Queue()
            
            try:
                remaining_timeout = timeout - elapsed
                await asyncio.wait_for(
                    self.queues[order_id].get(),
                    timeout=remaining_timeout,
                )
            except asyncio.TimeoutError:
                raise TimeoutError(
                    f"Terminal status not reached for order_id={order_id} within {timeout}s"
                )
    
    def get_events(self, order_id: str) -> List[dict]:
        """
        Get all events for an order.
        
        Args:
            order_id: Order ID
        
        Returns:
            List of events in chronological order
        """
        return self.events.get(order_id, [])
    
    async def cleanup(self) -> None:
        """Cleanup consumer group and resources."""
        if not self.redis_client:
            return
        
        for stream in self._subscribed_streams:
            try:
                # Delete consumer group
                await self.redis_client.xgroup_destroy(stream, self.consumer_group)
                log.info(f"Deleted consumer group {self.consumer_group} for stream {stream}")
            except Exception as e:
                log.warning(f"Failed to delete consumer group for stream {stream}: {e}")
```

**Verification:**
- File created with correct structure
- Unit tests for Redis stream reading
- Consumer group creation/deletion works

**Dependencies:** None

**Deliverable:** `redis_event_collector.py` with tests

---

### 1.4 Update event_collector.py (Day 6)

**File:** `smarttrade-tests/e2e/harness/event_collector.py`

**Changes:**
- Remove dependency on `bas_ws_client`
- Add support for new consolidated event schema
- Update status parsing logic

**Key Changes:**

```python
# Update terminal statuses for new schema
TERMINAL_STATUSES = {"FILLED", "CANCELLED", "REJECTED", "EXPIRED"}

# Update wait_for_status to handle status field
async def wait_for_status(
    self,
    order_id: str,
    status: str,
    timeout: float = 30.0,
) -> list[dict]:
    """Wait for order to reach a specific status."""
    start_time = asyncio.get_event_loop().time()
    
    while True:
        # Check current events
        events = self.get_events(order_id)
        for event in reversed(events):
            # NEW: Check status field in consolidated event schema
            event_status = (
                event.get("status")
                or event.get("data", {}).get("status")
                or event.get("order_status")
            )
            if event_status == status:
                return events
        
        # Check timeout
        elapsed = asyncio.get_event_loop().time() - start_time
        if elapsed > timeout:
            raise TimeoutError(
                f"Status {status} not reached for order_id={order_id} within {timeout}s"
            )
        
        # Wait for next event
        try:
            if order_id not in self.queues:
                await asyncio.sleep(0.1)
                continue
            
            remaining_timeout = timeout - elapsed
            await asyncio.wait_for(
                self.queues[order_id].get(),
                timeout=remaining_timeout,
            )
        except asyncio.TimeoutError:
            raise TimeoutError(
                f"Status {status} not reached for order_id={order_id} within {timeout}s"
            )
```

**Verification:**
- Status parsing works with consolidated schema
- All existing tests still pass (if any remain)

**Dependencies:** None

**Deliverable:** Updated `event_collector.py`

---

### 1.5 Update assertions.py (Day 7)

**File:** `smarttrade-tests/e2e/harness/assertions.py`

**Changes:**
- Add broker state verification assertions
- Add outbox pattern validation
- Update status-based validation

**Key Additions:**

```python
def assert_broker_state_matches_events(broker_state: dict, events: list[dict]) -> None:
    """
    Assert that broker state matches event-derived state.
    
    Validates that broker (source of truth) state matches the state
    reconstructed from events. This ensures event correctness.
    
    Args:
        broker_state: State from broker API
        events: Events collected from Redis Streams
    
    Raises:
        AssertionError: If states don't match
    """
    # Extract order state from events
    last_event = events[-1]
    event_status = last_event.get("data", {}).get("status")
    event_qty = last_event.get("data", {}).get("qty")
    event_price = last_event.get("data", {}).get("price")
    
    # Compare with broker state
    broker_status = broker_state.get("status")
    broker_qty = broker_state.get("qty")
    broker_price = broker_state.get("price")
    
    assert broker_status == event_status, (
        f"Status mismatch | broker={broker_status} | event={event_status}"
    )
    assert broker_qty == event_qty, (
        f"Qty mismatch | broker={broker_qty} | event={event_qty}"
    )
    assert broker_price == event_price, (
        f"Price mismatch | broker={broker_price} | event={event_price}"
    )


def assert_status_transition_correct(events: list[dict], expected_transitions: list[str]) -> None:
    """
    Assert that order status transitions follow expected sequence.
    
    Args:
        events: Order events in chronological order
        expected_transitions: Expected status sequence (e.g., ["PLACED", "FILLED"])
    
    Raises:
        AssertionError: If transitions don't match expected
    """
    actual_transitions = []
    for event in events:
        status = event.get("data", {}).get("status")
        if status and status not in actual_transitions:
            actual_transitions.append(status)
    
    assert actual_transitions == expected_transitions, (
        f"Status transition mismatch | expected={expected_transitions} | actual={actual_transitions}"
    )


def assert_outbox_event_published(outbox_record: dict, event: dict) -> None:
    """
    Assert that outbox record matches published event.
    
    Validates that outbox pattern correctly published event to Redis.
    
    Args:
        outbox_record: Outbox table record
        event: Event from Redis Stream
    
    Raises:
        AssertionError: If records don't match
    """
    assert outbox_record.get("event_type") == event.get("type"), (
        f"Event type mismatch | outbox={outbox_record.get('event_type')} | event={event.get('type')}"
    )
    assert outbox_record.get("event_id") == event.get("data", {}).get("event_id"), (
        f"Event ID mismatch | outbox={outbox_record.get('event_id')} | event={event.get('data', {}).get('event_id')}"
    )
    assert outbox_record.get("status") == "PUBLISHED", (
        f"Outbox record not published | status={outbox_record.get('status')}"
    )
```

**Verification:**
- New assertions work correctly
- Existing assertions still work

**Dependencies:** None

**Deliverable:** Updated `assertions.py`

---

### 1.6 Update config.py (Day 8)

**File:** `smarttrade-tests/e2e/config/config.py`

**Changes:**
- Add broker configuration
- Add Redis stream configuration
- Add outbox configuration

**Key Additions:**

```python
class TestConfig(BaseSettings):
    """E2E test configuration."""
    
    # Existing config...
    
    # NEW: Broker configuration
    broker_type: str = Field(default="pbs", description="Broker type: fyers or pbs")
    broker_api_url: str = Field(default="http://localhost:8002", description="Broker API URL")
    
    # NEW: Redis stream configuration
    redis_stream_consumer_group: str = Field(
        default="e2e-tests",
        description="Consumer group for Redis stream consumption"
    )
    
    # NEW: Outbox configuration
    outbox_table_name: str = Field(
        default="outbox",
        description="Outbox table name for testing"
    )
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
        extra = "ignore"
```

**Verification:**
- Configuration loads correctly
- Environment variables override defaults

**Dependencies:** None

**Deliverable:** Updated `config.py`

---

### 1.7 Update conftest.py (Day 9-10)

**File:** `smarttrade-tests/e2e/conftest.py`

**Changes:**
- Remove `bas_ws_client` fixture
- Add `broker_state_client` fixture
- Add `redis_event_collector` fixture
- Update imports

**Key Changes:**

```python
# Remove bas_ws_client import
from e2e.clients import (
    BASClient,
    # BASWebSocketClient,  # REMOVED
    MDSWebSocketClient,
    MockClient,
    PortfolioClient,
    JournalClient,
    BrokerStateClient,  # NEW
)
from e2e.harness import (
    EventCollector,
    AssertionEngine,
    ScenarioEngine,
    RedisEventCollector,  # NEW
)

# Remove bas_ws_client fixture
# @pytest_asyncio.fixture(scope="function")
# async def bas_ws_client(config, auth_token, test_account_id):
#     ...

# Add broker_state_client fixture
@pytest_asyncio.fixture(scope="function")
async def broker_state_client(config, auth_token) -> BrokerStateClient:
    """
    Provide broker state client for direct broker queries.
    
    Scope: function (created per test)
    """
    from e2e.clients.broker_state_client import create_broker_state_client
    
    client = create_broker_state_client(
        broker_type=config.broker_type,
        base_url=config.broker_api_url,
        token=auth_token,
    )
    async with client:
        yield client

# Add redis_event_collector fixture
@pytest_asyncio.fixture(scope="function")
async def redis_event_collector(config) -> RedisEventCollector:
    """
    Provide Redis event collector for stream-based event collection.
    
    Scope: function (created per test)
    """
    from e2e.harness.redis_event_collector import RedisEventCollector
    
    collector = RedisEventCollector(
        redis_url=config.redis_url,
        consumer_group=f"{config.redis_stream_consumer_group}-{uuid.uuid4().hex[:8]}",
    )
    await collector.connect()
    await collector.subscribe_to_streams([
        "events:order.updated",
        "events:trade.executed",
        "events:position.updated",
    ])
    
    yield collector
    
    await collector.cleanup()
    await collector.disconnect()

# Update existing fixtures to use new clients
@pytest_asyncio.fixture(scope="function")
async def event_collector(redis_event_collector: RedisEventCollector) -> EventCollector:
    """
    Provide event collector (now backed by Redis streams).
    
    Scope: function (created per test)
    """
    return EventCollector()
```

**Verification:**
- All fixtures load correctly
- No import errors
- Tests can use new fixtures

**Dependencies:** All previous tasks in Phase 1

**Deliverable:** Updated `conftest.py`

---

## Phase 1 Success Criteria

- [ ] `bas_ws_client.py` removed
- [ ] `broker_state_client.py` implemented and tested
- [ ] `redis_event_collector.py` implemented and tested
- [ ] `event_collector.py` updated for new schema
- [ ] `assertions.py` updated with new assertions
- [ ] `config.py` updated with new configuration
- [ ] `conftest.py` updated with new fixtures
- [ ] All fixtures load without errors
- [ ] Unit tests pass for new components

---

## Remaining Phases Overview

### Phase 2: Core Tests (Week 3-4)
- Update order lifecycle tests
- Update partial fill tests
- Update cancel order tests
- Update error path tests
- Update concurrent order tests

### Phase 3: Real Execution (Week 5)
- Update real execution tests
- Test PBS market data consumption

### Phase 4: Resilience (Week 6)
- Update resilience tests
- Add outbox crash recovery tests
- Add exactly-once semantics tests

### Phase 5: Integration (Week 7-8)
- Update integration tests
- Add instrument sync tests
- Add outbox pattern tests
- Add BAS-PBS sync tests
- Add Strategy advisory tests

### Phase 6: Architecture (Week 9)
- Update architecture boundary tests
- Update WebSocket tests
- Update event bus validation tests

### Phase 7: Full Regression (Week 10)
- Run full test suite
- Fix remaining issues
- Update documentation
- Validate CI/CD pipeline

---

## Risk Management

### High-Risk Items

1. **Broker API Availability**
   - **Risk:** Broker APIs may not be accessible or have rate limits
   - **Mitigation:** Use test environment with dedicated broker credentials
   - **Contingency:** Implement caching and query throttling

2. **Redis Stream Complexity**
   - **Risk:** Consumer groups may have edge cases or failures
   - **Mitigation:** Comprehensive error handling and logging
   - **Contingency:** Fall back to direct XREAD if consumer groups fail

3. **Event Schema Mismatch**
   - **Risk:** Actual event schema may differ from design
   - **Mitigation:** Verify schema against running services before implementation
   - **Contingency:** Make event parsing flexible and schema-agnostic where possible

### Medium-Risk Items

1. **Test Execution Time Increase**
   - **Risk:** Additional broker queries may slow tests
   - **Mitigation:** Optimize queries and use efficient Redis operations
   - **Contingency:** Increase test timeouts if needed

2. **Service Implementation Delays**
   - **Risk:** Services may not be ready when E2E tests need them
   - **Mitigation:** Coordinate closely with service teams
   - **Contingency:** Implement mocks for missing service capabilities

---

## Progress Tracking

### Week 1-2: Phase 1 - Infrastructure
- [ ] Day 1: Remove bas_ws_client.py
- [ ] Day 2-3: Add broker_state_client.py
- [ ] Day 4-5: Add redis_event_collector.py
- [ ] Day 6: Update event_collector.py
- [ ] Day 7: Update assertions.py
- [ ] Day 8: Update config.py
- [ ] Day 9-10: Update conftest.py

### Week 3-4: Phase 2 - Core Tests
- [ ] Update test_order_lifecycle_injection.py
- [ ] Update test_partial_fills_injection.py
- [ ] Update test_cancel_orders_injection.py
- [ ] Update test_error_paths_injection.py
- [ ] Update test_concurrent_orders_injection.py
- [ ] Run and validate injection tests

### Week 5: Phase 3 - Real Execution
- [ ] Update test_market_buy_real_execution.py
- [ ] Update test_partial_fills_real_execution.py
- [ ] Update test_execution_stress_scenarios.py
- [ ] Run and validate real execution tests

### Week 6: Phase 4 - Resilience
- [ ] Update test_resilience_timeouts.py
- [ ] Update test_resilience_event_handling.py
- [ ] Update test_resilience_partial_failures.py
- [ ] Add outbox crash recovery tests
- [ ] Run and validate resilience tests

### Week 7-8: Phase 5 - Integration
- [ ] Update test_journal_integration.py
- [ ] Update test_portfolio_integration.py
- [ ] Add test_instrument_sync.py
- [ ] Add test_outbox_pattern.py
- [ ] Add test_bas_pbs_sync.py
- [ ] Add test_strategy_advisory.py
- [ ] Run and validate integration tests

### Week 9: Phase 6 - Architecture
- [ ] Update test_architecture_boundaries.py
- [ ] Update test_websocket_client_routing.py
- [ ] Update test_websocket_separation_live.py
- [ ] Update test_event_bus_validation.py
- [ ] Run and validate architecture tests

### Week 10: Phase 7 - Full Regression
- [ ] Run full test suite
- [ ] Fix remaining issues
- [ ] Update documentation
- [ ] Validate CI/CD pipeline
- [ ] Final sign-off

---

## Communication Plan

### Weekly Status Updates

Every Monday, provide:
- Previous week progress
- Current week plan
- Blockers and risks
- Success metrics

### Stakeholder Reviews

- **Week 2:** Phase 1 completion review
- **Week 4:** Phase 2 completion review
- **Week 6:** Phase 3-4 completion review
- **Week 8:** Phase 5 completion review
- **Week 10:** Final review and sign-off

### Blocking Issues

Escalate immediately if:
- Service implementations are not ready
- Event schemas don't match design
- Broker APIs are inaccessible
- Redis Streams are not working
- Test execution time exceeds targets

---

## Success Metrics

### Quantitative Metrics

- **Test Coverage:** 62 tests (51 existing + 11 new)
- **Test Execution Time:** < 5s per test (average)
- **Test Reliability:** > 95% pass rate over 100 runs
- **Infrastructure Stability:** < 1% fixture failure rate

### Qualitative Metrics

- **Code Quality:** All code reviewed and approved
- **Documentation:** All components documented
- **Maintainability:** Clear separation of concerns
- **Debuggability:** Comprehensive logging and error messages

---

## Next Steps

1. **Review this execution plan** with all stakeholders
2. **Confirm prerequisites** are met
3. **Begin Phase 1.1** (Remove bas_ws_client.py)
4. **Daily standups** to track progress
5. **Weekly reviews** to ensure alignment

---

**Document Status:** Ready for Execution  
**Execution Start Date:** TBD (after prerequisites confirmed)  
**Owner:** E2E Test Team  
**Reviewers:** Architecture Team, Service Teams
