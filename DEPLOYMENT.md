# Deployment Information

## Public URL
https://day12-part6-production.up.railway.app

## Platform
Railway

## Test Commands

### 1. Health Check
```bash
curl https://day12-part6-production.up.railway.app/health
```
**Expected Response:**
```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "development",
  "uptime_seconds": 12.5,
  "total_requests": 1,
  "checks": {
    "llm": "mock"
  }
}
```

### 2. API Test (Without Authentication)
```bash
curl -X POST https://day12-part6-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```
**Expected Response:**
```json
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
```

### 3. API Test (With Incorrect Authentication)
```bash
curl -X POST https://day12-part6-production.up.railway.app/ask \
  -H "X-API-Key: wrong-key-123" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```
**Expected Response:**
```json
{"detail":"Invalid API key."}
```

### 4. API Test (With Correct Authentication)
```bash
curl -X POST https://day12-part6-production.up.railway.app/ask \
  -H "X-API-Key: 9m_yd_e7BOnniQMSEBBknQ" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```
**Expected Response:**
```json
{
  "question": "What is Docker?",
  "answer": "Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere! (mock response)",
  "model": "gpt-4o-mini",
  "timestamp": "2026-06-12T10:57:09.298Z"
}
```

### 5. Rate Limiting Test (Send >10 requests within a minute)
```bash
# Gửi liên tiếp 12 requests bằng loop
for i in {1..12}; do
  curl -s -X POST https://day12-part6-production.up.railway.app/ask \
    -H "X-API-Key: 9m_yd_e7BOnniQMSEBBknQ" \
    -H "Content-Type: application/json" \
    -d '{"question": "Test '$i'"}'
  echo ""
done
```
**Expected Response at 11th request:**
```json
{"detail":"Rate limit exceeded: 20 req/min"}
```

---

## Environment Variables Set
- `PORT` (Railway tự động gán động)
- `REDIS_URL` (Railway tự động gán sau khi add database Redis)
- `AGENT_API_KEY` = `9m_yd_e7BOnniQMSEBBknQ`
- `ENVIRONMENT` = `development`
- `DEBUG` = `false`

---

## Screenshots
- [Deployment Dashboard](screenshots/dashboard.png)
- [Service Running](screenshots/running.png)
- [Test Results](screenshots/test.png)
