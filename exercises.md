# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết câu trả lời ngay dưới mỗi câu hỏi (không để placeholder trống).
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phan Trọng Tiến  Mã học viên: 2A202601095

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên Railway, nếu quên set `AGENT_API_KEY` trên dashboard thì
container crash ngay lúc start với `ValidationError`. Tôi nhìn log 5 giây là
biết thiếu secret. Nếu để mặc định `"changeme"`, service vẫn "healthy",
bot quét Internet gọi `/ask` bằng khóa mặc định, và tôi chỉ phát hiện khi
nhìn hóa đơn hoặc khi quota tháng đã cạn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:20:11.123456+00:00", "user_id": "sv-test", "tokens_in": 4, "tokens_out": 12, "cost_usd": 0.0000078}
```

1) Lọc theo `user_id` / `event` trên Cloud log để biết ai gọi nhiều nhất.
2) Cộng dồn `cost_usd` theo cửa sổ thời gian để cảnh báo khi sắp vượt ngân sách.
`print("đã trả lời xong")` không có trường có cấu trúc nên không query được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.02 GB (`python:3.11` full + COPY hết context) |
| Multi-stage | 270 MB (`python:3.11-slim`, đo thật bằng `docker images`) |

Phần chênh lệch chủ yếu là toolchain/compiler, header hệ thống và các layer
thừa của base image đầy đủ. Multi-stage chỉ mang sang runtime những gì
`pip install --prefix=/install` tạo ra, nên image nhỏ và deploy nhanh hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại: layer `COPY requirements.txt` và `RUN pip install`
được cache lại vì `requirements.txt` không đổi. Chỉ các layer sau
(`COPY app`, `COPY utils`) chạy lại. Nếu đặt `COPY . .` trước `pip install`,
mỗi lần sửa một dòng code hash context đổi → Docker hủy cache và cài lại toàn
bộ dependency, build chậm hơn rõ rệt.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu app có RCE (ví dụ deserialize không an toàn), process trong container đang
là root → kẻ tấn công có full quyền trong container, dễ mount/đọc secret,
cài backdoor, và nếu có lỗ hổng escape/kernel thì lên được host với quyền cao.
Lệnh `USER appuser` cắt chuỗi ngay từ đầu: dù RCE thành công, process cũng chỉ
là user thường, không sẵn quyền root để thao túng filesystem hệ thống.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong ~2 giây: gửi 10 request lúc `xx:00:59` (hết quota phút
hiện tại), sang `xx:01:01` bộ đếm reset về 0 nên gửi thêm 10 request nữa. Sliding
window không có khe hở biên phút này vì luôn đếm đúng 60 giây gần nhất.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn **tần suất** (số request/phút). Cost guard giới hạn **tiền**
(USD/tháng). Ví dụ rate limit cho qua nhưng cost guard chặn: 5 request/phút
nhưng mỗi câu rất dài (nhiều token) khiến tổng chi phí vượt ngân sách tháng.
Ngược lại: request rẻ nhưng spam 100 lần/phút — cost còn dư mà rate limit trả
429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1) Redis chết → cả 3 `/health` trả 503 vì phụ thuộc Redis.
2) Orchestrator hiểu là liveness fail → restart cả 3 container cùng lúc.
3) Trong lúc restart, không còn instance phục vụ → outage toàn cụm.
4) Redis hồi phục nhưng traffic vẫn mất trong cửa sổ restart.
Tách `/ready` thì LB chỉ ngừng đẩy request, không restart hàng loạt.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis dùng chung, `history_length` tăng đều (0 → 2 → 4...) bất kể request
đi vào instance nào. Nếu lưu dict trong process, mỗi instance có bộ nhớ riêng:
request nhảy giữa A/B/C sẽ thấy `history_length` nhảy lung tung hoặc "quên"
hội thoại tùy container vừa nhận request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Deploy Railway lần đầu healthcheck fail: log ghi
`Invalid value for '--port': '$PORT' is not a valid integer`.
`railway.toml` đặt `uvicorn ... --port $PORT` ở dạng exec nên shell không expand
biến. Sửa thành `sh -c 'uvicorn ... --port ${PORT:-8000}'`, chuyển region sang
Southeast Asia (tránh incident US West), redeploy → `/health` và `/ready` 200.
