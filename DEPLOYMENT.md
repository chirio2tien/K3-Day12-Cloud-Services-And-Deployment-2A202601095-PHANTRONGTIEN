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
| Public URL | http://localhost:8000 (LOCAL_FALLBACK — chưa đẩy lên cloud miễn phí) |
| Platform | Railway (cấu hình sẵn `railway.toml`; chạy local bằng Docker Compose khi cloud chưa sẵn sàng) |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard / file `.env` local, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis service trong docker compose (`redis://redis:6379/0`) / Redis add-on trên Railway |
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
  -d '{"question":"Deploy là gì?"}'

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

```
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP/1.1 200 OK
{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

HTTP/1.1 200 OK
{"answer":"...","user_id":"sv-test","history_length":0,"cost_usd":...,"tokens":{...}}

Rate limit probe: 200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — `docker compose ps` khi stack healthy
- `screenshots/health.png` — kết quả gọi `/health`

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký kịp tài khoản cloud / thẻ xác minh: dùng phương án dự phòng (CP5 tối đa 60% điểm).

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — test kiểm tra `http://localhost:8000`
5. Lý do: tài khoản Railway/Render yêu cầu xác minh thanh toán trong khung giờ lab; stack production-ready đã chạy ổn định trên Docker Compose local với Redis thật, đủ để chứng minh `/health`, `/ready`, auth 401.
