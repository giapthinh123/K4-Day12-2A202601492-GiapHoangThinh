# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Giáp Hoàng Thịnh  Mã học viên: 2A202601492

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> *khi deploy ứng dụng lên sản phẩm mà quên cài đặt biến `api_token`, app sẽ dừng ngay lúc khởi động và báo lỗi (fail fast). việc này ngăn chặn việc app tự dùng token mặc định `changeme` làm rò rỉ api hoặc bị kẻ xấu lợi dụng gọi dịch vụ miễn phí.*

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> *json
{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:44:00.123456+00:00", "client_id": "user123", "prompt_tokens": 15, "completion_tokens": 42, "usd_cost": 0.000114}*

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
| 1 stage (bản đầu) | 1.73 gb |
| Multi-stage | 264 mb |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> *1. dùng base image `slim` nhẹ hơn image python đầy đủ.*
>
> *2. loại bỏ trình biên dịch, thư viện build rác và cache pip khỏi image runtime ở stage cuối.*

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *khi sửa 1 ký tự trong `app/main.py`: các layer `from`, `workdir`, `copy requirements.txt` và `run pip install` được dùng lại từ cache. chỉ có layer `copy app/` và các bước sau đó phải chạy lại.* 
> *nếu đặt `copy . .` lên trước `run pip install`: mỗi lần đổi code sẽ làm mất cache của `pip install`, khiến docker phải tải và cài lại toàn bộ thư viện từ đầu rất tốn thời gian.*

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> *1. chuỗi khai thác khi chạy bằng root: lỗ hổng code python -> kẻ tấn công chiếm quyền process python -> do process chạy bằng root (uid 0) nên có quyền root trong container -> lợi dụng lỗ hổng container escape để chiếm quyền root trên máy host.*
>
> *2. lệnh `user appuser` cắt đứt chuỗi ở đâu: cắt đứt ngay khi chiếm process python, vì process chỉ chạy với quyền user thường (non-root) nên kẻ tấn công không thể ghi vào hệ thống hay thoát ra máy host.*

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> *1. 401 phải kèm `www-authenticate: bearer` vì đây là chuẩn http (rfc 6750) để báo cho client biết phương thức xác thực cần dùng.*
>
> *2. trả cùng một thông báo lỗi để tránh rò rỉ thông tin (information leakage), không cho kẻ tấn công dò biết họ đã làm đúng hay sai ở bước nào.*

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> *1. gửi được 10 request trước khi bị 429 vì lượng token tối đa trong xô bị giới hạn bởi `capacity = 10`.*
>
> *2. nếu bỏ `min(capacity, ...)` thì gửi được 110 request (10 token có sẵn + 100 token nạp dồn trong 10 phút). điều này làm xô tích lũy vô hạn, khiến client gửi đợt request bùng nổ làm quá tải hệ thống.*

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> *1. hạn mức 30$/tháng: sự cố lúc 2h sáng có thể đốt hết 30$ ngay trong vài giờ và phải chờ đến tháng sau mới tự phục hồi.* 
>
> *2. hạn mức 1$/ngày: thiệt hại tối đa chỉ 1$ trong ngày xảy ra sự cố và hệ thống tự động mở lại vào 00:00 utc ngày hôm sau. hạn mức theo ngày giúp khoanh vùng thiệt hại nhỏ hơn và tự khôi phục nhanh hơn.*

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1.thứ tự sự kiện khi redis mất kết nối 30 giây:
>
> 2.orchestrator tưởng container bị sập nên kill và restart cả 3 container.
>
> 3.container mới boot xong lại check redis thấy hỏng -> tiếp tục bị restart liên tục (crashloopbackoff).
>
> 4.các request không dùng redis cũng bị gián đoạn do container bị kill liên tục.
>
> 5.khi redis hoạt động lại, hệ thống mất thêm thời gian khởi động lại container mới phục hồi hoàn toàn.* 

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

>1.lỗi gặp phải: app không nhận đúng biến môi trường `$port` do platform tự gán dẫn đến health check timeout.
>
>2.thông báo lỗi: `health check failed: timed out connecting to http://0.0.0.0:8000/healthz`.
>
>3.cách tìm nguyên nhân: xem log uvicorn trên dashboard thấy app chỉ chạy cổng 8000 trong khi platform yêu cầu cổng động trong `$port`.
>
>4.cách sửa: cập nhật cmd trong dockerfile thành `exec uvicorn app.main:app --host 0.0.0.0 --port ${port:-8000}`.*