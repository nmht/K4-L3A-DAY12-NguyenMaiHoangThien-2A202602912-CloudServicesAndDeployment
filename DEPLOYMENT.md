# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Mai Hoàng Thiện |
| Mã học viên | 2A202602912 |
| Repo | https://github.com/nmht/K4-L3A-DAY12-NguyenMaiHoangThien-2A202602912-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-c5c9.up.railway.app |
| Platform | Railway |
| Ngày deploy | 28/09/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | redis://default:<password>@day12-redis.railway.internal:6379 |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-production-c5c9.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-production-c5c9.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-production-c5c9.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-production-c5c9.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-production-c5c9.up.railway.app/ask \
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
Content-Type: application/json
Date: Mon, 28 Sep 2026 08:35:17 GMT
Server: railway-hikari
x-railway-request-id: k7xFycdfRQueS3rUCYBc-A
Content-Length: 57
x-hikari-trace: sin1.d1nj
x-railway-edge: sin1
Connection: keep-alive

{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Mon, 28 Sep 2026 09:04:10 GMT
Server: railway-hikari
x-railway-request-id: OpXeZtF7RI6tXReoWUN5dQ
Content-Length: 39
x-hikari-trace: sin1.98a6
x-railway-edge: sin1
Connection: keep-alive

{"detail":"invalid or missing API key"}

HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 28 Sep 2026 09:04:11 GMT
Server: railway-hikari
x-railway-request-id: VyQnJVFMTLyN_6-eljLL4A
Content-Length: 279
x-hikari-trace: sin1.hs0s
x-railway-edge: sin1
vary: accept-encoding
Connection: keep-alive

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```

```
Theo nguyên tắc Stateless (Phi trạng thái) của 12-factor app, các instance của ứng dụng không được phép lưu trữ state (như lịch sử trò chuyện, rate limit quota) trong RAM của chính nó. Nếu không tách Redis ra:

Khi container bị restart hoặc crash, toàn bộ dữ liệu lịch sử sẽ bị mất (bị "mất trí nhớ").
Khi scale hệ thống lên nhiều instance (ví dụ 3 containers), một user gọi API lần 1 vào container A, lần 2 request rơi vào container B sẽ không thấy lịch sử cũ đâu. Việc tách Redis ra làm một service độc lập đóng vai trò là "Single Source of Truth", đảm bảo tính nhất quán của State giữa mọi instance và không bị mất khi ứng dụng khởi động lại.
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

*(Đã deploy thành công trên Railway nên bỏ qua phần này)*
