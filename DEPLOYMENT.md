# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục         | Nội dung                                                                |
| ----------- | ----------------------------------------------------------------------- |
| Họ và tên   | Dương Đức Trung                                                         |
| Mã học viên | 2A202601237                                                             |
| Repo        | https://github.com/duongductrung1708/K3-Day12-2A202601237-DuongDucTrung |

## Service

| Mục         | Nội dung                              |
| ----------- | ------------------------------------- |
| Public URL  | https://day12-agent-0oan.onrender.com |
| Platform    | Render                                |
| Ngày deploy | 2026-08-10                            |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến                    | Đã set | Ghi chú                                       |
| ----------------------- | ------ | --------------------------------------------- |
| `PORT`                  | ✅     | đặt trong .env file                           |
| `AGENT_API_KEY`         | ✅     | đặt trong .env file, không nằm trong repo     |
| `REDIS_URL`             | ✅     | redis://redis:6379/0 (Docker Compose service) |
| `RATE_LIMIT_PER_MINUTE` | ✅     | 10                                            |
| `MONTHLY_BUDGET_USD`    | ✅     | 10.0                                          |
| `LOG_LEVEL`             | ✅     | INFO                                          |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP/1.1 200 OK
{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

HTTP/1.1 200 OK
{"answer":"Ngắn gọn: Hello thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên.","user_id":"WZDvAfEptHDT8FBPQu6x0Ky6kjWRZhh3pGCFGL4vgxo","history_length":0,"cost_usd":0.0001,"tokens":{"in":10,"out":20}}

200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Sử dụng phương án dự phòng do không có tài khoản cloud (Railway/Render) sẵn có.
Đã deploy thành công bằng Docker Compose với 3 agent replicas và Redis service.
```
