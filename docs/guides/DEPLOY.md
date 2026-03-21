# How to: Deploy SmartTrade to Production

---

## Architecture Overview

```
Internet
    ↓
nginx (SSL termination, reverse proxy)
    ├── / → Frontend (static files)
    ├── /api/v1/auth/* → Auth Service (:8001)
    ├── /api/v1/* → BAS (:8005)
    ├── /mds/* → MDS (:8004)
    └── /ws/* → MDS WebSocket (:8004)

Internal Network
    ├── PostgreSQL (:5432)
    ├── Redis (:6379)
    └── Mock Service (:8002)
```

---

## 1. Environment Preparation

### Required Secrets

Generate all secret values before deployment:

```bash
# JWT Secret Key
python -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"

# Token Encryption Key (AES-256)
python -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"

# PostgreSQL password
python -c "import secrets; print(secrets.token_urlsafe(32))"

# Redis password
python -c "import secrets; print(secrets.token_urlsafe(16))"
```

Store all secrets in a secrets manager (HashiCorp Vault, AWS Secrets Manager, or encrypted `.env` files). **Never commit secrets to git.**

### SSL Certificate

```bash
# Let's Encrypt (production)
certbot certonly --nginx -d smarttrade.yourdomain.com

# Self-signed (staging/testing)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

---

## 2. Docker Compose (Recommended)

### docker-compose.yml (production)

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: smarttrade
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-dbs.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "smarttrade"]
      interval: 10s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
    restart: unless-stopped

  auth:
    build: ./authentication-service
    environment:
      ENV: prod
      DATABASE_URL: postgresql+asyncpg://smarttrade:${POSTGRES_PASSWORD}@postgres:5432/smarttrade_authentication_service
      JWT_SECRET_KEY: ${JWT_SECRET_KEY}
      TOKEN_ENCRYPTION_KEY: ${TOKEN_ENCRYPTION_KEY}
      EVENT_BUS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      SENTRY_ENABLED: "true"
      SENTRY_DSN: ${SENTRY_DSN}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  mds:
    build: ./market-data-service
    environment:
      ENV: prod
      DATABASE_URL: postgresql+asyncpg://smarttrade:${POSTGRES_PASSWORD}@postgres:5432/smarttrade_market_data_service
      JWT_SECRET_KEY: ${JWT_SECRET_KEY}
      EVENT_BUS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      FYERS_APP_ID: ${FYERS_APP_ID}
    depends_on: [postgres, redis, auth]
    restart: unless-stopped

  bas:
    build: ./broker-adapter-service
    environment:
      ENV: prod
      DATABASE_URL: postgresql+asyncpg://smarttrade:${POSTGRES_PASSWORD}@postgres:5432/smarttrade_broker_adapter_service
      JWT_SECRET_KEY: ${JWT_SECRET_KEY}
      TOKEN_ENCRYPTION_KEY: ${TOKEN_ENCRYPTION_KEY}
      EVENT_BUS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      FYERS_APP_ID: ${FYERS_APP_ID}
      FYERS_APP_SECRET: ${FYERS_APP_SECRET}
      MDS_URL: http://mds:8004
    depends_on: [postgres, redis, auth, mds]
    restart: unless-stopped

  frontend:
    build:
      context: ./smarttrade-frontend
      args:
        VITE_AUTH_BASE: https://smarttrade.yourdomain.com/api/v1/auth
        VITE_BAS_BASE: https://smarttrade.yourdomain.com/api/v1
        VITE_MDS_BASE: https://smarttrade.yourdomain.com/mds
        VITE_WAS_BASE: wss://smarttrade.yourdomain.com/ws
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on: [auth, mds, bas, frontend]
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### init-dbs.sql

```sql
CREATE DATABASE smarttrade_authentication_service;
CREATE DATABASE smarttrade_market_data_service;
CREATE DATABASE smarttrade_broker_adapter_service;
CREATE DATABASE smarttrade_mock_service;
```

---

## 3. nginx Configuration

```nginx
# nginx.conf
events { worker_connections 1024; }

http {
    upstream auth    { server auth:8001; }
    upstream bas     { server bas:8005; }
    upstream mds     { server mds:8004; }
    upstream frontend { server frontend:80; }

    server {
        listen 80;
        server_name smarttrade.yourdomain.com;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name smarttrade.yourdomain.com;

        ssl_certificate /etc/letsencrypt/live/smarttrade.yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/smarttrade.yourdomain.com/privkey.pem;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

        # Auth service
        location /api/v1/auth {
            proxy_pass http://auth;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # BAS (trading API)
        location /api/v1/ {
            proxy_pass http://bas;
            proxy_set_header Host $host;
        }

        # MDS REST
        location /mds/ {
            proxy_pass http://mds/;
            proxy_set_header Host $host;
        }

        # MDS WebSocket
        location /ws/ {
            proxy_pass http://mds;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 3600;
        }

        # Frontend SPA
        location / {
            proxy_pass http://frontend;
            try_files $uri $uri/ /index.html;
        }
    }
}
```

---

## 4. Database Migrations

Run migrations before starting services (or as part of service startup):

```bash
# Run as part of deployment
docker-compose exec auth uv run alembic upgrade head
docker-compose exec mds uv run alembic upgrade head
docker-compose exec bas uv run alembic upgrade head
```

Or add to each service's `Dockerfile`:

```dockerfile
CMD ["sh", "-c", "uv run alembic upgrade head && uv run uvicorn service.main:app --host 0.0.0.0 --port 8001"]
```

---

## 5. Deployment Steps

```bash
# 1. Build all images
docker-compose build

# 2. Start infrastructure first
docker-compose up postgres redis -d

# 3. Wait for health checks
docker-compose ps  # Both should show "healthy"

# 4. Start services
docker-compose up auth mds bas mock frontend nginx -d

# 5. Run migrations
docker-compose exec auth uv run alembic upgrade head
docker-compose exec mds uv run alembic upgrade head
docker-compose exec bas uv run alembic upgrade head

# 6. Seed admin user
docker-compose exec auth uv run python -c "
from authentication_service.seeder import seed_admin
import asyncio
asyncio.run(seed_admin(email='admin@yourcompany.com', password='<strong-password>'))
"

# 7. Verify health
curl https://smarttrade.yourdomain.com/api/v1/auth/ready
curl https://smarttrade.yourdomain.com/mds/ready
curl https://smarttrade.yourdomain.com/api/v1/ready
```

---

## 6. Rolling Updates (Zero Downtime)

For updates to a single service:

```bash
# 1. Build new image
docker-compose build bas

# 2. Scale up (2 instances briefly)
docker-compose up -d --scale bas=2 bas

# 3. Wait for new instance healthy
sleep 30

# 4. Remove old instance
docker-compose up -d --scale bas=1 bas
```

For database migrations with zero downtime:
- Migrations must be backward-compatible (additive only)
- Deploy migration first, then code
- Never remove columns or rename in the same deploy as the code change

---

## 7. Monitoring Setup

### Prometheus

Add to `docker-compose.yml`:

```yaml
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
```

`prometheus.yml`:
```yaml
scrape_configs:
  - job_name: smarttrade
    static_configs:
      - targets: ['auth:8001', 'mds:8004', 'bas:8005']
    metrics_path: /metrics
    scrape_interval: 15s
```

### Grafana

```yaml
  grafana:
    image: grafana/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
```

Import the SmartTrade dashboard template from `smarttrade-project/docs/grafana/`.

---

## 8. Backup Strategy

### PostgreSQL

```bash
# Daily backup (cron job)
0 2 * * * docker-compose exec postgres pg_dumpall -U smarttrade > /backups/smarttrade-$(date +%Y%m%d).sql

# Restore
docker-compose exec -T postgres psql -U smarttrade < /backups/smarttrade-20260321.sql
```

### Redis

Redis persistence is enabled via `--appendonly yes`. For snapshots:
```bash
docker-compose exec redis redis-cli BGSAVE
docker cp smarttrade_redis_1:/data/dump.rdb /backups/redis-$(date +%Y%m%d).rdb
```

---

## 9. Pre-Production Checklist

- [ ] All secrets generated and stored securely (not in git)
- [ ] SSL certificate valid and auto-renewal configured
- [ ] PostgreSQL passwords changed from defaults
- [ ] Redis password set
- [ ] Sentry DSN configured for error tracking
- [ ] All services pass `/ready` health check
- [ ] Prometheus scraping all services
- [ ] Grafana dashboard shows metrics
- [ ] Backup script scheduled and tested
- [ ] nginx rate limiting configured (prevent DDoS)
- [ ] CORS origins restricted to production domain
- [ ] `ENV=prod` set (disables debug endpoints)
