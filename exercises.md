# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tuấn Đức  Mã học viên: 2A202601380

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên production, nếu kỹ sư quên khai báo biến `API_TOKEN` trên Cloud Dashboard, cơ chế Fail Fast khiến app dừng lại ngay lập tức và báo lỗi trong deployment log. Nhờ đó ta phát hiện và sửa lỗi ngay lật tức trước khi app nhận traffic. Ngược lại, nếu để giá trị mặc định `"changeme"`, app vẫn khởi động thành công nhưng kẻ tấn công có thể thử token mặc định `"changeme"` để gọi API hoàn toàn miễn phí, gây rò rỉ dữ liệu và làm nảy sinh chi phí LLM rất lớn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON thu được:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T09:19:01.677393+00:00", "client_id": "sv01", "usd_cost": 0.0001}`

Hai việc làm được với log JSON:
1. Dễ dàng truy vấn và lọc dữ liệu bằng phần mềm gom log (vd: lọc các sự kiện có `severity == "ERROR"` hoặc thống kê số lượt gọi theo từng `client_id`).
2. Tích hợp tự động vào các hệ thống giám sát trên Cloud (GCP Logging, Datadog, Datadog) để vẽ biểu đồ chi phí `usd_cost` và đặt cảnh báo tự động khi chi phí tăng đột biến.

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
| 1 stage (bản đầu) | ~1020 MB |
| Multi-stage | ~310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~710MB) là toàn bộ bộ công cụ biên dịch (compilers, build-essential, g++, python-dev headers, wheel cache) chỉ cần thiết ở quá trình cài đặt thư viện ở stage builder. Multi-stage build loại bỏ hoàn toàn các công cụ này khỏi image runtime cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile tối ưu: layer `COPY requirements.txt` và `RUN pip install` được dùng lại hoàn toàn từ Docker cache, chỉ có layer `COPY . .` và các lệnh sau đó mới phải chạy lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`, mỗi khi sửa 1 dòng code, layer `COPY . .` bị thay đổi làm vô hiệu hóa cache của tất cả các layer phía sau, buộc Docker phải tải và cài lại toàn bộ thư viện qua `pip install` gây tốn nhiều thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Kẻ tấn công khai thác lỗ hổng RCE trong ứng dụng Python $\rightarrow$ Chiếm được quyền thực thi lệnh bên trong container dưới UID 0 (root) $\rightarrow$ Kẻ tấn công khai thác lỗ hổng văng khỏi container (container escape / socket mount) để tương tác với Kernel của máy host $\rightarrow$ Do process chạy UID 0 nên kẻ tấn công có ngay quyền root trên máy host.
Lệnh `USER appuser` chuyển process sang UID 10001 (user không có đặc quyền), cắt đứt chuỗi tấn công ngay từ bước đầu: kẻ tấn công chỉ có quyền hạn vô cùng hạn chế trong container và không thể thực hiện các thao tác root trên host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- Header `WWW-Authenticate: Bearer` là bắt buộc theo chuẩn HTTP RFC 6750 để thông báo cho HTTP client biết hình thức xác thực chuẩn mà API yêu cầu.
- Trả cùng một thông báo lỗi 401 đồng nhất ngăn kẻ tấn công thu thập thông tin dò dẫm (tránh việc tiết lộ "token đúng nhưng sai scheme" hay "token không tồn tại").

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- Client gửi được tối đa **10 request** liên tiếp trước khi bị 429 (do số token tích lũy bị giới hạn bởi `capacity=10`).
- Nếu bỏ `min(capacity, ...)`, số token sau 10 phút sẽ nạp dồn thành `10 token ban đầu + (10 phút * 10 token/phút) = 110 token`. Client sẽ gửi được **110 request** liên tiếp trong 1 giây, vô hiệu hóa tác dụng chống bùng nổ traffic của rate limiter.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- Với hạn mức $30/tháng: Sự cố lúc 2h sáng có thể làm đốt sạch toàn bộ $30 ngân sách của cả tháng chỉ trong vài giờ. Thiệt hại tối đa là $30 và service phải chờ đến đầu tháng sau mới khôi phục.
- Với hạn mức $1/ngày: Thiệt hại tối đa bị khoanh vùng ở $1.0 cho ngày hôm đó. Đến 00:00 UTC ngày tiếp theo, key ngân sách tự reset và service khôi phục bình thường mà không cần ai can thiệp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối 30s $\rightarrow$ Endpoint gộp trả về lỗi 503.
2. Orchestrator hiểu lầm container bị chết nên gửi SIGTERM/SIGKILL và restart cả 3 container cùng lúc.
3. Trong 30s Redis sập, 3 container rơi vào vòng lặp liên tục restart và không thể khởi động xong.
4. Khi Redis hoạt động trở lại, các container vẫn đang khởi động dở dang, biến sự cố nhỏ của Redis thành sự cố ngưng trệ toàn bộ hệ thống (Cascading Failure).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: `Attempt #1 failed with service unavailable. Healthcheck failed` trên Railway dashboard.
- Nguyên nhân: Lệnh `startCommand` trong `railway.toml` ban đầu truyền thẳng chuỗi `$PORT` mà không chạy qua shell `sh -c`, khiến Uvicorn không nhận dạng được cổng động do Railway gán.
- Cách sửa: Sửa `startCommand` trong `railway.toml` thành `sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'` và cập nhật HEALTHCHECK đọc biến môi trường `PORT`.
