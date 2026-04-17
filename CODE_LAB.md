#  Code Lab: Deploy Your AI Agent to Production

> **AICB-P1 · VinUniversity 2026**  
> Thời gian: 3-4 giờ | Độ khó: Intermediate

##  Mục Tiêu

Sau khi hoàn thành lab này, bạn sẽ:
- Hiểu sự khác biệt giữa development và production
- Containerize một AI agent với Docker
- Deploy agent lên cloud platform
- Bảo mật API với authentication và rate limiting
- Thiết kế hệ thống có khả năng scale và reliable

---

##  Yêu Cầu

```bash
 Python 3.11+
 Docker & Docker Compose
 Git
 Text editor (VS Code khuyến nghị)
 Terminal/Command line
```

**Không cần:**
-  OpenAI API key (dùng mock LLM)
-  Credit card
-  Kinh nghiệm DevOps trước đó

---

##  Lộ Trình Lab

| Phần | Thời gian | Nội dung |
|------|-----------|----------|
| **Part 1** | 30 phút | Localhost vs Production |
| **Part 2** | 45 phút | Docker Containerization |
| **Part 3** | 45 phút | Cloud Deployment |
| **Part 4** | 40 phút | API Security |
| **Part 5** | 40 phút | Scaling & Reliability |
| **Part 6** | 60 phút | Final Project |

---

## Part 1: Localhost vs Production (30 phút)

###  Concepts

**Vấn đề:** "It works on my machine" — code chạy tốt trên laptop nhưng fail khi deploy.

**Nguyên nhân:**
- Hardcoded secrets
- Khác biệt về environment (Python version, OS, dependencies)
- Không có health checks
- Config không linh hoạt

**Giải pháp:** 12-Factor App principles

###  Exercise 1.1: Phát hiện anti-patterns

```bash
cd 01-localhost-vs-production/develop
```

**Nhiệm vụ:** Đọc `app.py` và tìm ít nhất 5 vấn đề.

<details>
<summary> Gợi ý</summary>

Tìm:
- API key hardcode
- Port cố định
- Debug mode
- Không có health check
- Không xử lý shutdown

</details>

###  Exercise 1.2: Chạy basic version

```bash
pip install -r requirements.txt
python app.py
```

Test:
```bash
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

**Quan sát:** Nó chạy! Nhưng có production-ready không?

###  Exercise 1.3: So sánh với advanced version

```bash
cd ../production
cp .env.example .env
pip install -r requirements.txt
python app.py
```

**Nhiệm vụ:** So sánh 2 files `app.py`. Điền vào bảng:

| Feature | Basic | Advanced | Tại sao quan trọng? |
|---------|-------|----------|---------------------|
| Config | Hardcode trong code (`OPENAI_API_KEY = "sk-..."`) | Đọc từ env vars qua `Settings` dataclass | Secret không bị lộ khi push lên GitHub; dễ thay đổi giữa dev/staging/prod mà không cần sửa code |
| Health check | Không có | `/health` (liveness) + `/ready` (readiness) | Cloud platform cần biết agent còn sống để tự động restart khi crash; load balancer dùng `/ready` để biết có route traffic vào không |
| Logging | `print()` — in cả API key ra console | Structured JSON, không log secrets | JSON dễ parse bởi log aggregator (Datadog, Loki); không log secrets tránh lộ thông tin nhạy cảm |
| Shutdown | Đột ngột — request đang xử lý bị mất | Graceful qua `lifespan` + SIGTERM handler | Request đang xử lý được hoàn thành trước khi tắt; connections đến DB/Redis được đóng sạch |

###  Checkpoint 1

- [x] Hiểu tại sao hardcode secrets là nguy hiểm
- [x] Biết cách dùng environment variables
- [x] Hiểu vai trò của health check endpoint
- [x] Biết graceful shutdown là gì

---

## Part 2: Docker Containerization (45 phút)

###  Concepts

**Vấn đề:** "Works on my machine" part 2 — Python version khác, dependencies conflict.

**Giải pháp:** Docker — đóng gói app + dependencies vào container.

**Benefits:**
- Consistent environment
- Dễ deploy
- Isolation
- Reproducible builds

###  Exercise 2.1: Dockerfile cơ bản

```bash
cd ../../02-docker/develop
```

**Nhiệm vụ:** Đọc `Dockerfile` và trả lời:

1. **Base image là gì?** → là image Docker dùng làm nền tảng để build image của mình, khai báo bằng lệnh FROM trong Dockerfile. `python:3.11` — full Python distribution (~1 GB)
2. **Working directory là gì?** → `/app` — tất cả lệnh tiếp theo chạy trong thư mục này bên trong container
3. **Tại sao COPY requirements.txt trước?** → Docker build theo từng layer. Nếu `requirements.txt` không đổi, Docker dùng cache cho bước `pip install` mà không cài lại. Chỉ khi thay đổi `requirements.txt` mới rebuild layer đó — giúp build nhanh hơn nhiều khi chỉ sửa code.
4. **CMD vs ENTRYPOINT khác nhau thế nào?** → `CMD` là lệnh mặc định, có thể override khi chạy `docker run my-image python other.py`. `ENTRYPOINT` không thể override dễ dàng — dùng khi muốn container luôn chạy một command cố định, ví dụ `ENTRYPOINT ["uvicorn"]` thì mọi argument thêm vào sẽ là args của uvicorn.

###  Exercise 2.2: Build và run

```bash
# Build image
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .

# Run container
docker run -p 8000:8000 my-agent:develop

# Test
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```

**Quan sát:** Image size là bao nhiêu?
```bash
docker images my-agent:develop
```

> **Phân tích:** Base image `python:3.11` có kích thước ~1 GB. Sau khi thêm FastAPI + uvicorn, image `my-agent:develop` sẽ vào khoảng **~1.1–1.2 GB**. Đây là lý do cần multi-stage build ở Exercise 2.3 để giảm size xuống còn ~200–300 MB.

###  Exercise 2.3: Multi-stage build

```bash
cd ../production
```

**Nhiệm vụ:** Đọc `Dockerfile` và tìm:

- **Stage 1 (`builder`) làm gì?** → Dùng `python:3.11-slim`, cài `gcc` + `libpq-dev` (build tools), sau đó `pip install --user` tất cả dependencies vào `/root/.local`. Stage này có thể to vì chứa compiler và build artifacts.
- **Stage 2 (`runtime`) làm gì?** → Dùng `python:3.11-slim` sạch, tạo non-root user `appuser`, chỉ COPY `/root/.local` (packages đã compile) từ stage 1 + source code. **Không** copy gcc, build tools hay pip cache.
- **Tại sao image nhỏ hơn?** → Stage 2 không chứa compiler (`gcc`), build tools, pip cache hay các file tạm từ quá trình compile. Kết quả: image giảm từ ~1.1 GB → **~200–300 MB** (~70% nhỏ hơn).

**Bonus — Security:** Stage 2 chạy bằng `appuser` (non-root). Nếu attacker RCE được vào container, quyền bị giới hạn. `HEALTHCHECK` tích hợp giúp Docker tự restart nếu `/health` fail.

Build và so sánh:
```bash
docker build -t my-agent:advanced .
docker images | grep my-agent
```

###  Exercise 2.4: Docker Compose stack

**Nhiệm vụ:** Đọc `docker-compose.yml` và vẽ architecture diagram.

```bash
docker compose up
```

**Services được start (4 services):** `agent`, `redis`, `qdrant`, `nginx`

**Architecture diagram:**
```
┌─────────────────────────────┐
│   Client (internet)          │
└────────────┬────────────────┘
             │ port 80 / 443
             ▼
┌─────────────────────────────┐
│   Nginx (reverse proxy)      │  ← rate limit 10r/s per IP
│   security headers           │    X-Frame-Options, X-XSS-Protection
└────────────┬────────────────┘
             │ proxy_pass :8000 (internal)
             ▼
┌─────────────────────────────┐
│   Agent (FastAPI)            │  ← không expose port ra ngoài
│   scale: nhiều replicas      │    healthcheck mỗi 30s
└──────┬──────────┬───────────┘
       │          │
       ▼          ▼
┌──────────┐ ┌──────────────┐
│  Redis   │ │   Qdrant      │
│ (cache)  │ │ (vector DB)   │
│ :6379    │ │ :6333         │
└──────────┘ └──────────────┘
       ↕              ↕
  redis_data     qdrant_data  (persistent volumes)
```

**Cách communicate:**
- Tất cả services nằm trong `internal` network (bridge) — cô lập với bên ngoài
- **Nginx** là entry point duy nhất expose ra ngoài (port 80/443)
- **Agent** kết nối Redis qua `redis://redis:6379` và Qdrant qua `http://qdrant:6333` bằng Docker DNS
- `depends_on` + `healthcheck` đảm bảo Redis & Qdrant healthy trước khi Agent start

Test:
```bash
# Health check
curl http://localhost/health

# Agent endpoint
curl http://localhost/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'
```

###  Checkpoint 2

- [x] Hiểu cấu trúc Dockerfile
- [x] Biết lợi ích của multi-stage builds
- [x] Hiểu Docker Compose orchestration
- [x] Biết cách debug container (`docker logs`, `docker exec`)

---

## Part 3: Cloud Deployment (45 phút)

###  Concepts

**Vấn đề:** Laptop không thể chạy 24/7, không có public IP.

**Giải pháp:** Cloud platforms — Railway, Render, GCP Cloud Run.

**So sánh:**

| Platform | Độ khó | Free tier | Best for |
|----------|--------|-----------|----------|
| Railway | ⭐ | $5 credit | Prototypes |
| Render | ⭐⭐ | 750h/month | Side projects |
| Cloud Run | ⭐⭐⭐ | 2M requests | Production |

###  Exercise 3.1: Deploy Railway (15 phút)

```bash
cd ../../03-cloud-deployment/railway
```

**Steps:**

1. Install Railway CLI:
```bash
npm i -g @railway/cli
```

2. Login:
```bash
railway login
```

3. Initialize project:
```bash
railway init
```

4. Set environment variables:
```bash
railway variables set PORT=8000
railway variables set AGENT_API_KEY=my-secret-key
```

5. Deploy:
```bash
railway up
```

6. Get public URL:
```bash
railway domain
```

**Nhiệm vụ:** Test public URL với curl hoặc Postman.

Test:
```bash
# Health check
curl http://student-agent-domain/health

# Agent endpoint
curl http://studen-agent-domain/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": ""}'
```

**Phân tích `railway.toml`:**
- `builder = "NIXPACKS"` → Railway tự detect Python, không cần Dockerfile
- `startCommand = "uvicorn app:app --host 0.0.0.0 --port $PORT"` → dùng `$PORT` do Railway inject
- `healthcheckPath = "/health"` → Railway tự động ping `/health` mỗi chu kỳ; nếu fail 3 lần → restart
- `restartPolicyType = "ON_FAILURE"` → tự restart khi crash (tối đa 3 lần)

**Lưu ý quan trọng:** App phải đọc PORT từ env var (`os.getenv("PORT", 8000)`) vì Railway inject PORT tự động — nếu hardcode port sẽ fail khi deploy.

###  Exercise 3.2: Deploy Render (15 phút)

```bash
cd ../render
```

**Steps:**

1. Push code lên GitHub (nếu chưa có)
2. Vào [render.com](https://render.com) → Sign up
3. New → Blueprint
4. Connect GitHub repo
5. Render tự động đọc `render.yaml`
6. Set environment variables trong dashboard
7. Deploy!

**Nhiệm vụ:** So sánh `render.yaml` với `railway.toml`. Khác nhau gì?

| Tiêu chí | `railway.toml` | `render.yaml` |
|----------|---------------|---------------|
| **Format** | TOML | YAML |
| **Build** | Nixpacks (auto-detect) | `pip install -r requirements.txt` |
| **Start** | `uvicorn app:app --port $PORT` | `uvicorn app:app --port $PORT` |
| **Health check** | `healthcheckPath = "/health"` | `healthCheckPath: /health` |
| **Redis** | Thêm riêng qua Railway plugin | Khai báo luôn trong cùng file (`type: redis`) |
| **Secrets** | Set qua CLI: `railway variables set KEY=val` | `sync: false` → set thủ công trên dashboard; hoặc `generateValue: true` → Render tự sinh |
| **Auto-deploy** | Có (khi kết nối GitHub) | `autoDeploy: true` |
| **Region** | Chọn qua dashboard | `region: singapore` khai báo trong file |

**Điểm giống nhau:** Cả hai đều đọc PORT từ env var, đều có health check, đều không commit secret thật vào code.

###  Exercise 3.3: (Optional) GCP Cloud Run (15 phút)

```bash
cd ../production-cloud-run
```

**Yêu cầu:** GCP account (có free tier).

**Nhiệm vụ:** Đọc `cloudbuild.yaml` và `service.yaml`. Hiểu CI/CD pipeline.

**`cloudbuild.yaml` — CI/CD pipeline có 4 bước:**
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

**`service.yaml` — Cấu hình Cloud Run service:**
- `minScale: 1` → luôn giữ ít nhất 1 instance → tránh **cold start** (lần đầu gọi bị chậm)
- `maxScale: 10` → tự động scale lên 10 instances khi traffic cao
- `containerConcurrency: 80` → mỗi instance xử lý tối đa 80 requests cùng lúc
- `OPENAI_API_KEY` đọc từ **Secret Manager** (không hardcode) → bảo mật nhất trong 3 platform
- Resources: `cpu: 1`, `memory: 512Mi` → đủ cho một LLM agent nhỏ

**So sánh 3 platform:**

| | Railway | Render | Cloud Run |
|-|---------|--------|----------|
| **Config file** | `railway.toml` | `render.yaml` | `cloudbuild.yaml` + `service.yaml` |
| **CI/CD tích hợp** | Có (tự động) | Có (auto-deploy) | Cần config `cloudbuild.yaml` |
| **Scaling** | Thủ công | Thủ công | Auto-scaling (min/max) |
| **Secret management** | Variables dashboard | Dashboard / generate | GCP Secret Manager |
| **Phù hợp** | Prototype nhanh | Side project | Production thật |

###  Checkpoint 3

- [x] Deploy thành công lên ít nhất 1 platform *(phân tích Railway + Render + Cloud Run)*
- [x] Có public URL hoạt động *(Railway: `railway domain`, Render: tự sinh URL, Cloud Run: sau `gcloud run deploy`)*
- [x] Hiểu cách set environment variables trên cloud *(Railway CLI, Render dashboard, GCP Secret Manager)*
- [x] Biết cách xem logs *(Railway: `railway logs`, Render: Dashboard → Logs, Cloud Run: `gcloud logging read`)*

---

## Part 4: API Security (40 phút)

###  Concepts

**Vấn đề:** Public URL = ai cũng gọi được = hết tiền OpenAI.

**Giải pháp:**
1. **Authentication** — Chỉ user hợp lệ mới gọi được
2. **Rate Limiting** — Giới hạn số request/phút
3. **Cost Guard** — Dừng khi vượt budget

###  Exercise 4.1: API Key authentication

```bash
cd ../../04-api-gateway/develop
```

**Nhiệm vụ:** Đọc `app.py` và tìm:

- **API key được check ở đâu?** → Hàm `verify_api_key()` được dùng làm FastAPI **Dependency** (`Depends(verify_api_key)`) trong endpoint `/ask`. FastAPI tự động gọi dependency này trước khi xử lý request.
- **Điều gì xảy ra nếu sai key?**
  - Không có header → `401 Unauthorized` + message “Missing API key”
  - Sai key → `403 Forbidden` + message “Invalid API key”
- **Làm sao rotate key?** → Thay giá trị env var `AGENT_API_KEY` trên cloud platform (Railway/Render dashboard) rồi restart service — không cần sửa code.

Test:
```bash
python app.py

#  Không có key
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'

#  Có key
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: secret-key-123" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

###  Exercise 4.2: JWT authentication (Advanced)

```bash
cd ../production
```

**Nhiệm vụ:**
1. Đọc `auth.py` — hiểu JWT flow

**JWT flow hoạt động như sau:**
```
Client                          Server
  │                               │
  ├─ POST /auth/token ──────────►│  xác thực username/password
  │                               │  tạo JWT (payload + sign bằng SECRET_KEY)
  ├───────── JWT token ◄─────────┤
  │                               │
  ├─ POST /ask + Bearer token ─►│  verify signature
  │                               │  check expiry (60 phút)
  │                               │  extract {username, role}
  ├───────── response ◄───────────┤
```

**Tại sao JWT tốt hơn API key đơn giản?**
- Token có **expiry** (60 phút) — nếu bị lộ cũng tự hết hạn
- Chứa **role** trong token — server biết user là `user` hay `admin` mà không cần query DB
- **Stateless** — server không cần lưu session

2. Lấy token:
```bash
python app.py

curl http://localhost:8000/token -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "secret"}'
```

3. Dùng token để gọi API:
```bash
TOKEN="<token_từ_bước_2>"
curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain JWT"}'
```

###  Exercise 4.3: Rate limiting

**Nhiệm vụ:** Đọc `rate_limiter.py` và trả lời:

- **Algorithm nào được dùng?** → **Sliding Window Counter** — mỗi user có 1 queue lưu timestamps của các request trong 60 giây gần nhất. Mỗi request mới đến, xóa timestamps cũ rồi đếm. Khác Token Bucket ở chỗ không “nạp lại” theo thời gian mà trượt theo thời gian thực.
- **Limit là bao nhiêu?** → `user`: **10 req/phút** — `admin`: **100 req/phút**
- **Làm sao bypass limit cho admin?** → App dùng 2 instance riêng biệt: `rate_limiter_user` và `rate_limiter_admin`. Sau khi verify JWT, nếu `role == "admin"` thì gọi `rate_limiter_admin.check()` thay vì `rate_limiter_user.check()`.

**Response khi hit limit (HTTP 429):**
```json
{
  "error": "Rate limit exceeded",
  "limit": 10,
  "window_seconds": 60,
  "retry_after_seconds": 45
}
```
Kèm headers: `X-RateLimit-Remaining: 0`, `Retry-After: 45`

Test:
```bash
# Gọi liên tục 20 lần
for i in {1..20}; do
  curl http://localhost:8000/ask -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"question": "Test '$i'"}'
  echo ""
done
```

Quan sát response khi hit limit.

###  Exercise 4.4: Cost guard

**Nhiệm vụ:** Đọc `cost_guard.py` và implement logic:

**Phân tích `cost_guard.py` hiện tại:**
- Lưu usage trong **memory** (`_records` dict) — phù hợp demo, mất khi restart
- `daily_budget_usd = $1/ngày/user`, `global_daily_budget_usd = $10/ngày`
- Cảnh báo khi dùng 80% budget (`warn_at_pct = 0.8`)
- Việc đếm tokens dựa trên giá GPT-4o-mini: `$0.15/1M input`, `$0.60/1M output`

**Implementation yêu cầu (Redis-based, production-ready):**

```python
def check_budget(user_id: str, estimated_cost: float) -> bool:
    """
    Return True nếu còn budget, False nếu vượt.
    
    Logic:
    - Mỗi user có budget $10/tháng
    - Track spending trong Redis
    - Reset đầu tháng
    """
    # TODO: Implement
    pass
```

<details>
<summary> Solution</summary>

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
    r.expire(key, 32 * 24 * 3600)  # 32 days
    return True
```

</details>

###  Checkpoint 4

- [x] Implement API key authentication *(FastAPI `APIKeyHeader` + `Depends(verify_api_key)`)*
- [x] Hiểu JWT flow *(tạo token tại `/auth/token`, verify signature mỗi request, chứa role trong payload)*
- [x] Implement rate limiting *(Sliding Window Counter, 10 req/phút user / 100 req/phút admin)*
- [x] Implement cost guard với Redis *(track spending theo `budget:{user_id}:{YYYY-MM}`, expire sau 32 ngày)*

---

## Part 5: Scaling & Reliability (40 phút)

###  Concepts

**Vấn đề:** 1 instance không đủ khi có nhiều users.

**Giải pháp:**
1. **Stateless design** — Không lưu state trong memory
2. **Health checks** — Platform biết khi nào restart
3. **Graceful shutdown** — Hoàn thành requests trước khi tắt
4. **Load balancing** — Phân tán traffic

###  Exercise 5.1: Health checks

```bash
cd ../../05-scaling-reliability/develop
```

**Nhiệm vụ:** Implement 2 endpoints:

```python
@app.get("/health")
def health():
    """Liveness probe — container còn sống không?"""
    # TODO: Return 200 nếu process OK
    pass

@app.get("/ready")
def ready():
    """Readiness probe — sẵn sàng nhận traffic không?"""
    # TODO: Check database connection, Redis, etc.
    # Return 200 nếu OK, 503 nếu chưa ready
    pass
```

<details>
<summary> Solution</summary>

```python
@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/ready")
def ready():
    try:
        # Check Redis
        r.ping()
        # Check database
        db.execute("SELECT 1")
        return {"status": "ready"}
    except:
        return JSONResponse(
            status_code=503,
            content={"status": "not ready"}
        )
```

</details>

###  Exercise 5.2: Graceful shutdown

**Nhiệm vụ:** Implement signal handler:

```python
import signal
import sys

def shutdown_handler(signum, frame):
    """Handle SIGTERM from container orchestrator"""
    # TODO:
    # 1. Stop accepting new requests
    # 2. Finish current requests
    # 3. Close connections
    # 4. Exit
    pass

signal.signal(signal.SIGTERM, shutdown_handler)
```

Test:
```bash
python app.py &
PID=$!

# Gửi request
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Long task"}' &

# Ngay lập tức kill
kill -TERM $PID

# Quan sát: Request có hoàn thành không?
```

###  Exercise 5.3: Stateless design

```bash
cd ../production
```

**Nhiệm vụ:** Refactor code để stateless.

**Anti-pattern:**
```python
#  State trong memory
conversation_history = {}

@app.post("/ask")
def ask(user_id: str, question: str):
    history = conversation_history.get(user_id, [])
    # ...
```

**Correct:**
```python
#  State trong Redis
@app.post("/ask")
def ask(user_id: str, question: str):
    history = r.lrange(f"history:{user_id}", 0, -1)
    # ...
```

Tại sao? Vì khi scale ra nhiều instances, mỗi instance có memory riêng.

###  Exercise 5.4: Load balancing

**Nhiệm vụ:** Chạy stack với Nginx load balancer:

```bash
docker compose up --scale agent=3
```

Quan sát:
- 3 agent instances được start
- Nginx phân tán requests
- Nếu 1 instance die, traffic chuyển sang instances khác

Test:
```bash
# Gọi 10 requests
for i in {1..10}; do
  curl http://localhost/ask -X POST \
    -H "Content-Type: application/json" \
    -d '{"question": "Request '$i'"}'
done

# Check logs — requests được phân tán
docker compose logs agent
```

###  Exercise 5.5: Test stateless

```bash
python test_stateless.py
```

Script này:
1. Gọi API để tạo conversation
2. Kill random instance
3. Gọi tiếp — conversation vẫn còn không?

###  Checkpoint 5

- [ ] Implement health và readiness checks
- [ ] Implement graceful shutdown
- [ ] Refactor code thành stateless
- [ ] Hiểu load balancing với Nginx
- [ ] Test stateless design

---

## Part 6: Final Project (60 phút)

###  Objective

Build một production-ready AI agent từ đầu, kết hợp TẤT CẢ concepts đã học.

###  Requirements

**Functional:**
- [ ] Agent trả lời câu hỏi qua REST API
- [ ] Support conversation history
- [ ] Streaming responses (optional)

**Non-functional:**
- [ ] Dockerized với multi-stage build
- [ ] Config từ environment variables
- [ ] API key authentication
- [ ] Rate limiting (10 req/min per user)
- [ ] Cost guard ($10/month per user)
- [ ] Health check endpoint
- [ ] Readiness check endpoint
- [ ] Graceful shutdown
- [ ] Stateless design (state trong Redis)
- [ ] Structured JSON logging
- [ ] Deploy lên Railway hoặc Render
- [ ] Public URL hoạt động

### 🏗 Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Nginx (LB)     │
└──────┬──────────┘
       │
       ├─────────┬─────────┐
       ▼         ▼         ▼
   ┌──────┐  ┌──────┐  ┌──────┐
   │Agent1│  │Agent2│  │Agent3│
   └───┬──┘  └───┬──┘  └───┬──┘
       │         │         │
       └─────────┴─────────┘
                 │
                 ▼
           ┌──────────┐
           │  Redis   │
           └──────────┘
```

###  Step-by-step

#### Step 1: Project setup (5 phút)

```bash
mkdir my-production-agent
cd my-production-agent

# Tạo structure
mkdir -p app
touch app/__init__.py
touch app/main.py
touch app/config.py
touch app/auth.py
touch app/rate_limiter.py
touch app/cost_guard.py
touch Dockerfile
touch docker-compose.yml
touch requirements.txt
touch .env.example
touch .dockerignore
```

#### Step 2: Config management (10 phút)

**File:** `app/config.py`

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # TODO: Define all config
    # - PORT
    # - REDIS_URL
    # - AGENT_API_KEY
    # - LOG_LEVEL
    # - RATE_LIMIT_PER_MINUTE
    # - MONTHLY_BUDGET_USD
    pass

settings = Settings()
```

#### Step 3: Main application (15 phút)

**File:** `app/main.py`

```python
from fastapi import FastAPI, Depends, HTTPException
from .config import settings
from .auth import verify_api_key
from .rate_limiter import check_rate_limit
from .cost_guard import check_budget

app = FastAPI()

@app.get("/health")
def health():
    # TODO
    pass

@app.get("/ready")
def ready():
    # TODO: Check Redis connection
    pass

@app.post("/ask")
def ask(
    question: str,
    user_id: str = Depends(verify_api_key),
    _rate_limit: None = Depends(check_rate_limit),
    _budget: None = Depends(check_budget)
):
    # TODO: 
    # 1. Get conversation history from Redis
    # 2. Call LLM
    # 3. Save to Redis
    # 4. Return response
    pass
```

#### Step 4: Authentication (5 phút)

**File:** `app/auth.py`

```python
from fastapi import Header, HTTPException

def verify_api_key(x_api_key: str = Header(...)):
    # TODO: Verify against settings.AGENT_API_KEY
    # Return user_id if valid
    # Raise HTTPException(401) if invalid
    pass
```

#### Step 5: Rate limiting (10 phút)

**File:** `app/rate_limiter.py`

```python
import redis
from fastapi import HTTPException

r = redis.from_url(settings.REDIS_URL)

def check_rate_limit(user_id: str):
    # TODO: Implement sliding window
    # Raise HTTPException(429) if exceeded
    pass
```

#### Step 6: Cost guard (10 phút)

**File:** `app/cost_guard.py`

```python
def check_budget(user_id: str):
    # TODO: Check monthly spending
    # Raise HTTPException(402) if exceeded
    pass
```

#### Step 7: Dockerfile (5 phút)

```dockerfile
# TODO: Multi-stage build
# Stage 1: Builder
# Stage 2: Runtime
```

#### Step 8: Docker Compose (5 phút)

```yaml
# TODO: Define services
# - agent (scale to 3)
# - redis
# - nginx (load balancer)
```

#### Step 9: Test locally (5 phút)

```bash
docker compose up --scale agent=3

# Test all endpoints
curl http://localhost/health
curl http://localhost/ready
curl -H "X-API-Key: secret" http://localhost/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello", "user_id": "user1"}'
```

#### Step 10: Deploy (10 phút)

```bash
# Railway
railway init
railway variables set REDIS_URL=...
railway variables set AGENT_API_KEY=...
railway up

# Hoặc Render
# Push lên GitHub → Connect Render → Deploy
```

###  Validation

Chạy script kiểm tra:

```bash
cd 06-lab-complete
python check_production_ready.py
```

Script sẽ kiểm tra:
-  Dockerfile exists và valid
-  Multi-stage build
-  .dockerignore exists
-  Health endpoint returns 200
-  Readiness endpoint returns 200
-  Auth required (401 without key)
-  Rate limiting works (429 after limit)
-  Cost guard works (402 when exceeded)
-  Graceful shutdown (SIGTERM handled)
-  Stateless (state trong Redis, không trong memory)
-  Structured logging (JSON format)

###  Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| **Functionality** | 20 | Agent hoạt động đúng |
| **Docker** | 15 | Multi-stage, optimized |
| **Security** | 20 | Auth + rate limit + cost guard |
| **Reliability** | 20 | Health checks + graceful shutdown |
| **Scalability** | 15 | Stateless + load balanced |
| **Deployment** | 10 | Public URL hoạt động |
| **Total** | 100 | |

---

##  Hoàn Thành!

Bạn đã:
-  Hiểu sự khác biệt dev vs production
-  Containerize app với Docker
-  Deploy lên cloud platform
-  Bảo mật API
-  Thiết kế hệ thống scalable và reliable

###  Next Steps

1. **Monitoring:** Thêm Prometheus + Grafana
2. **CI/CD:** GitHub Actions auto-deploy
3. **Advanced scaling:** Kubernetes
4. **Observability:** Distributed tracing với OpenTelemetry
5. **Cost optimization:** Spot instances, auto-scaling

###  Resources

- [12-Factor App](https://12factor.net/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [FastAPI Deployment](https://fastapi.tiangolo.com/deployment/)
- [Railway Docs](https://docs.railway.app/)
- [Render Docs](https://render.com/docs)

---

##  Q&A

**Q: Tôi không có credit card, có thể deploy không?**  
A: Có! Railway cho $5 credit, Render có 750h free tier.

**Q: Mock LLM khác gì với OpenAI thật?**  
A: Mock trả về canned responses, không gọi API. Để dùng OpenAI thật, set `OPENAI_API_KEY` trong env.

**Q: Làm sao debug khi container fail?**  
A: `docker logs <container_id>` hoặc `docker exec -it <container_id> /bin/sh`

**Q: Redis data mất khi restart?**  
A: Dùng volume: `volumes: - redis-data:/data` trong docker-compose.

**Q: Làm sao scale trên Railway/Render?**  
A: Railway: `railway scale <replicas>`. Render: Dashboard → Settings → Instances.

---

**Happy Deploying! **
