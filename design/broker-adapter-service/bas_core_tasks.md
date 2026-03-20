# 📋 Broker Adapter Service — Core Implementation Tasks

## **Task 1: Update Broker Config Client**
**Purpose:** Ensure BAS loads broker config aligned with the new schema.  

- **Input:**  
  - `broker_id` (string).  
  - Broker Config Service API response (JSON matching `broker_config_schema.json`).  

- **Output:**  
  - `BrokerAdapterConfig` Pydantic object.  

- **Steps:**  
  1. Replace usage of old `BrokerConfig` with `BrokerAdapterConfig`.  
  2. Add schema version check (`config.schema_version == "1.0"`).  
  3. Cache configs per broker.  

- **Definition of Done (DoD):**  
  - Config loads successfully into `BrokerAdapterConfig`.  
  - Errors handled: 404 → `BROKER_NOT_FOUND`, schema mismatch → `CONFIG_INVALID`.  
  - Config version mismatch returns error.  

- **Acceptance Criteria:**  
  - Given valid config → returns parsed Pydantic object.  
  - Given invalid config → raises `SmartTradeError(CONFIG_INVALID)`.  
  - Given unknown broker → raises `SmartTradeError(BROKER_NOT_FOUND)`.

---

## **Task 2: Operation Resolution**
**Purpose:** Map SmartTrade canonical operation to broker-specific config entry.  

- **Input:**  
  - `operation_type` (e.g., `"place_order"`).  
  - `instrument` (e.g., `"OPTIONS"`).  
  - `product_type` (e.g., `"INTRADAY"`).  

- **Output:**  
  - Matching `OperationConfig` object from config.  

- **Steps:**  
  1. Collect all operations with matching `operation_type`.  
  2. If multiple → filter by `applies_to.instrument` and `applies_to.product_types`.  
  3. Return best match.  

- **DoD:**  
  - Always returns exactly one operation config or raises error.  

- **Acceptance Criteria:**  
  - Given a config with multiple variants → correct one chosen by `applies_to`.  
  - No match → raises `SmartTradeError(VAL_002)`.

---

## **Task 3: Order Validation**
**Purpose:** Prevent invalid orders before calling broker.  

- **Input:**  
  - Normalized order request (Pydantic model).  
  - `OperationConfig` (with validations).  

- **Output:**  
  - Either order passes, or `SmartTradeError` raised with details.  

- **Steps:**  
  1. Run `validations.rules` (all_required, forbidden, range, conditional, relation, severity).  
  2. Run `instrument_validations` for instrument type.  
  3. Enforce severity rules (blocking vs warning).  

- **DoD:**  
  - All invalid inputs are caught before API call.  
  - Warnings logged but don’t block execution.  

- **Acceptance Criteria:**  
  - Missing required field → raises `SmartTradeError(VAL_001)`.  
  - Field out of range → raises error.  
  - Conditional validation fails → raises error.  
  - Warning rule → logs warning only.

---

## **Task 4: Request Construction**
**Purpose:** Transform normalized order → broker payload.  

- **Input:**  
  - Normalized order request.  
  - `OperationConfig.request` mappings, default_values, instrument_mappings.  

- **Output:**  
  - Broker-ready request payload (dict).  

- **Steps:**  
  1. Apply field mappings (`symbol → sym`).  
  2. Inject default values if missing.  
  3. Apply instrument mappings (e.g., `NIFTY24SEP → NSE:25SEP2024`).  
  4. Strip unsupported fields.  

- **DoD:**  
  - Broker request matches schema.  

- **Acceptance Criteria:**  
  - Symbol correctly mapped.  
  - Missing `order_type` filled with default.  
  - Options order uses broker’s expected strike/expiry format.

---

## **Task 5: Request Signing**
**Purpose:** Add signing headers for brokers requiring HMAC.  

- **Input:**  
  - Broker request payload + headers.  
  - `extra_config.signing`.  

- **Output:**  
  - Request with signed headers (`signature_header`, `timestamp_header`).  

- **Steps:**  
  1. Generate timestamp in required format.  
  2. Compute HMAC using secret key (from Broker Auth Service).  
  3. Attach signature + timestamp headers.  

- **DoD:**  
  - Signed requests accepted by broker.  

- **Acceptance Criteria:**  
  - Given correct key → signature matches broker expectation.  
  - Incorrect → broker rejects request (test case).  

---

## **Task 6: Rate Limiting**
**Purpose:** Prevent hitting broker limits.  

- **Input:**  
  - Config: `extra_config.rate_limits`.  
  - Incoming requests.  

- **Output:**  
  - Requests throttled/respected per config.  

- **Steps:**  
  1. Implement in-memory counters or token-bucket.  
  2. Enforce per-minute, per-second, per-endpoint.  

- **DoD:**  
  - No broker-side rate limit errors under load test.  

- **Acceptance Criteria:**  
  - 120 requests with per_minute=100 → last 20 delayed/throttled.  
  - Endpoint-specific limits honored.  

---

## **Task 7: Response Handling**
**Purpose:** Normalize broker responses → SmartTrade standard.  

- **Input:**  
  - Broker raw response JSON.  
  - `OperationConfig.response.mappings`.  

- **Output:**  
  - Normalized SmartTrade response object.  

- **Steps:**  
  1. Apply response mappings (`id → order_id`).  
  2. Validate fields if `response.validations` present.  
  3. Return normalized object.  

- **DoD:**  
  - All BAS outputs consistent with SmartTrade schemas.  

- **Acceptance Criteria:**  
  - Broker order_id returned as normalized `order_id`.  
  - Broker status mapped correctly (e.g., `COMPLETE → filled`).  

---

## **Task 8: WebSocket Manager**
**Purpose:** Handle streaming updates from brokers.  

- **Input:**  
  - `extra_config.websockets`.  

- **Output:**  
  - Published SmartTrade events: `order.*`, `position.*`, `market.*`.  

- **Steps:**  
  1. Connect to correct WS URL (`orders`, `positions`, etc.).  
  2. Send auth payload if defined.  
  3. Send subscribe/unsubscribe messages with placeholders.  
  4. Maintain heartbeat.  
  5. Reconnect on drop.  

- **DoD:**  
  - Real-time updates flow into event bus.  

- **Acceptance Criteria:**  
  - Receiving order update → `order.updated` event published.  
  - Connection drop → BAS reconnects automatically.  
  - Heartbeat keeps connection alive.  

---

## **Task 9: Event Publishing**
**Purpose:** Notify other services.  

- **Input:**  
  - Normalized responses, updates, errors.  

- **Output:**  
  - Events on Redis/Kafka (`order.*`, `position.*`, `risk.*`).  

- **Steps:**  
  1. Filter sensitive data via `filter_data(..., safe=True)`.  
  2. Publish with trace_id for observability.  

- **DoD:**  
  - Journal, Risk Engine, Notifications consume events successfully.  

- **Acceptance Criteria:**  
  - Every placed order emits `order.placed`.  
  - Filled order emits `order.filled`.  
  - No secrets (tokens, keys) leak in events.  

---

📌 **Implementation Order:**  
1. Config client  
2. Operation resolution  
3. Validation  
4. Request building  
5. Signing  
6. Rate limiting  
7. Response handling  
8. WebSockets  
9. Event publishing  
