# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Phan Trọng Tiến |
| Mã học viên | 2A202601095 |
| Repo | https://github.com/chirio2tien/K3-Day12-Cloud-Services-And-Deployment-2A202601095-PHANTRONGTIEN |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-0900.up.railway.app |
| Platform | Railway |
| Region | Southeast Asia (Singapore) — chuyển khỏi US West vì incident networking |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong Railway Variables, không nằm trong repo |
| `REDIS_URL` | ✅ | reference tới Redis plugin: `${{Redis.REDIS_URL}}` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://agent-production-0900.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"}
curl -i https://agent-production-0900.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://agent-production-0900.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://agent-production-0900.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

```
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP/1.1 200 OK
{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

Uvicorn running on http://0.0.0.0:8080
region: Southeast Asia · status: Online
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — stack Docker Compose / dashboard khi verify local
- `screenshots/health.png` — kết quả gọi `/health` và `/ready`

## Sự cố deploy đã xử lý

1. Railway US West partial outage → chuyển replica sang Southeast Asia.
2. Healthcheck fail vì `railway.toml` dùng `uvicorn ... --port $PORT` không qua shell → uvicorn nhận literal `$PORT`. Sửa thành `sh -c 'uvicorn ... --port ${PORT:-8000}'`.
