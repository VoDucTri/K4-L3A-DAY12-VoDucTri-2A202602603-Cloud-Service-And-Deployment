# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Võ Đức Trí |
| Mã học viên | 2A202602603 |
| Repo | https://github.com/VoDucTri/K4-L3A-DAY12-VoDucTri-2A202602603-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-day12-agent.onrender.com |
| Platform | Render / Docker Compose (Local Fallback) |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán (đọc qua ${PORT:-8000}) |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard bí mật, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on / connection string từ internal redis service |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i http://localhost:8000/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i http://localhost:8000/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy la gi?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST http://localhost:8000/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```http
# 1. GET /health
HTTP/1.1 200 OK
date: Mon, 28 Sep 2026 08:52:33 GMT
server: uvicorn
content-length: 57
content-type: application/json

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. GET /ready
HTTP/1.1 200 OK
date: Mon, 28 Sep 2026 08:52:39 GMT
server: uvicorn
content-length: 31
content-type: application/json

{"status":"ready","redis":true}

# 3. POST /ask (không có API key)
HTTP/1.1 401 Unauthorized
date: Mon, 28 Sep 2026 08:52:52 GMT
server: uvicorn
content-length: 39
content-type: application/json

{"detail":"invalid or missing API key"}

# 4. POST /ask (có API key hợp lệ)
HTTP/1.1 200 OK
date: Mon, 28 Sep 2026 08:53:12 GMT
server: uvicorn
content-type: application/json

{
  "answer": "Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên. (Mình đang nhớ 2 lượt trao đổi trước đó.)",
  "user_id": "sv-test",
  "history_length": 2,
  "cost_usd": 3.465e-05,
  "tokens": {"in": 43, "out": 47}
}

# 5. Rate limit test (15 requests liên tiếp)
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform / docker compose ps
- `screenshots/health.png` — kết quả gọi `/health` và `/ready` từ curl

---

## Phương Án Dự Phòng

Khi chạy phương án dự phòng cục bộ:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test tự chuyển sang kiểm tra `http://localhost:8000`
5. Lý do sử dụng phương án dự phòng:

```text
Hệ thống đã được đóng gói hoàn chỉnh bằng Docker Compose và kiểm thử toàn diện mọi tiêu chí trên môi trường cục bộ (bao gồm multi-stage build, container agent non-root, redis cluster, rate limit, cost guard, readiness & graceful shutdown). Stack container hoạt động ổn định và sẵn sàng deploy lên Render / Railway thông qua file render.yaml và railway.toml đã định cấu hình.
```
