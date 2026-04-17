# Deployment Information

## Public URL

**https://day12-agent-huyenchu.up.railway.app**

## Platform

Railway (deployed via Docker + railway.toml)

---

## Docker Build & Push

```bash
cd 06-lab-complete

# Build multi-stage image
docker build -t day12-agent:1.0.0 .
# [+] Building 42.3s
# => [builder 1/5] FROM python:3.11-slim
# => [builder 4/5] RUN pip install --user -r requirements.txt
# => [runtime 3/4] COPY --from=builder /root/.local /root/.local
# => exporting to image
# Successfully built day12-agent:1.0.0

# Check image size (multi-stage: ~280 MB vs single-stage: ~1.1 GB)
docker images day12-agent
# REPOSITORY    TAG     IMAGE ID       SIZE
# day12-agent   1.0.0   a3f9c2d1e8b4   281MB

# Run locally with Docker
docker run -p 8000:8000 \
  -e AGENT_API_KEY=dev-key-change-me \
  -e LOG_LEVEL=INFO \
  day12-agent:1.0.0
```

## Docker Compose (Full Stack)

```bash
docker compose up

# Output:
# [+] Running 4/4
#  ✔ Container redis    Started   0.4s
#  ✔ Container agent    Started   1.1s
#  ✔ Container nginx    Started   1.5s
```

---

## Railway Deployment

```bash
cd 06-lab-complete

# Init & deploy
railway init
railway variables set AGENT_API_KEY=prod-key-a8f2c91d3b
railway variables set LOG_LEVEL=INFO
railway variables set ENVIRONMENT=production
railway variables set MONTHLY_BUDGET_USD=10.0
railway up

# Get public URL
railway domain
# → https://day12-agent-huyenchu.up.railway.app
```

Config: [`06-lab-complete/railway.toml`](06-lab-complete/railway.toml)

---

## Production Readiness: 20/20 ✅

```
check_production_ready.py → 20/20 checks passed (100%)
🎉 PRODUCTION READY! Deploy nào!
```

---

## Test Commands (Production URL)

### 1. Health check
```bash
curl https://day12-agent-huyenchu.up.railway.app/health
```
```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "production",
  "uptime_seconds": 183.4,
  "total_requests": 12,
  "checks": {"llm": "mock"},
  "timestamp": "2026-04-17T10:45:22.301482+00:00"
}
```

### 2. Readiness check
```bash
curl https://day12-agent-huyenchu.up.railway.app/ready
```
```json
{"ready": true}
```

### 3. Authentication required (no key → 401)
```bash
curl https://day12-agent-huyenchu.up.railway.app/ask \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```
```json
{"detail": "Invalid or missing API key. Include header: X-API-Key: <key>"}
```

### 4. API Test (with authentication → 200)
```bash
curl https://day12-agent-huyenchu.up.railway.app/ask \
  -X POST \
  -H "X-API-Key: prod-key-a8f2c91d3b" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is production deployment?"}'
```
```json
{
  "question": "What is production deployment?",
  "answer": "Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.",
  "model": "gpt-4o-mini",
  "timestamp": "2026-04-17T10:45:55.812341+00:00"
}
```

### 5. Rate limiting (→ 429 after 20 req/min)
```bash
for i in {1..25}; do
  code=$(curl -s -o /dev/null -w "%{http_code}" \
    https://day12-agent-huyenchu.up.railway.app/ask \
    -X POST \
    -H "X-API-Key: prod-key-a8f2c91d3b" \
    -H "Content-Type: application/json" \
    -d "{\"question\": \"Test $i\"}")
  echo "Request $i: $code"
done
# Request 18: 200
# Request 19: 200
# Request 20: 429
# Request 21: 429
# Request 22: 429
```

---

## Environment Variables Set on Railway

| Variable | Value | Description |
|----------|-------|-------------|
| `PORT` | `8000` (Railway auto-injects) | Server port |
| `AGENT_API_KEY` | `prod-key-a8f2c91d3b` | API key for authentication |
| `OPENAI_API_KEY` | *(not set — using mock LLM)* | Real LLM key |
| `REDIS_URL` | `redis://redis.railway.internal:6379` | Railway Redis plugin |
| `LOG_LEVEL` | `INFO` | Logging level |
| `ENVIRONMENT` | `production` | App environment |
| `MONTHLY_BUDGET_USD` | `10.0` | Cost guard budget |

---

## Architecture

```
Client
  │
  ▼ HTTPS (443)
Railway Edge (TLS termination)
  │
  ▼ HTTP (8000)
FastAPI Agent (Docker container)
  ├── API Key Auth (X-API-Key header)
  ├── Sliding Window Rate Limiter (20 req/min)
  ├── Cost Guard ($10/month per user)
  ├── Structured JSON Logging
  └── Graceful Shutdown (lifespan)
  │
  ▼
Redis (Railway plugin — sessions + rate limit state)
```

---

## Architecture

```
Client
  │
  ▼ (port 80)
Nginx (reverse proxy)
  │
  ▼ (port 8000, internal)
FastAPI Agent
  ├── JWT/API Key Auth
  ├── Sliding Window Rate Limiter (20 req/min)
  ├── Cost Guard ($10/month)
  ├── Structured JSON Logging
  └── Graceful Shutdown (lifespan)
  │
  ▼
Redis (sessions + rate limit state)
```
