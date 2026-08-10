# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các câu hỏi mẫu bằng câu trả lời.

> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Đăng Nguyên  Mã học viên: 2A202601798

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên Cloud (Railway/Render), nếu ta đặt giá trị mặc định cho `api_token` là `"changeme"`, khi developer quên thiết lập biến môi trường `API_TOKEN` trên Dashboard, ứng dụng vẫn sẽ khởi động thành công. Tuy nhiên, kẻ xấu có thể dò thử API với token mặc định `"changeme"` và gọi miễn phí dịch vụ LLM của chúng ta, làm thất thoát ngân sách cho đến khi nhận được hóa đơn điện toán. Ngược lại, việc không đặt mặc định giúp ứng dụng áp dụng nguyên lý Fail Fast — ném lỗi `ValidationError` và dừng khởi động lập tức khi thiếu secret, giúp ta phát hiện và khắc phục ngay trước khi ứng dụng nhận bất kỳ request thực tế nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON thu được:
```json
{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:16:40.123456+00:00", "client_id": "sv-test", "prompt_tokens": 15, "completion_tokens": 42, "usd_cost": 0.00015}
```

Hai việc làm được với log JSON mà `print()` thông thường không làm được:
1. **Lọc và tìm kiếm tự động trên Cloud Logging**: Các hệ thống thu thập log (GCP Cloud Logging, Datadog, Grafana Loki) có thể parse các trường JSON để lọc theo mức độ nghiêm trọng `severity == "ERROR"` hoặc tìm kiếm chính xác mọi log của một `client_id` nhất định.
2. **Truy vấn thống kê và Cài đặt Cảnh báo (Alerts)**: Có thể tính tổng chi phí `usd_cost` hoặc lượng token tiêu thụ theo thời gian thực để vẽ biểu đồ giám sát và tự động kích hoạt cảnh báo khi chi phí tăng đột biến.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.8 GB |
| Multi-stage | ~250 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~1.55 GB) bao gồm hệ điều hành Debian đầy đủ, bộ công cụ biên dịch (GCC, Make, python-dev headers), trình quản lý gói apt cache, pip build cache và các công cụ phát triển không cần thiết cho môi trường runtime. Ở phiên bản Multi-stage, ta sử dụng base image tinh gọn `python:3.11-slim` và chỉ copy các thư viện đã được cài đặt từ stage `builder` sang stage runtime `runner`, loại bỏ hoàn bộ toolchain biên dịch thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa 1 ký tự trong `app/main.py`:
- Các layer `COPY requirements.txt .` và `RUN pip install` ở stage `builder` được sử dụng lại hoàn toàn từ Docker build cache.
- Layer `COPY app /app/app` và các bước sau đó bị cache miss và phải chạy lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần chỉnh sửa dù chỉ 1 dòng code, layer `COPY . .` sẽ làm dội cache (cache miss), buộc Docker phải tải và cài đặt lại toàn bộ các thư viện Python từ đầu (tốn nhiều phút thay vì chỉ mất 2 giây).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công khai thác lỗ hổng trong ứng dụng Python (ví dụ: Remote Code Execution - RCE qua `eval()` hoặc Command Injection).
2. Khi container chạy với quyền root (`UID 0`), tiến trình Python bị chiếm quyền kiểm soát sẽ sở hữu quyền root tối cao bên trong container.
3. Nếu kẻ tấn công khai thác tiếp một lỗ hổng thoát khỏi container (container breakout), họ sẽ chiếm được ngay quyền root trực tiếp trên máy host.
Lệnh `USER appuser` cắt đứt chuỗi tấn công ở bước 2: Khi tiến trình Python bị chiếm quyền, kẻ tấn công chỉ có quyền của một người dùng thông thường (`UID 1000`), không có quyền sudo/root, bị giới hạn phạm vi tác động bên trong container và không thể leo thang chiếm quyền trên máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- Header `WWW-Authenticate: Bearer` là bắt buộc theo chuẩn RFC 6750 đối với response HTTP 401 để chỉ dẫn cho client (browser, Postman, curl) biết phương thức xác thực mà server yêu cầu là Bearer Token.
- Trả về cùng một thông báo lỗi cho cả 3 trường hợp là nguyên tắc tối thiểu về an toàn thông tin. Nếu thông báo chi tiết như "token không tồn tại", kẻ tấn công sẽ biết scheme và header đã đúng để tiếp tục thực hiện tấn công dò quét (brute-force). Việc trả cùng một thông báo lỗi ngăn chặn kẻ tấn công thu thập thông tin về cấu trúc và trạng thái của hệ thống.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- Client sẽ gửi được tối đa đúng **10 request** liên tiếp trước khi nhận lỗi 429.
- Nếu bỏ đoạn `min(capacity, ...)`: Con số đó sẽ tăng lên thành **110 request** liên tiếp (10 token ban đầu cộng với 100 token được nạp thêm trong 10 phút im lặng). Điều này làm vô hiệu hóa khả năng kiểm soát lưu lượng burst của thuật toán Token Bucket, cho phép client tích lũy vô hạn token sau thời gian chờ để tấn công làm quá tải hệ thống trong 1 giây.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- Hạn mức $30/tháng: Client có thể tiêu tốn toàn bộ $30 ngân sách chỉ trong vài phút của đợt tấn công từ 2h sáng. Thiệt hại tối đa là **$30**, và dịch vụ của client đó sẽ bị khóa hoàn toàn đến hết tháng (chỉ hồi phục vào ngày đầu tiên của tháng sau).
- Hạn mức $1/ngày: Sự cố từ 2h sáng chỉ gây thiệt hại tối đa **$1**. Ngay khi đạt mốc $1, Cost Guard tự động trả lỗi HTTP 402 Payment Required để chặn đứng các request tiếp theo. Dịch vụ sẽ tự động hồi phục hoàn toàn vào 00:00 UTC ngày hôm sau mà không cần sự can thiệp thủ công.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện khi gộp 2 endpoint và Redis mất kết nối 30 giây:
1. Redis gặp sự cố gián đoạn kết nối trong 30 giây.
2. Cụm Load Balancer gọi health check chung và cả 3 container đều trả về trạng thái thất bại (non-200).
3. Container Orchestrator (Docker/Kubernetes) đánh giá cả 3 container đã bị lỗi (unhealthy) và liên tục gửi tín hiệu `SIGKILL` để khởi động lại toàn bộ cụm 3 container.
4. Hành động khởi động lại đồng loạt tạo ra thảm họa cascading failure (tải CPU/RAM tăng vọt khi ứng dụng boot lại), khiến dịch vụ ngưng hoạt động hoàn toàn (100% downtime) ngay cả với những endpoint không hề phụ thuộc vào Redis.
Việc tách riêng `/healthz` (chỉ kiểm tra tiến trình sống) và `/readyz` (kiểm tra dependency) giúp Load Balancer tạm ngừng gửi traffic tới container khi Redis nghẽn qua `/readyz`, nhưng `/healthz` vẫn giữ container hoạt động chờ Redis tự phục hồi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: `Error: Invalid value for '--port': '$PORT' is not a valid integer.` khi container khởi động trên Railway.
- Nguyên nhân: File cấu hình `railway.toml` chứa dòng `startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"`. Railway thực thi trực tiếp câu lệnh này mà không qua môi trường Shell (`sh -c`), làm cho biến môi trường `$PORT` không được giải mã thành cổng số thực tế mà bị truyền nguyên văn dạng chuỗi `"$PORT"` vào tham số `--port` của Uvicorn.
- Cách khắc phục: Cập nhật dòng `startCommand` trong `railway.toml` thành `startCommand = "sh -c 'exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'"` để Shell giải mã biến môi trường `$PORT` trước khi chạy Uvicorn.

