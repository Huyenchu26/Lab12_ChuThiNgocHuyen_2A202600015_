# Deployment Information

## Public URL
https://lab12-agent-production-4620.up.railway.app

## Platform
Railway / Render / Cloud Run

## Test Commands

### Health Check
```bash
curl https://lab12-agent-production-4620.up.railway.app/health

{"status":"ok","version":"1.0.0","environment":"development","uptime_seconds":1724.7,"total_requests":8,"checks":{"llm":"mock"},"timestamp":"2026-04-17T10:46:03.550484+00:00"}
```

### API Test (with authentication)
```bash
curl -X POST https://lab12-agent-production-4620.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me-in-production" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'

{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","model":"gpt-4o-mini","timestamp":"2026-04-17T10:46:29.104612+00:00"}
```

## Environment Variables Set
- PORT=8000
- AGENT_API_KEY=dev-key-change-me-in-production
- LOG_LEVEL=Info

## Screenshots
- [Deployment dashboard](dashboard.png)
- [Service running](deploy-railway.png)
- [Test results](test.png)