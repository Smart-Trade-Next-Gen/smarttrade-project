# 📋 Broker Adapter Service — Core Class Implementation Tasks

## **Task 1: Define BrokerAdapterService Class**
**Purpose:** Create a central orchestrator class that wires all BAS helpers (config, validation, signing, rate limiting, WS, events).  

- **Input:**  
  - Broker ID, User ID, normalized order requests.  

- **Output:**  
  - Exposed async methods (`place_order`, `modify_order`, `cancel_order`, `get_positions`).  

- **DoD:**  
  - Class skeleton created.  
  - Methods delegate to internal helpers.  

- **Acceptance Criteria:**  
  - Class compiles and loads without runtime errors.  
  - Methods stubs ready for logic wiring.  

---

## **Task 2: Integrate BrokerConfigClient**
**Purpose:** Fetch broker config per broker.  

- **Input:** `broker_id`  
- **Output:** `BrokerAdapterConfig`  
- **DoD:**  
  - Always fetches via `BrokerConfigClient.get_config`.  
  - Handles caching & version checks automatically.  
- **Acceptance Criteria:**  
  - Calling `get_config` returns config object.  

---

## **Task 3: Integrate BrokerAuthClient**
**Purpose:** Fetch credentials (tokens, API keys, cookies) from Broker Auth Service.  

- **Input:** `broker_id`, `user_id`  
- **Output:** `BrokerCredential`  
- **DoD:**  
  - Credentials fetched and cached.  
  - Refresh/invalidate on `user.credentials.updated`.  
- **Acceptance Criteria:**  
  - Valid credentials injected into API requests.  

---

## **Task 4: Integrate UserSettingsClient**
**Purpose:** Fetch user-level settings (risk, defaults, notifications).  

- **Input:** `user_id`  
- **Output:** `UserSettings`  
- **DoD:**  
  - User settings loaded before validation/execution.  
  - Cache invalidated on `user.settings.updated`.  
- **Acceptance Criteria:**  
  - Default product type applied if missing.  

---

## **Task 5: Implement place_order**
**Purpose:** Core orchestration for new order.  

- **Input:** Normalized `OrderRequest` (dict), `broker_id`, `user_id`.  
- **Steps:**  
  1. Load broker config.  
  2. Resolve operation (Task 2).  
  3. Validate order (Rule Engine, Task 3).  
  4. Build request payload (Task 4).  
  5. Fetch credentials & sign request (Task 5).  
  6. Apply rate limits (Task 6).  
  7. Call broker REST API.  
  8. Normalize response (Task 7).  
  9. Publish `order.placed` event (Task 9).  

- **Output:** Normalized order response.  
- **DoD:**  
  - Successful order results in SmartTrade-standard response.  
  - Event published to bus.  
- **Acceptance Criteria:**  
  - Valid order → `order.placed`.  
  - Invalid order → validation error.  
  - Broker error → wrapped as `SmartTradeError`.  

---

## **Task 6: Implement modify_order**
- Same pipeline as `place_order`, but publishes `order.modified`.  
- **Acceptance Criteria:** Broker call modifies order, SmartTrade event published.  

---

## **Task 7: Implement cancel_order**
- Simplified pipeline: no payload build beyond `order_id`.  
- Publishes `order.canceled`.  
- **Acceptance Criteria:** Order removed in broker + event published.  

---

## **Task 8: Implement get_positions**
- **Input:** `user_id`, `broker_id`.  
- **Steps:**  
  1. Resolve operation (`get_positions`).  
  2. Fetch credentials, sign, rate-limit, call broker.  
  3. Normalize via `ResponseHandler`.  
  4. Publish `position.updated`.  
- **Acceptance Criteria:**  
  - Returns normalized list of positions.  
  - Event published.  

---

## **Task 9: Integrate WebSocketManager**
**Purpose:** Stream order + position updates.  

- **Input:** `broker_id`, `user_id`, config.ws, credentials.  
- **Output:** Events (`order.updated`, `position.updated`).  
- **DoD:**  
  - Auto-reconnect enabled.  
  - Heartbeat respected.  
- **Acceptance Criteria:**  
  - BAS stays connected.  
  - Events flow into bus.  

---

## **Task 10: Error Handling & Observability**
**Purpose:** Ensure consistent error codes, logging, tracing.  

- **DoD:**  
  - All errors wrapped in `SmartTradeError`.  
  - Trace ID injected into every log + event.  
- **Acceptance Criteria:**  
  - Broker timeouts produce `BROKER_TIMEOUT`.  
  - Invalid config produces `CONFIG_INVALID`.  
  - Events/logs include `trace_id`.  

---

📌 **Implementation Order:**  
1. Define `BrokerAdapterService` class skeleton.  
2. Add config + auth + user settings integration.  
3. Implement order lifecycle methods.  
4. Add WS manager.  
5. Finalize error handling & observability.  
