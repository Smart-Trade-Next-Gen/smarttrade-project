# Design Document Template

Use this template when creating a new cross-service design document.

**File naming:**
```
smarttrade-project/design/cross-service/YYYY-MM-DD-<feature-name>-v1.md
```

**Replace tokens:**
- `<DATE>` → Current date (YYYY-MM-DD)
- `<FEATURE>` → Feature name (lowercase, hyphens)
- `<Service>` → Service name (exact from CLAUDE.md)

---

## Template

```markdown
# <Feature> Design
**Date:** <DATE> | **Version:** v1 | **Status:** Design

## Executive Summary

One paragraph describing:
- What is this feature?
- Why are we building it? (business/technical driver)
- Which services are affected?
- Success criteria

## Architecture Diagram

ASCII diagram or description showing:
- Service boundaries
- Data flows (arrows showing direction)
- Event flows (labeled with event name)
- API calls (labeled with endpoint)

Example:
\`\`\`
Frontend      BAS          MDS
   |           |            |
   |--POST /orders---------->|
   |           |            |
   |<--order.placed---------|
   |           |            |
   |           |--publish market.tick--|
   |<--price update-----|---|
\`\`\`

## Service-Specific Sections

### Broker Adapter Service

**What BAS implements:**
- New models: List with brief description
- New schemas: Request/response types
- New routes: POST/GET/PUT/DELETE endpoints
- Database changes: New tables, migrations
- Events: Publishes what? Consumes what?
- Dependencies: On which other services?
- Error handling: New error codes

Example:
\`\`\`
**Models:** Order, Trade, Settlement (Decimal amounts, user-scoped)
**Schemas:** OrderRequest, OrderResponse, TradeResponse
**Routes:** POST /api/v1/orders, GET /api/v1/orders/{order_id}
**DB:** orders table (order_id, user_id, amount, status)
**Events:** Publishes order.placed, order.filled
**Dependencies:** Consumes market.tick from MDS
**Error Codes:** ORD_001 (invalid order), ORD_002 (insufficient funds)
\`\`\`

### Market Data Service

(Repeat for MDS)

### Authentication Service

(Repeat for Auth)

### Frontend

(Repeat for Frontend)

## Detailed Specifications

### Event Schemas

For each event published, define full schema:

\`\`\`yaml
event_name: order.placed
schema:
  order_id: UUID
  user_id: UUID
  symbol: string
  quantity: integer
  price: string (Decimal)
  timestamp: ISO8601
\`\`\`

### API Contracts

For each new endpoint, define OpenAPI spec:

\`\`\`yaml
POST /api/v1/orders:
  request:
    symbol: string (required)
    quantity: integer (required)
    price: string (Decimal, required)
  response:
    order_id: UUID
    status: string (enum: pending, filled, rejected)
  errors:
    - 400: Invalid order (ORD_001)
    - 401: Unauthorized
    - 422: Insufficient funds (ORD_002)
\`\`\`

### Data Models

Define database tables/migrations:

\`\`\`sql
CREATE TABLE orders (
  order_id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  symbol VARCHAR(10) NOT NULL,
  quantity INTEGER NOT NULL,
  price NUMERIC(20, 8) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
\`\`\`

### Validation Rules

- User must have trading authorization (RBAC role check)
- Order quantity must be > 0
- Price must be > 0
- Symbol must exist in instrument list
- User account must not be frozen

## Dependencies & Sequencing

What must happen first? What blocks what?

\`\`\`
1. smarttrade-common: Add Order, Trade models → BLOCKS all services
2. BAS: Implement order creation endpoint → BLOCKS MDS price validation
3. MDS: Implement price updates → BLOCKS BAS settlement
4. Frontend: Implement order UI → can start after BAS API is ready
\`\`\`

## Testing Strategy

### Unit Tests
- Order validation (quantity > 0, price > 0, symbol exists)
- Risk checks (margin, loss limits)
- Decimal arithmetic (no float rounding errors)
- Error code generation

### Integration Tests
- Order → Event publishing → Event consumption
- Database persistence (real DB, real transactions)
- RBAC authorization checks

### E2E Tests
- Full order placement flow (Frontend → BAS → MDS → BAS settle → Frontend)
- Cross-service event propagation
- Error scenarios (rejected orders, network failures)

## Success Criteria

- [ ] All services implement their sections per spec
- [ ] All events/APIs match spec exactly
- [ ] Unit tests: 90%+ coverage on business logic
- [ ] Integration tests: Order flow end-to-end
- [ ] E2E tests: Full workflow passing
- [ ] All tests passing before merge
```

---

## Tips for Writing Design Docs

1. **Be specific:** Don't write "add validation" — say "Validate order quantity > 0, price > 0, symbol in instrument list"
2. **Include examples:** Show schema, API, event examples
3. **Define data:** Explicitly state Decimal vs float, table names, column types
4. **Name error codes:** ORD_001, ORD_002 (prefix_NNN) for easy lookup
5. **Sequence matters:** List what blocks what so implementers know order
6. **Link to tests:** Design should make testing obvious
