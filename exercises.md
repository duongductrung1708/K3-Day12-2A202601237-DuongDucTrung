# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dương Đức Trung Mã học viên: 2A202601237.

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên production, nếu để mặc định "changeme" thì app sẽ khởi động bình thường với key giả. Bất kỳ ai cũng có thể gọi API bằng key này, dẫn đến bill tăng đột biến hoặc dữ liệu bị lộ mà tôi không biết cho đến khi quá muộn. Với fail-fast (không có default), nếu quên set AGENT_API_KEY thật, app sẽ crash ngay tại startup với lỗi rõ ràng, buộc tôi phải fix config trước khi deploy tiếp.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON: {"timestamp": "2026-08-10T03:03:07.136221+00:00", "level": "info", "event": "ask_completed", "user_id": "sv01", "tokens_in": 10, "tokens_out": 20, "cost_usd": 0.0001}

1. Lọc theo user_id hoặc level trong vài giây (Grafana, Datadog parse được)
2. Tìm kiếm các request có cost_usd > 0.01 để phát hiện usage bất thường

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản               | Dung lượng |
| ----------------- | ---------- |
| 1 stage (bản đầu) | ~800 MB    |
| Multi-stage       | ~64 MB     |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch (~736 MB) là compiler, build tools, pip cache, và source code build tools từ stage builder. Multi-stage build chỉ copy kết quả đã cài đặt sang stage runtime, không mang theo những thứ không cần thiết để chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại (COPY requirements.txt trước pip install): chỉ layer COPY app/ app/ và COPY utils/ utils/ phải chạy lại vì source code đã đổi. Các layer COPY requirements.txt và RUN pip install vẫn dùng cache.

Nếu đặt COPY . . trước RUN pip install: layer COPY . . bị invalidate khi sửa code → layer RUN pip install cũng bị theo → phải cài lại toàn bộ thư viện mỗi lần sửa 1 dòng code, build mất 1-2 phút thay vì vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Lỗ hổng trong code → kẻ tấn công escape khỏi container → có quyền root trong container → leo thang ra máy host thông qua container runtime hoặc mount volumes → kiểm soát toàn bộ hệ thống.

Lệnh USER cắt đứt chuỗi này ở bước: container chạy bằng user thường (appuser) thay vì root →即使 kẻ tấn công escape khỏi container, họ chỉ có quyền user thường, không có quyền root trên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với fixed window (reset lúc giây 00), user có thể gửi 10 request ở giây 59 và 10 request ở giây 00 = 20 request trong 2 giây mà vẫn "đúng luật". Cách đạt được: gửi 10 request ngay trước khi phút kết thúc, rồi ngay khi phút mới bắt đầu gửi thêm 10 request nữa.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số lượng request/phút (bảo vệ hạ tầng khỏi DDoS). Cost guard giới hạn số tiền/tháng (bảo vệ ngân sách).

Rate limit cho qua nhưng cost guard chặn: user gửi 1 request/phút (không vi phạm rate limit) nhưng mỗi request tốn 50k token ($5/request) → sau 2 request đã hết budget $10 → request thứ 3 bị 402.

Cost guard cho qua nhưng rate limit chặn: user gửi 20 request trong 10 giây (vượt rate limit) nhưng mỗi request chỉ tốn 100 token ($0.0001) → tổng chi phí rất thấp nhưng vẫn bị 429 vì gửi quá nhanh.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Redis mất kết nối 30 giây → /health trả 503 vì nó kiểm tra Redis → load balancer nghĩ cả 3 container đều chết → restart tất cả 3 container → user đang xử lý request bị cắt giữa chừng → trải nghiệm người dùng bị gián đoạn dù chỉ là lỗi tạm thời của Redis.

Đúng thứ tự: Redis mất kết nối → /health trả 503 → load balancer pull traffic → orchestrator restart container → request đang chạy bị hủy.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với dict Python: history_length sẽ nhảy loạn xạ (1, 1, 1, 2, 2) vì mỗi container có dict riêng. Nginx điều phối round-robin nên request rơi vào container khác nhau không thấy lịch sử của nhau.

Với Redis: history_length tăng đều (1, 2, 3, 4, 5) vì tất cả container cùng nhìn vào một Redis instance, lịch sử được chia sẻ giữa các container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi: Health check timeout khi deploy lên Railway.

Thông báo lỗi: Container health check failed after 3 retries.

Tìm nguyên nhân: Kiểm tra log Railway thấy health check gọi /health nhưng không có response. Đọc code phát hiện /health đang phụ thuộc Redis ping, nhưng Railway Redis add-on chưa sẵn sàng khi container start.

Cách sửa: Tách /health (không gọi Redis) và /ready (gọi Redis ping). /health chỉ kiểm tra process còn sống, /ready mới kiểm tra dependency. Sau khi sửa, health check pass.
