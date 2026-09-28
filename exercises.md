# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).Họ và tên: Nguyễn Mai Hoàng Thiện  Mã học viên: 2A202602912

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: tôi deploy lên Railway nhưng quên set biến `AGENT_API_KEY` trên dashboard cloud. Nếu để mặc định `"changeme"`, app vẫn khởi động bình thường — health check xanh, không có cảnh báo nào. Bất kỳ ai biết giá trị `"changeme"` (hoặc đoán thử) đều gọi được `/ask` miễn phí bằng tài khoản LLM của tôi, đốt ngân sách mà tôi không hay biết. Với `agent_api_key` không có mặc định, app crash ngay lúc khởi động với lỗi `ValidationError: agent_api_key field required` — Railway báo deploy thất bại, tôi nhận thông báo ngay và phải đi set key đúng trước khi có traffic nào đến service.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được sau khi gọi `/ask`:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:04:11.342178+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`Hai việc làm được mà `print("đã trả lời xong")` không làm được:

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
| 1 stage (bản đầu) | 285 MB     |
| Multi-stage       | 178 MB     |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch \~107 MB đến từ các công cụ build mà stage `builder` cài vào nhưng stage `runtime` không cần: compiler C/C++ (`gcc`, `g++`), header file hệ thống (`python3-dev`), các file `.pyc` và metadata tạm thời tạo ra khi biên dịch các package như `pydantic-core`. Trong multi-stage build, chỉ thư mục `/install` chứa file `.so` và bytecode đã biên dịch được `COPY --from=builder` sang — toàn bộ toolchain và file trung gian bị bỏ lại ở stage builder, không bao giờ xuất hiện trong image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại (đúng thứ tự): khi tôi sửa một ký tự trong `app/main.py` và build lại, Docker dùng lại cache của các layer `FROM python:3.11-slim AS builder`, `WORKDIR /app`, `COPY requirements.txt .` và `RUN pip install --prefix=/install`. Chỉ layer `COPY . .` ở stage runtime trở đi phải chạy lại vì checksum của source code thay đổi — build mất khoảng 5 giây thay vì vài phút.Nếu đặt `COPY . .` lên **trước** `RUN pip install`: mỗi lần sửa bất kỳ dòng code nào, Docker thấy COPY layer thay đổi → cache miss → `RUN pip install -r requirements.txt` phải chạy lại từ đầu, tải lại toàn bộ thư viện qua mạng, mất 2–3 phút. Đây là lý do cần COPY requirements trước, COPY source code sau.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi chạy root: (1) Code Python có lỗ hổng RCE — ví dụ deserialize pickle từ input người dùng không kiểm tra. (2) Kẻ tấn công gửi payload khai thác lỗ hổng → có shell bên trong container với quyền **root** (UID 0). (3) Container root mặc định chia sẻ Linux namespace với host — kẻ tấn công có thể mount `/proc/1/root` hoặc khai thác `docker.sock` (nếu bị mount) để thoát ra ngoài container → có quyền root trên **máy host**.`USER appuser` (UID 10001) cắt đứt ở bước 2→3: Dù kẻ tấn công khai thác thành công lỗ hổng, shell thu được chỉ có quyền của `appuser` — không thể mount filesystem host, không thể write vào `/etc`, không thể leo thang đặc quyền lên host. Còn một lớp bảo vệ UID ngăn cản dù code đã bị khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request** trong 2 giây. Cách đạt được: gửi 10 request lúc giây `:59` của phút thứ nhất (đếm 10/10 — đúng hạn mức, không bị chặn), rồi ngay khi đồng hồ qua giây `:00` của phút tiếp theo (bộ đếm reset về 0), gửi thêm 10 request nữa. Tổng 20 request trong khoảng 2 giây nhưng bộ đếm theo phút đồng hồ coi đây là "10 request phút A + 10 request phút B" — đều hợp lệ. Sliding window 60 giây của `rate_limiter.py` dùng Redis ZSET (`zremrangebyscore` + `zcard`) để luôn nhìn vào 60 giây **gần nhất**, không bao giờ có lỗ hổng reset đầu phút này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm **số lượng request** trong cửa sổ thời gian (10 req/60s trong `rate_limiter.py`), không quan tâm mỗi request tốn bao nhiêu tiền. Cost guard trong `cost_guard.py` đếm **tổng số tiền** đã tiêu trong tháng (`incrbyfloat` trên Redis), không quan tâm request đến nhanh hay chậm.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối → `/health` (đã gộp, có gọi `store.ping()`) bắt đầu trả 503 trên cả 3 container. (2) Load balancer / orchestrator nhận 503 từ liveness probe → đánh dấu cả 3 container là **unhealthy**. (3) Orchestrator lần lượt **kill và restart** cả 3 container. (4) Các container mới khởi động xong nhưng Redis vẫn chưa về → `/health` vẫn 503 → lại bị kill. (5) Vòng lặp crash-loop tiếp diễn suốt 30 giây Redis offline — service hoàn toàn ngừng phục vụ dù bản thân app Python vẫn còn khỏe.Tách `/health` và `/ready` như trong `main.py` giải quyết vấn đề: `/health` (liveness) chỉ check `lifecycle.shutting_down`, không đụng Redis → container không bị restart. `/ready` (readiness) check `store.ping()` → load balancer tạm thời ngừng gửi traffic vào, process vẫn chạy và tự phục hồi khi Redis về.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis (stateless đúng nghĩa): `history_length` tăng đều qua mỗi lần gọi (0, 1, 2, 3...) bất kể request rơi vào container nào trong 3 instance — vì mọi instance đều đọc/ghi cùng một Redis List theo key `history:{user_id}` trong `store.py`.Nếu lưu trong dict Python: Mỗi container có dict riêng trong RAM của nó. Load balancer phân phối request round-robin → câu hỏi 1 vào container A (dict A: length=1), câu hỏi 2 vào container B (dict B chưa có gì: length=0), câu hỏi 3 vào container C (length=0), câu hỏi 4 lại vào container A (length=1 chứ không phải 3). `history_length` sẽ nhảy loạn theo kiểu `0, 1, 0, 0, 1, 1, 0...` — agent "mất trí nhớ" ngẫu nhiên tùy may rủi request rơi vào instance nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS\_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp phải: Health check timeout sau khi deploy lên Railway. Dashboard hiển thị service "Deploying" rồi chuyển "Failed" sau 2 phút, không có response từ `/health`.Thông báo lỗi trong Railway logs:Tìm ra nguyên nhân: Vào tab "Variables" của Railway, phát hiện Railway tự gán `PORT=3000` nhưng CMD trong Dockerfile cũ bind cứng port 8000 (`uvicorn app.main:app --port 8000`) → Railway gọi health check trên port 3000 nhưng app lại lắng nghe trên 8000 → connection refused.Sửa ra sao: Cập nhật CMD trong Dockerfile thành `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để đọc biến `PORT` do Railway inject, với fallback 8000 khi chạy local. Redeploy — health check pass ngay lần đầu, output `/health` trả `{"status":"ok","service":"day12-agent","version":"1.0.0"}` như trong `DEPLOYMENT.md`.
