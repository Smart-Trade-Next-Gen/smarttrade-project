# Authentication Service — Design Document

**Version:** 1.0
**Port:** 8001
**Database:** `smarttrade_authentication_service`

---

## 1. Responsibility

Sole authority for user identity within SmartTrade. All other services trust JWTs issued here. No other service manages user credentials.

**In scope:**
- User registration + validation
- Password hashing (bcrypt)
- JWT access token issuance (HS256)
- Refresh token rotation
- Logout (token revocation)
- Password change
- RBAC role assignment

**Out of scope:**
- Broker authentication (handled by BAS)
- Authorization enforcement on other service routes (each service validates JWTs independently)

---

## 2. Data Models

### User

```python
class User(Base):
    id: UUID                    # PK, auto-generated
    email: str                  # unique, indexed
    hashed_password: str        # bcrypt, cost factor 12
    role: str                   # "admin" | "trader" | "viewer"
    is_active: bool             # False = suspended
    created_at: datetime
    updated_at: datetime
```

### RefreshToken

```python
class RefreshToken(Base):
    id: UUID
    user_id: UUID               # FK → User
    token_hash: str             # SHA-256 of raw token
    expires_at: datetime
    revoked: bool               # True after use or explicit logout
    created_at: datetime
    used_at: datetime | None
```

---

## 3. API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/register` | None | Create new user |
| POST | `/login` | None | Authenticate + issue tokens |
| POST | `/logout` | Bearer | Revoke refresh token |
| POST | `/refresh` | Refresh token | Issue new access token, rotate refresh |
| POST | `/change-password` | Bearer | Update password |
| GET | `/me` | Bearer | Current user profile |

### POST /login — Response

```json
{
  "access_token": "eyJ...",
  "refresh_token": "uuid-v4",
  "token_type": "bearer",
  "expires_in": 900
}
```

---

## 4. Token Strategy

### Access Token (JWT)

```json
{
  "sub": "user-uuid",
  "role": "trader",
  "email": "user@example.com",
  "iat": 1711008000,
  "exp": 1711008900
}
```

- Signed with `JWT_SECRET_KEY` (HS256)
- 15-minute expiry (not configurable in prod)
- Stateless — validated on each request without DB lookup
- Carries only `user_id`, `role`, `email` — minimal claims

### Refresh Token

- 128-bit random UUID (version 4)
- Stored as SHA-256 hash in DB (never plaintext)
- 7-day expiry
- Single-use: revoked immediately on use, new token issued
- Family tracking: compromised token invalidates entire family (future enhancement)

---

## 5. Service Layer

### UserService

```python
async def register(email, password) -> User
    # 1. Validate email format + uniqueness
    # 2. Validate password complexity
    # 3. bcrypt.hash(password, rounds=12)
    # 4. User.save()
    # 5. EventBus.publish(user.registered)
    # 6. AuditLog.record(REGISTER, user_id)

async def login(email, password) -> TokenPair
    # 1. UserRepository.get_by_email(email)
    # 2. bcrypt.verify(password, user.hashed_password)
    # 3. If invalid: AuditLog.record(LOGIN_FAILED)
    # 4. JWTUtils.sign(user)
    # 5. RefreshTokenRepository.create(user_id)
    # 6. AuditLog.record(LOGIN_SUCCESS)
    # 7. Return TokenPair

async def refresh(refresh_token) -> TokenPair
    # 1. SHA-256 hash the incoming token
    # 2. RefreshTokenRepository.get_by_hash(hash)
    # 3. Assert: not revoked, not expired
    # 4. RefreshTokenRepository.revoke(token_id)      [ACID]
    # 5. RefreshTokenRepository.create(user_id)       [ACID]
    # 6. JWTUtils.sign(user)
    # 7. Return new TokenPair

async def logout(user_id, refresh_token) -> None
    # 1. Find + revoke refresh token
    # 2. AuditLog.record(LOGOUT)

async def change_password(user_id, old_password, new_password) -> None
    # 1. Verify old_password
    # 2. bcrypt.hash(new_password)
    # 3. User.update(hashed_password=...)
    # 4. Revoke all existing refresh tokens for user
    # 5. AuditLog.record(PASSWORD_CHANGED)
```

---

## 6. Security Hardening

### Password Policy (validation.py)

- Minimum 8 characters
- At least one uppercase, one lowercase, one digit
- Common password blocklist check
- Email format validation (RFC 5322 subset)

### Brute Force Protection

- Rate limiting via `smarttrade_common.resilience.distributed_rate_limiter`
- Per-IP: 20 requests/minute on `/login`
- Per-email: 5 failed attempts → 15-minute lockout (tracked in Redis)

### Timing Attack Prevention

- Always run bcrypt verify even for non-existent users (dummy hash comparison)
- Constant-time string comparison for token hashes

---

## 7. RBAC

Roles are simple strings in the JWT. Policy files define route-level permissions:

```yaml
# rbac_policies.yaml (example)
policies:
  - role: admin
    routes: ["*"]
  - role: trader
    routes: ["/api/v1/orders/*", "/api/v1/portfolio/*", "/api/v1/positions/*"]
  - role: viewer
    routes: ["/api/v1/portfolio/*", "/api/v1/positions/*"]
```

RBAC validation happens in each service via `smarttrade_common.security.rbac`, not centrally. The Auth service is only responsible for embedding the `role` claim in the JWT.

---

## 8. Error Codes

| Code | Message | HTTP |
|------|---------|------|
| `AUTH_001` | Invalid credentials | 401 |
| `AUTH_002` | Email already registered | 409 |
| `AUTH_003` | Refresh token expired or revoked | 401 |
| `AUTH_004` | Account suspended | 403 |
| `AUTH_005` | Password policy violation | 422 |
| `AUTH_006` | Invalid token | 401 |

---

## 9. Test Coverage

| Test File | Scope |
|-----------|-------|
| `test_services_register.py` | Registration validation, duplicate detection |
| `test_services_login.py` | Credential verification, token issuance |
| `test_services_logout.py` | Token revocation |
| `test_services_refresh.py` | Token rotation, expiry handling |
| `test_services_refresh_rotation.py` | Security: reuse of revoked token |
| `test_routes_auth.py` | HTTP layer: request/response format |
| `test_tokens_access.py` | JWT sign/verify/expiry |
| `test_tokens_refresh.py` | Refresh token DB lifecycle |
| `test_change_password.py` | Password update + token invalidation |
| `test_security_edges.py` | Timing attacks, brute force |
| `test_refresh_rotation.py` | Rotation under concurrent requests |
