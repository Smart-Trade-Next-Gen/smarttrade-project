# 📋 Broker Adapter Service – Developer Task Plan

---

## 🟢 Phase 1 – Foundations
1. **Models**
   - Define Pydantic models for normalized requests/responses:
     - `OrderRequest`, `OrderResponse`
     - `Position`, `Fund`, `Holding`
     - Event models: `OrderPlacedEvent`, `OrderUpdatedEvent`, etc.
   - Add schema sensitivity (mark tokens/keys with `json_schema_extra={"sensitive": True}`).

2. **Schemas**
   - Create OpenAPI contracts (`schemas/orders.py`).
   - Cover all REST endpoints (`/orders/place`, `/orders/modify`, `/orders/cancel`, etc.).

3. **Repositories** *(only if local persistence is needed)*  
   - Likely not required → BAS remains stateless.  
   - Keep stub repo for audit if required later.

✅ Deliverable: Models + OpenAPI schemas, validated in CI.

---

## 🟡 Phase 2 – Core Clients
4. **Config Client**
   - Implement `config_client.py`:
     - Fetch broker YAML from Broker Config Service.
     - Validate against `broker_config_schema.json`.
     - Cache with TTL + background refresh.
     - Run self-tests (from `fyers.yaml`).

5. **Auth Client**
   - Implement `auth_client.py`:
     - Fetch tokens from Broker Auth Service.
     - Decrypt with `smarttrade_common.security.cryptography`.
     - Cache with TTL + background refresh.

6. **User Settings Client**
   - Implement `user_settings_client.py`:
     - Fetch user defaults (TIF, product type).
     - Cache with TTL + invalidate on `user.settings.updated` event.

✅ Deliverable: Three clients with caching + tests.

---

## 🟠 Phase 3 – Core Services
7. **Rule Validator**
   - Implement `rule_validator.py`.
   - Precompile rules at config load.
   - Expose Prometheus metrics (`bas_validation_failures_total`).

8. **Adapter Core**
   - Implement `adapter.py` orchestration:
     - Load config, user settings, token.
     - Apply defaults from user settings.
     - Run precompiled validations.
     - Call broker API via `BaseServiceClient` (retry/backoff).
     - Run plugin preprocess/sign/postprocess hooks.
     - Run response validation.
     - Publish normalized event.

✅ Deliverable: Core orchestration flow with hooks.

---

## 🔵 Phase 4 – Plugins
9. **Base Plugin**
   - Implement `plugins/base.py` with placeholders (`preprocess_request`, `sign_request`, etc.).

10. **Fyers Plugin**
    - Implement `plugins/fyers.py` overrides:
      - Sign requests (`Authorization: app_id:access_token`).
      - WS connect with token.
      - WS subscription (`SUBSCRIBE` payload).
      - WS parse (`PLACED`, `FILLED`, `REJECTED`, `CANCELLED`).

11. **Zerodha/Angel Stubs**
    - Create `plugins/zerodha.py` and `plugins/angel.py` with `TODO` markers.

✅ Deliverable: Working Fyers plugin + stubs for others.

---

## 🟣 Phase 5 – API Layer
12. **Routes**
    - Implement `api/orders.py`:
      - `POST /orders/place`
      - `POST /orders/modify`
      - `POST /orders/cancel`
      - `GET /orders/{id}`
      - `GET /positions`
      - `GET /funds`
      - `GET /holdings`
    - Routes must be **thin**: delegate to core adapter.
    - Apply RBAC with `@require_policy`.

✅ Deliverable: REST API exposed, OpenAPI auto-generated.

---

## 🟤 Phase 6 – Events & Observability
13. **Event Publisher**
    - Implement `events/publisher.py`:
      - Publish normalized `order.*` events.
      - Use `filter_data(..., safe=True)` before publish.
      - Async publish (don’t block response).

14. **Observability**
    - Structured JSON logging.
    - Prometheus `/metrics` endpoint:
      - `bas_requests_total{operation,status}`
      - `bas_validation_failures_total{rule}`
    - OpenTelemetry tracing:
      - Wrap external broker calls + WS.

15. **WebSocket Handler**
    - Implement WS runner:
      - Connect via plugin.
      - Manage subscriptions.
      - Parse events via plugin.
      - Publish `order.updated` events.
      - Use bounded asyncio queue to handle load.

✅ Deliverable: Event-driven observability & WS updates.

---

## 🟡 Phase 7 – Testing
16. **Unit Tests**
    - Models, validators, plugins.
    - Config + Auth + User Settings clients.

17. **Contract Tests**
    - BAS ↔ Broker Auth Service.
    - BAS ↔ Broker Config Service.
    - BAS ↔ User Settings Service.

18. **Integration Tests**
    - Place/modify/cancel order flows with `httpx_mock`.
    - WS event parsing.

19. **Self Tests**
    - Run broker config examples (`fyers.yaml`) automatically in CI.

✅ Deliverable: 80%+ coverage, tests green in CI.

---

## ⚫ Phase 8 – Deployment
20. **Docker**
    - Multi-stage Dockerfile (runtime + dev).  
    - `docker-compose.override.yml` for local development.

21. **CI/CD**
    - GitHub Actions pipeline:
      - Lint + type check (ruff, mypy).  
      - Run tests + coverage gate (≥80%).  
      - Validate broker configs with self-tests.  

✅ Deliverable: Production-ready deployment pipeline.

---

# 🚀 Implementation Order (per guideline)
1. Models  
2. Schemas  
3. Config/Auth/User Settings clients  
4. Rule Validator  
5. Adapter Core  
6. Plugins  
7. Routes  
8. Event Publisher + WS Handler  
9. Observability  
10. Tests  
11. Deployment  
