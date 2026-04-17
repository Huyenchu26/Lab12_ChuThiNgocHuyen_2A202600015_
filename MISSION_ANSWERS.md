# Day 12 Lab - Mission Answers

> **Name:** Chu Thị Ngọc Huyền 
> **Date:** 17/04/2026

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `develop/app.py`

1. **Hardcoded API key** — `OPENAI_API_KEY = "sk-..."` nằm trực tiếp trong code → bị lộ khi push lên GitHub
2. **Hardcoded port** — `uvicorn.run(app, port=8000)` → không linh hoạt khi deploy trên cloud (Railway/Render inject PORT tự động)
3. **Debug mode bật** — `debug=True` trong production → lộ stack trace, chạy chậm hơn, tắt tính năng security
4. **Không có health check** — Cloud platform không biết app còn sống để tự restart khi crash
5. **Không có graceful shutdown** — Request đang xử lý bị mất khi SIGTERM
6. **Logging không cấu trúc** — Dùng `print()` thay vì structured JSON → không thể parse bằng log aggregator

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Tại sao quan trọng? |
|---------|---------|------------|---------------------|
| Config | Hardcode trong code (`OPENAI_API_KEY = "sk-..."`) | Đọc từ env vars qua `Settings` dataclass | Secret không bị lộ khi push lên GitHub; dễ thay đổi giữa dev/staging/prod mà không cần sửa code |
| Health check | Không có | `/health` (liveness) + `/ready` (readiness) | Cloud platform cần biết agent còn sống để tự động restart khi crash; load balancer dùng `/ready` để biết có route traffic vào không |
| Logging | `print()` — in cả API key ra console | Structured JSON, không log secrets | JSON dễ parse bởi log aggregator (Datadog, Loki); không log secrets tránh lộ thông tin nhạy cảm |
| Shutdown | Đột ngột — request đang xử lý bị mất | Graceful qua `lifespan` + SIGTERM handler | Request đang xử lý được hoàn thành trước khi tắt; connections đến DB/Redis được đóng sạch |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions

1. **Base image là gì?** → `python:3.11` — full Python distribution (~1 GB). Base image là image Docker dùng làm nền tảng để build image của mình, khai báo bằng lệnh `FROM` trong Dockerfile.
2. **Working directory là gì?** → `/app` — tất cả lệnh tiếp theo chạy trong thư mục này bên trong container.
3. **Tại sao COPY requirements.txt trước?** → Docker build theo từng layer. Nếu `requirements.txt` không đổi, Docker dùng cache cho bước `pip install` mà không cài lại → build nhanh hơn khi chỉ sửa code.
4. **CMD vs ENTRYPOINT?** → `CMD` là lệnh mặc định, có thể override khi chạy `docker run`. `ENTRYPOINT` không override dễ dàng — dùng khi muốn container luôn chạy một command cố định.

### Exercise 2.3: Image size comparison

- **Develop** (`python:3.11`): ~1.1–1.2 GB
- **Production** (multi-stage `python:3.11-slim`): ~200–300 MB
- **Difference**: ~70–80% nhỏ hơn

**Multi-stage build giảm size như thế nào:**
- Stage 1 (`builder`): Cài `gcc`, `libpq-dev`, `pip install --user` vào `/root/.local`
- Stage 2 (`runtime`): COPY chỉ `/root/.local` từ stage 1 + source code, **không** copy compiler/build tools/pip cache

### Exercise 2.4: Docker Compose architecture

```
┌─────────────────────────────┐
│   Client (internet)          │
└────────────┬────────────────┘
             │ port 80 / 443
             ▼
┌─────────────────────────────┐
│   Nginx (reverse proxy)      │  ← rate limit, security headers
└────────────┬────────────────┘
             │ proxy_pass :8000 (internal)
             ▼
┌─────────────────────────────┐
│   Agent (FastAPI)            │  ← không expose port ra ngoài
│   healthcheck mỗi 30s        │
└──────┬──────────┬───────────┘
       │          │
       ▼          ▼
┌──────────┐ ┌──────────────┐
│  Redis   │ │   Qdrant      │
│ (cache)  │ │ (vector DB)   │
└──────────┘ └──────────────┘
       ↕              ↕
  redis_data     qdrant_data  (persistent volumes)
```

**Services:** `agent`, `redis`, `qdrant`, `nginx` — tất cả trong network `internal`, chỉ Nginx expose ra ngoài.

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment

**Phân tích `railway.toml`:**
- `builder = "NIXPACKS"` → Railway tự detect Python, không cần Dockerfile
- `startCommand = "uvicorn app:app --host 0.0.0.0 --port $PORT"` → dùng `$PORT` do Railway inject
- `healthcheckPath = "/health"` → Railway ping `/health` định kỳ; fail 3 lần → restart
- `restartPolicyType = "ON_FAILURE"` → tự restart khi crash

**Live URL:** https://lab12-agent-production-4620.up.railway.app

### Exercise 3.2: render.yaml vs railway.toml

| Tiêu chí | `railway.toml` | `render.yaml` |
|----------|---------------|---------------|
| **Format** | TOML | YAML |
| **Build** | Nixpacks (auto-detect) | `pip install -r requirements.txt` |
| **Start** | `uvicorn app:app --port $PORT` | `uvicorn app:app --port $PORT` |
| **Health check** | `healthcheckPath = "/health"` | `healthCheckPath: /health` |
| **Redis** | Thêm riêng qua Railway plugin | Khai báo trong cùng file (`type: redis`) |
| **Secrets** | `railway variables set KEY=val` | `sync: false` → set thủ công trên dashboard |
| **Auto-deploy** | Tự động khi kết nối GitHub | `autoDeploy: true` |
| **Region** | Chọn qua dashboard | `region: singapore` khai báo trong file |

### Exercise 3.3: Cloud Run CI/CD pipeline

**`cloudbuild.yaml` — 4 bước:**
```
Push to GitHub main
       │
       ▼
① Run tests (pytest)
       │ pass
       ▼
② docker build → tag với COMMIT_SHA
       │
       ▼
③ docker push → Google Container Registry
       │
       ▼
④ gcloud run deploy → Cloud Run live!
```

**`service.yaml`:**
- `minScale: 1` → giữ ít nhất 1 instance → tránh cold start
- `maxScale: 10` → tự scale khi traffic cao
- `containerConcurrency: 80` → mỗi instance xử lý 80 requests đồng thời
- `OPENAI_API_KEY` từ **GCP Secret Manager** → bảo mật nhất trong 3 platform

---

## Part 4: API Security

### Exercise 4.1: API Key authentication

**Test results (từ `develop/app.py`):**

```bash
# Không có key → 401
curl http://localhost:8000/ask -X POST -H "Content-Type: application/json" -d '{"question":"Hello"}'
# Response: {"detail": "Missing API key"}

# Có key → 200
curl http://localhost:8000/ask -X POST -H "X-API-Key: secret-key-123" -H "Content-Type: application/json" -d '{"question":"Hello"}'
# Response: {"answer": "...mock response..."}
```

**API key được check ở đâu:** Hàm `verify_api_key()` là FastAPI Dependency dùng trong `/ask` endpoint.

**Rotate key:** Thay env var `AGENT_API_KEY` trên cloud platform dashboard rồi restart — không cần sửa code.

### Exercise 4.2: JWT flow

```
Client → POST /auth/token (username/password)
       ← JWT token (payload + signature)

Client → POST /ask + Bearer token
       → Server: verify signature + check expiry (60 phút) + extract role
       ← Response
```

**JWT tốt hơn API key đơn giản:** Token có expiry tự hết hạn, chứa role (stateless auth), không cần query DB mỗi request.

### Exercise 4.3: Rate limiting

- **Algorithm:** Sliding Window Counter — queue timestamps 60 giây gần nhất
- **Limits:** `user`: 10 req/phút; `admin`: 100 req/phút
- **Bypass cho admin:** App check `role` từ JWT, dùng `rate_limiter_admin` instance khác

**Response khi hit limit:**
```json
{"error": "Rate limit exceeded", "limit": 10, "window_seconds": 60, "retry_after_seconds": 45}
```

### Exercise 4.4: Cost guard

**Phân tích `cost_guard.py`:**
- Track usage trong memory (`_records` dict) — phù hợp demo, mất khi restart
- `daily_budget_usd = $1/ngày/user`, `global_daily_budget_usd = $10/ngày`
- Cảnh báo khi dùng 80% budget

**Production implementation (Redis-based):**
```python
import redis
from datetime import datetime

r = redis.Redis()

def check_budget(user_id: str, estimated_cost: float) -> bool:
    month_key = datetime.now().strftime("%Y-%m")
    key = f"budget:{user_id}:{month_key}"
    current = float(r.get(key) or 0)
    if current + estimated_cost > 10:
        return False
    r.incrbyfloat(key, estimated_cost)
    r.expire(key, 32 * 24 * 3600)  # 32 days TTL
    return True
```

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks

**`/health` (Liveness probe):** "Agent còn sống không?" — Platform gọi định kỳ. Fail → restart container. Luôn trả 200 trừ khi process crash.

**`/ready` (Readiness probe):** "Sẵn sàng nhận traffic chưa?" — Load balancer dùng cái này. Trả 503 khi đang khởi động, 200 khi sẵn sàng.

**Khác nhau:** `/health` fail → **restart**. `/ready` fail → **ngừng route traffic** (container vẫn sống).

**Implementation:**
```python
@app.get("/health")
def health():
    uptime = round(time.time() - START_TIME, 1)
    return {"status": "ok", "uptime_seconds": uptime, "version": "1.0.0"}

@app.get("/ready")
def ready():
    if not _is_ready:
        raise HTTPException(status_code=503, detail="Agent not ready yet")
    return {"ready": True}
```

### Exercise 5.2: Graceful shutdown

**Implementation (FastAPI `lifespan`):**
```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    global _is_ready
    _is_ready = True          # Startup: sẵn sàng nhận request

    yield                     # App đang chạy

    # Shutdown (khi nhận SIGTERM):
    _is_ready = False         # 1. Ngừng nhận request mới
    while _in_flight_requests > 0:
        time.sleep(1)         # 2. Chờ request hiện tại xong
    logger.info("Shutdown complete")  # 3. Tắt sạch
```

### Exercise 5.3: Stateless design

**Anti-pattern:** Lưu conversation history trong memory → khi scale lên nhiều instances, request 2 đến instance khác → mất history.

**Solution:** Lưu trong Redis → bất kỳ instance nào cũng đọc được.
```python
def save_session(session_id: str, data: dict):
    _redis.setex(f"session:{session_id}", 3600, json.dumps(data))

def load_session(session_id: str) -> dict:
    data = _redis.get(f"session:{session_id}")
    return json.loads(data) if data else {}
```

### Exercise 5.4: Load balancing với Nginx

- `upstream agent_cluster` → Docker DNS `agent` resolve đến tất cả instances (round-robin)
- `proxy_next_upstream error timeout http_503` → nếu 1 instance fail, thử instance khác (tối đa 3 lần)
- `add_header X-Served-By $upstream_addr` → biết request đi vào instance nào

### Exercise 5.5: Stateless test results

```
Request 1: Instance 172.18.0.3:8000
Request 2: Instance 172.18.0.4:8000
Request 3: Instance 172.18.0.5:8000
Request 4: Instance 172.18.0.3:8000  (round-robin)

Instances used: 3 different instances
Total messages in history: 10 (5 user + 5 assistant) — vẫn đủ dù qua nhiều instances
PASS: Stateless design verified!
```

---

## Part 6: Final Project — Production Agent

### Local test results

Server chạy tại `http://localhost:8000` với mock LLM.

**Health check:**
```bash
curl http://localhost:8000/health
# {"status":"ok","version":"1.0.0","environment":"development","uptime_seconds":12.8,"total_requests":1}
```

**Auth (401 without key):**
```bash
curl http://localhost:8000/ask -X POST -H "Content-Type: application/json" -d '{"question":"Hello"}'
# {"detail": "Invalid or missing API key. Include header: X-API-Key: <key>"}
```

**Auth (200 with key):**
```bash
curl http://localhost:8000/ask -X POST -H "X-API-Key: dev-key-change-me" -H "Content-Type: application/json" -d '{"question":"Hello production!"}'
# {"question":"Hello production!","answer":"Đây là câu trả lời từ AI agent (mock)...","model":"gpt-4o-mini"}
```

**Rate limiting (429 after 20 req/min):**
```
Request 18: 200
Request 19: 200
Request 20: 429  ← Rate limit hit
Request 21: 429
Request 22: 429
```

### Production test results (Railway)

**Public URL:** https://lab12-agent-production-4620.up.railway.app

**Health check:**
```bash
curl https://lab12-agent-production-4620.up.railway.app/health
# {"status":"ok","version":"1.0.0","environment":"development","uptime_seconds":1724.7,"total_requests":8,"checks":{"llm":"mock"},"timestamp":"2026-04-17T10:46:03.550484+00:00"}
```

**API Test (với authentication):**
```bash
curl -X POST https://lab12-agent-production-4620.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me-in-production" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
# {"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","model":"gpt-4o-mini","timestamp":"2026-04-17T10:46:29.104612+00:00"}
```

### Production readiness check: 20/20

```
check_production_ready.py → Result: 20/20 checks passed (100%)
🎉 PRODUCTION READY!
```

