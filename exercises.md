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

Khi deploy lên production, nếu người vận hành quên khai báo `API_TOKEN` trên dashboard cloud, cơ chế fail fast làm service dừng ngay lúc khởi động và ghi rõ lỗi vào deployment log. Nhờ vậy tôi phát hiện, bổ sung secret và deploy lại trước khi service nhận traffic. Nếu dùng mặc định `"changeme"`, service vẫn chạy bình thường nhưng token đó có thể bị đoán dễ dàng; người lạ có thể gọi `/chat`, tiêu ngân sách LLM và có thể truy cập dữ liệu mà endpoint xử lý.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON thu được:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T09:19:01.677393+00:00", "client_id": "sv01", "usd_cost": 0.0001}`

Hai việc làm được với log JSON:
1. Có thể lọc các sự kiện lỗi theo `severity == "ERROR"` hoặc thống kê số lượt gọi theo từng `client_id` trong công cụ gom log.
2. Có thể vẽ biểu đồ tổng `usd_cost` theo thời gian và đặt cảnh báo khi chi phí hay tỷ lệ lỗi tăng đột biến trên các nền tảng như Google Cloud Logging hoặc Datadog.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Lệnh `docker images` cho thấy bản multi-stage nhỏ hơn khoảng 1.46 GB. Bản một-stage mang theo base image Python đầy đủ cùng các công cụ và artefact chỉ phục vụ lúc cài dependency; ngoài ra `COPY . .` trước `pip install` làm layer dependency kém hiệu quả hơn. Multi-stage chỉ copy dependency đã cài từ builder và source cần chạy sang runtime `python:3.11-slim`, nên không mang compiler, cache build hay môi trường build vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, khi chỉ sửa `app/main.py`, các layer tải base image, `COPY requirements.txt`, cài dependency ở builder và copy dependency sang runtime được lấy từ cache. Docker chỉ cần chạy lại `COPY . .` ở runtime và tạo lại image cuối.

Nếu đặt `COPY . .` trước `RUN pip install`, thay đổi một ký tự trong source sẽ làm layer copy thay đổi và vô hiệu hóa toàn bộ cache phía sau. Docker sẽ phải tải/cài lại dependencies mỗi lần build thay vì chỉ copy code mới.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện có thể là: kẻ tấn công khai thác RCE trong ứng dụng Python, thực thi lệnh trong container dưới UID 0, rồi lợi dụng cấu hình nguy hiểm (ví dụ mount Docker socket) hoặc một lỗ hổng container escape để leo thang sang host. Chạy app bằng root làm tác động ban đầu lớn hơn vì mã độc đã có mọi quyền quản trị trong container.

`USER appuser` chuyển process sang UID 10001 trước khi chạy Uvicorn. Nếu xảy ra RCE, kẻ tấn công chỉ bắt đầu với quyền user thường: không thể ghi vào các vị trí chỉ root được phép hay cài/sửa cấu hình hệ thống trong container. Lệnh này không thay thế việc vá lỗ hổng container escape, nhưng giảm đáng kể phạm vi thiệt hại và thêm một lớp ngăn leo thang đặc quyền.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- Header `WWW-Authenticate: Bearer` là yêu cầu của response 401 theo RFC 6750; nó cho HTTP client biết API yêu cầu cơ chế Bearer token.
- Trả cùng một thông báo 401 cho mọi trường hợp tránh trở thành oracle cho kẻ tấn công. Nếu API phân biệt "đúng token nhưng sai scheme" với "token sai", người dò token sẽ có thêm tín hiệu để thu hẹp không gian đoán.

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
- Với hạn mức $1/ngày: Thiệt hại tối đa bị khoanh vùng ở $1.0 cho ngày hôm đó. Đến 00:00 UTC ngày tiếp theo, service dùng key chi tiêu của ngày mới nên tự hoạt động lại mà không cần can thiệp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối trong 30 giây, endpoint gộp trả 503 cho cả ba instance.
2. Orchestrator coi từng instance là không còn live và restart chúng; trong khi đó load balancer cũng không còn instance healthy để gửi request.
3. Trong thời gian Redis vẫn lỗi, các instance mới có thể tiếp tục fail health check và bị restart tiếp.
4. Khi Redis hồi phục, cụm vẫn cần thời gian khởi động và đăng ký healthy lại. Một lỗi dependency ngắn đã bị khuếch đại thành gián đoạn toàn bộ service. Tách `/healthz` và `/readyz` tránh chuỗi sự kiện này: liveness vẫn 200, còn load balancer chỉ tạm ngừng gửi traffic qua readiness.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: `Attempt #1 failed with service unavailable. Healthcheck failed` trên Railway dashboard.
- Cách tìm nguyên nhân: tôi mở deployment logs, kiểm tra healthcheck path và đối chiếu cổng Railway cấp với lệnh chạy Uvicorn. Service chưa bind đúng cổng động nên Railway không gọi được `/healthz`.
- Cách sửa: tôi đổi `startCommand` trong `railway.toml` thành `sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'` và dùng healthcheck gọi `/healthz`. Sau lần deploy kế tiếp, dashboard báo service online và endpoint trả 200.
