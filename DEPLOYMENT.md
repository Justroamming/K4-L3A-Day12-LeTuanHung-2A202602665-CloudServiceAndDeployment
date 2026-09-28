# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Lê Tuấn Hưng |
| Mã học viên | 2A202602665 |
| Repo | https://github.com/Justroamming/K4-L3A-Day12-LeTuanHung-2A202602665-CloudServiceAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-y83p.onrender.com |
| Platform | Render
| Ngày deploy | 28/09/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Redis add-on |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-y83p.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-y83p.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-y83p.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-y83p.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-y83p.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness 
curl -i https://day12-agent-y83p.onrender.com/health
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 10:03:11 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
rndr-id: 5a34aab7-1002-4a93
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-cache-status: DYNAMIC
CF-RAY: a421eab32f0f06fa-HKG
alt-svc: h3=":443"; ma=86400

{"status":"ok","service":"day12-agent","version":"1.0.0"}

2. Readiness
curl -i https://day12-agent-y83p.onrender.com/ready
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 10:04:29 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: bdf71f0d-c2b3-4e26
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a421ec9879a9fbc9-HKG
alt-svc: h3=":443"; ma=86400

{"status":"ready","redis":true}

# 3. Không có API key
curl -i -X POST https://day12-agent-y83p.onrender.com/ask ^
More?   -H "Content-Type: application/json" ^
More?   -d "{\"question\":\"Hello\"}"
HTTP/1.1 401 Unauthorized
Date: Mon, 28 Sep 2026 10:06:16 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: aba37841-7f3d-496a
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a421ef3aa9ccddc4-HKG
alt-svc: h3=":443"; ma=86400

{"detail":"invalid or missing API key"}

4.
curl -i -X POST https://day12-agent-y83p.onrender.com/ask ^
More?   -H "Content-Type: application/json" ^
More?   -H "X-API-Key: %AGENT_API_KEY%" ^
More?   -H "X-User-Id: sv-test" ^
More?   -d "{\"question\":\"Deploy là gì?\"}"
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 10:19:23 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 63868211-9164-4413
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a422026e3b1f9b7e-HKG
alt-svc: h3=":443"; ma=86400

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

5.
for /L %i in (1,1,15) do @curl -s -o NUL -w "%{http_code} " -X POST https://day12-agent-y83p.onrender.com/ask -H "Content-Type: application/json" -H "X-API-Key: %AGENT_API_KEY%" -H "X-User-Id: sv-test" -d "{\"question\":\"test\"}"
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429 
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---
