# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Duy Đức            Mã học viên: 2A202602019

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định là "changeme", khi deploy một phiên bản lên production mà quên set biến môi trường, app vẫn khởi động bình thường. Kẻ tấn công có thể dễ dàng truy cập API với token "changeme" và đánh cắp dữ liệu hoặc đốt sạch tài nguyên. Việc "chết sớm" khiến quá trình deploy thất bại ngay lập tức, báo động cho lập trình viên sửa lỗi trước khi user bị ảnh hưởng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> {"level": "info", "timestamp": "2026-08-10T07:46:24.000000Z", "message": "chat_completed", "client_id": "sv-test", "prompt_tokens": 12, "completion_tokens": 40, "usd_cost": 0.00052}
> Hai việc làm được: 
> 1. Dùng các hệ thống như Elasticsearch để filter, query dễ dàng (ví dụ: tìm tất cả log có `usd_cost > 0.01`).
> 2. Có thể phân tích thống kê tự động hoặc vẽ biểu đồ, ví dụ như tính tổng lượng token sử dụng của một client trong ngày mà không cần dùng regex để parse chuỗi phức tạp.

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
| 1 stage (bản đầu) | ~1.03 GB |
| Multi-stage | ~145 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch là do bản 1 stage chứa toàn bộ công cụ build, compiler (như gcc), mã nguồn, pip cache... Còn bản multi-stage chỉ copy thư mục `site-packages` (chứa các thư viện đã build xong) sang một môi trường siêu nhẹ (python:3.11-slim) mà không lưu lại các công cụ dư thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Các layer cài thư viện (`COPY requirements.txt .`, `RUN pip install`) được tái sử dụng (cached). Chỉ có layer `COPY . .` và các layer sau đó bị chạy lại.
> Nếu đặt `COPY . .` lên trước, mỗi khi sửa code, Docker sẽ không dùng lại cache ở bước đó nữa, kéo theo lệnh `RUN pip install` phải chạy lại từ đầu, làm quá trình build cực kỳ chậm.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng có lỗ hổng thực thi mã từ xa (RCE), hacker có thể chạy mã trên container với quyền root. Từ đó, nếu cấu hình lỏng lẻo, hacker có thể leo thang đặc quyền để tương tác với Docker socket hoặc file hệ thống của máy host, chiếm luôn máy host. 
> Lệnh `USER nonroot` cắt đứt chuỗi ở ngay bước đầu: hacker khi RCE chỉ có quyền hạn chế của `nonroot`, không thể cài đặt thêm công cụ hay đọc file nhạy cảm trong container.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> 401 kèm header `WWW-Authenticate` là tiêu chuẩn của HTTP, báo cho client (như trình duyệt hoặc tool) biết cần xác thực bằng phương thức Bearer.
> Ta trả chung thông báo lỗi để không tiết lộ manh mối cho kẻ tấn công (ngăn chặn dò tìm lỗi - enumeration attack). Nếu nói rõ "sai scheme", attacker biết token đã gửi có thể là đúng. Nói chung chung làm attacker phải tự đoán tất cả.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Nó gửi được tối đa 10 request trước khi bị 429 (bằng capacity). 
> Nếu bỏ `min(capacity, ...)`, token sẽ cộng dồn thành: 10 + (10 * 10) = 110 token. Do đó nó sẽ gửi được 110 request liên tiếp. Việc này phá hỏng mục đích giới hạn tải đột biến của bucket.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với $30/tháng: Khách hàng mất hết $30 trong vài tiếng, và API của họ bị khóa hoàn toàn đến cuối tháng (không tự phục hồi nếu không nạp thêm).
> Với $1/ngày: Khách hàng chỉ mất tối đa $1 cho ngày đó. Thiệt hại được giới hạn tối thiểu. API sẽ bị khóa hôm đó nhưng sẽ tự động hồi phục ngay khi sang ngày mới (budget được cấp lại $1).

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
> 1. Redis chết. Cả 3 container khi check healthz đều trả về lỗi.
> 2. Kubernetes/Orchestrator hiểu lầm rằng bản thân 3 container bị hỏng, nên tiến hành restart/kill các container này.
> 3. Các container khởi động lại nhưng Redis vẫn chưa có, vòng lặp restart vô ích diễn ra làm sập toàn bộ hệ thống (downtime). Thay vì chỉ tạm ngắt traffic (readyz), ta lại giết bỏ cả service đang hoạt động.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp phải: Health check timeout trên nền tảng cloud.
> Nguyên nhân tìm được (qua phần log deploy): Ứng dụng luôn listen ở port 8000 cứng thay vì đọc biến môi trường `$PORT` do cloud provider cung cấp, khiến load balancer không thể ping tới service.
> Khắc phục: Cập nhật file docker-compose hoặc lệnh chạy thành `uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}` để hỗ trợ port động.
