# 📋 Broker Adapter Service — Unit Test Plan

## 1. Test Environment Setup
- **Fixtures:**
  - `app_client`: FastAPI test client with `DummyBus` (no Redis/DB).  
  - `auth_headers`: JWT with roles `admin,user`.  
  - `mock_broker_config`: Returns dummy `BrokerAdapterConfig`.  
  - `mock_broker_auth`: Returns dummy `CredentialSchema`.  
- **Purpose:** Isolate BAS from external services, make tests deterministic.  

---

## 2. Core Functional Areas

### 2.1 Config Loading (BrokerConfigClient)
- **Input:** `broker_id`  
- **Output:** `BrokerAdapterConfig` object or error  
- **Precondition:** Broker Config Service mocked  
- **Postcondition:** Config cached in memory  
- **Acceptance Criteria:**  
  - Valid broker → returns parsed config.  
  - Invalid broker → raises `BROKER_NOT_FOUND`.  
  - Schema mismatch → raises `CONFIG_INVALID`.  

---

### 2.2 Operation Resolution
- **Input:** `operation_type`, `instrument`, `product_type`  
- **Output:** Matching `OperationConfig`  
- **Acceptance Criteria:**  
  - Correct variant selected.  
  - No match → raises `VAL_002`.  

---

### 2.3 Order Validation
- **Input:** `PlaceOrderRequest` + `OperationConfig`  
- **Output:** Either valid or `SmartTradeError`  
- **Acceptance Criteria:**  
  - Missing required → `VAL_001`.  
  - Out of range → error.  
  - Conditional fail → error.  
  - Warning rules → logged only.  

---

### 2.4 Request Construction
- **Input:** Normalized order request + config mappings  
- **Output:** Broker-ready JSON payload  
- **Acceptance Criteria:**  
  - Field mappings applied.  
  - Defaults injected.  
  - Instrument translated.  
  - Unsupported fields removed.  

---

### 2.5 Request Signing
- **Input:** Payload + credentials + signing config  
- **Output:** Signed request with headers  
- **Acceptance Criteria:**  
  - Correct signature when key valid.  
  - Wrong key → signature mismatch.  

---

### 2.6 Rate Limiting
- **Input:** Config with `rate_limits` + multiple requests  
- **Output:** Requests throttled/delayed  
- **Acceptance Criteria:**  
  - Per-minute/per-second respected.  
  - Endpoint-specific honored.  

---

### 2.7 Response Handling
- **Input:** Broker raw response + config mappings  
- **Output:** Normalized `OrderResponseSchema`  
- **Acceptance Criteria:**  
  - Status mapped correctly.  
  - Broker ID → SmartTrade `order_id`.  
  - Missing/invalid fields → error.  

---

### 2.8 WebSocket Manager
- **Input:** WebSocket config + credentials  
- **Output:** SmartTrade events (`order.updated`, `position.updated`)  
- **Acceptance Criteria:**  
  - Updates → published events.  
  - Drop → reconnects.  
  - Heartbeat keeps alive.  

---

### 2.9 Event Publishing
- **Input:** Normalized response / WS update  
- **Output:** Event on `DummyBus`  
- **Acceptance Criteria:**  
  - `order.placed` → emitted.  
  - `order.filled` → emitted.  
  - Sensitive data filtered.  

---

## 3. Service-Orchestration Tests (BrokerAdapterService Class)
- **Methods:** `place_order`, `modify_order`, `cancel_order`, `get_positions`  
- **Inputs:** Broker ID, User ID, Normalized order requests  
- **Outputs:** Normalized responses + published events  
- **Acceptance Criteria:**  
  - `place_order` → returns SmartTrade response + `order.placed`.  
  - `modify_order` → returns response + `order.modified`.  
  - `cancel_order` → returns response + `order.canceled`.  
  - `get_positions` → returns list + `position.updated`.  

---

## 4. API Contract Tests (OpenAPI Spec)
- Endpoints:  
  - `POST /api/v1/orders/{broker_id}` → Place order  
  - `PUT /api/v1/orders/{broker_id}/{order_id}` → Modify order  
  - `DELETE /api/v1/orders/{broker_id}/{order_id}` → Cancel order  
  - `GET /api/v1/orders/{broker_id}/{order_id}` → Query order  
  - Internal equivalents with `{user_id}`  
- **Acceptance Criteria:**  
  - Valid requests → `200` with schema.  
  - Invalid → `422 Validation Error`.  
  - Missing auth → `401 Unauthorized`.  

---

## 5. Negative & Edge Cases
- Expired/invalid JWT → `401`.  
- Unsupported product type/instrument → validation error.  
- Broker timeout → `BROKER_TIMEOUT`.  
- High load → throttling kicks in.  

---

## 6. Test Coverage Goals
- 80%+ line coverage with `pytest --cov`.  
- Contract tests aligned with OpenAPI.  
- RBAC enforced (`admin-only` blocked for normal user).  
- Event filtering ensures no sensitive fields.  

---

## 7. Execution Strategy
- **Unit Tests:** Isolate helpers (config, auth, validation, signing).  
- **Service Tests:** Exercise `BrokerAdapterService` methods with mocks.  
- **API Tests:** Validate FastAPI routes against OpenAPI spec.  
- **Fixtures:**  
  - `mock_broker_config` → returns dummy config.  
  - `mock_broker_auth` → returns dummy credentials.  
  - `mock_external_services` → both together.  
  - `DummyBus` for capturing published events.  

---

✅ This test plan balances **unit tests, orchestration tests, and API contract tests**, while mocking external dependencies.


---

## 8. Additional Areas (Identified During Review)

### 8.1 Cache Behavior
- **Scope:** BrokerAuthClient, BrokerConfigClient
- **Tests:**
  - First call → populates cache.
  - Second call (same key) → served from cache, no network call.
  - `force_refresh=True` → bypasses cache and repopulates.
  - `invalidate_cache()` → clears entry; next call refetches.

### 8.2 Dispatcher Safety
- **Scope:** EventDispatcher.publish
- **Tests:**
  - Accepts only `BaseEvent` subclasses.
  - Rejects raw `dict` payloads (raises `SmartTradeError`).
  - Always applies `filter_data(..., safe=True)` before bus publish.

### 8.3 Modify & Cancel Order Path Alignment
- **Scope:** UserBrokerSession + Routes
- **Tests:**
  - Path `order_id` is injected into modify/cancel logic automatically.
  - Body does not need to provide `order_id`.
  - Mismatch between path/body `order_id` → raises validation error.

### 8.4 Lifespan Initialization
- **Scope:** FastAPI app startup/shutdown
- **Tests:**
  - `EventBus.init()` is invoked.
  - RBAC policies loaded from config file.
  - Tracing/metrics initialized when enabled.

### 8.5 Concurrency & Singleton Behavior
- **Scope:** AdapterManager, BrokerAdapterService
- **Tests:**
  - Parallel `get_service(broker_id)` calls return the same service instance.
  - Multiple `user_id` sessions coexist without interfering.
  - Concurrent calls use locks to avoid race conditions.

### 8.6 WebSocket Reconnection
- **Scope:** WebSocketManager
- **Tests:**
  - Simulated connection drop triggers reconnect.
  - Subscriptions resubscribed after reconnect.
  - Heartbeat mechanism keeps connection alive.

---
