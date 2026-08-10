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

Tình huống cụ thể: Khi deploy ứng dụng lên môi trường Production/Staging, dev hoặc pipeline CI/CD quên khai báo biến môi trường `API_TOKEN`.

- **Nếu có giá trị mặc định là `"changeme"`:** App vẫn khởi động bình thường, health check xanh (Pass 200 OK). Hệ thống đi vào hoạt động mà không ai nhận ra sơ suất. Kẻ tấn công hoặc bot tự động trên internet có thể thử token mặc định phổ biến này để gọi API thành công, dẫn đến lãng phí ngân sách LLM hoặc rò rỉ dữ liệu.
- **Việc "Fail Fast" (chết sớm):** Khi ứng dụng khởi động, Pydantic Settings phát hiện thiếu `api_token` và quăng `ValidationError`, khiến app crash ngay lập tức. Deployment pipeline báo lỗi FAILED, load balancer không đẩy traffic vào, và phát cảnh báo khẩn cấp tới DevOps để bổ sung secret đúng trước khi ứng dụng phục vụ người dùng thật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:44:00.123456+00:00", "client_id": "user123", "prompt_tokens": 15, "completion_tokens": 42, "usd_cost": 0.000114}
```

Hai việc làm được với log JSON mà `print("đã trả lời xong")` không làm được:
1. **Lọc, tìm kiếm & Cảnh báo tự động trên hệ thống tập trung (Log Aggregator / Cloud Logging):** Các hệ thống như Datadog, Google Cloud Logging, ELK tự động parse các trường JSON (`severity`, `client_id`, `usd_cost`). Ta có thể dễ dàng truy vấn `severity = "ERROR"`, `client_id = "user123"` hoặc cấu hình Alert khi `usd_cost` tăng bất thường. Chuỗi `print` thuần túy không chứa metadata nên rất khó lọc hoặc phải dùng Regex phức tạp.
2. **Thống kê & Phân tích chi số (Metrics Analytics):** Nhờ có các trường định lượng (`prompt_tokens`, `completion_tokens`, `usd_cost`, `ts`, `client_id`), ta có thể viết câu lệnh SQL/KQL/PromQL để tính tổng chi phí LLM theo từng client trong ngày hoặc thống kê số token trung bình per request. Điều này bất khả thi với câu lệnh `print` thông thường.

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
| 1 stage (bản đầu) | 1.73 GB (ảo: 1.73GB / nén: 446MB) |
| Multi-stage | 264 MB (ảo: 264MB / nén: 62.2MB) |

Giải thích phần dung lượng chênh lệch:
1. **Base Image:** Bản 1-stage dùng OS image đầy đủ chứa nhiều gói phần mềm không cần thiết cho runtime (như `gcc`, `g++`, `make`, header files `python3-dev`, git,...). Bản Multi-stage dùng base `slim` loại bỏ hầu hết các package này.
2. **Build tools & Compilers:** Trình biên dịch C/C++ và các dependency tạm phục vụ biên dịch wheel chỉ nằm ở stage `builder` và không bị đưa vào image runtime cuối cùng.
3. **Pip Cache & Tệp rác:** Stage runtime chỉ copy thư viện đã cài đặt (`/install`), loại bỏ toàn bộ `~/.cache/pip` và các tệp tạm phát sinh trong quá trình build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Khi sửa 1 ký tự trong `app/main.py` và build lại:**
  - *Layer được dùng lại từ cache:* `FROM python:3.12-slim AS builder`, `WORKDIR`, `COPY requirements.txt .`, và layer `RUN pip install ...`.
  - *Layer phải chạy lại:* `COPY app/ app/` (do checksum của `app/main.py` thay đổi) và tất cả các instruction phía sau nó (`COPY utils/`, `USER`, `EXPOSE`, `HEALTHCHECK`, `CMD`).
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Mỗi lần sửa code (`app/main.py`), checksum của thư mục thay đổi làm invalidated cache của layer `COPY . .`.
  - Layer `RUN pip install` nằm phía sau bị mất cache và phải **tải + cài đặt lại toàn bộ thư viện từ đầu**, làm thời gian build lại tăng từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện khai thác khi container chạy bằng root:**
  1. Kẻ tấn công khai thác lỗ hổng trong code Python (ví dụ RCE hoặc Command Injection).
  2. Kẻ tấn công thực thi mã độc và chiếm quyền kiểm soát process Python bên trong container.
  3. Vì container chạy bằng root (UID 0), process Python có quyền root trong container.
  4. Kẻ tấn công lợi dụng các lỗ hổng Container Escape (như cgroup release_agent, mounting `/var/run/docker.sock`, hoặc kernel exploits) để thoát khỏi ranh giới cách ly của container.
  5. Do UID 0 trong container tương ứng với UID 0 (root) trên máy Host, kẻ tấn công chiếm toàn quyền kiểm soát máy Host.
- **Lệnh `USER appuser` cắt đứt chuỗi sự kiện ở đâu:**
  - Lệnh `USER` chuyển process sang chạy dưới quyền user thường (non-root, UID 1000).
  - Chuỗi sự kiện bị **cắt đứt ngay từ bước (3)**: Kẻ tấn công chỉ có quyền hạn hạn chế của `appuser` bên trong container, không thể ghi vào hệ thống container, không thể sudo hay khai thác các privilege escalation để escape ra máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- **Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`:**
  - Đây là quy định bắt buộc theo chuẩn **HTTP/1.1 (RFC 7235 & RFC 6750)** đối với phản hồi `401 Unauthorized`.
  - Header này báo cho client (browser, Postman, API SDK) biết chính xác cơ chế xác thực mà Server yêu cầu là loại nào (ở đây là phương thức `Bearer` token), giúp client biết cách tự động dựng lại request đúng định dạng.
- **Vì sao trả cùng một thông báo lỗi cho cả ba trường hợp:**
  - Tuân thủ nguyên tắc bảo mật **Information Leakage Prevention** (chống rò rỉ thông tin).
  - Nếu trả về lỗi chi tiết (như *"Thiếu header"*, *"Sai scheme"* hay *"Sai token"*), kẻ tấn công có thể lợi dụng để thực hiện cuộc tấn công thăm dò (enumeration/brute-force) để biết từng bước thử nghiệm của mình đã đúng đến đâu. Việc dùng chung một thông báo duy nhất giúp vô hiệu hóa kỹ thuật thăm dò này.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- **Số request gửi được trước khi bị 429:** **10 request**.
  - *Giải thích:* Sau 10 phút im lặng, lượng token tích lũy theo lý thuyết là $10 \text{ phút} \times 10 \text{ tokens/phút} = 100 \text{ tokens}$. Tuy nhiên hàm `available()` có giới hạn `min(capacity, tokens)` với `capacity = 10`, nên số token trong xô tối đa chỉ là 10. Gửi 10 request liên tiếp sẽ dùng hết 10 token, request thứ 11 sẽ nhận 429.
- **Nếu bỏ `min(capacity, ...)` trong `available()`:**
  - Con số đó thành **110 request** (10 token ban đầu + 100 token nạp dồn trong 10 phút).
  - *Tại sao:* Xô không có trần giới hạn sẽ tích tụ token vô hạn trong thời gian im lặng. Client có thể xả đợt bùng nổ (burst) cực lớn 110 request liên tiếp, làm mất tác dụng khống chế tải của Token Bucket và có thể làm sập hệ thống downstream.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- **Với hạn mức $30/tháng:**
  - *Thiệt hại tối đa:* Sự cố lúc 2h sáng có thể đốt sạch toàn bộ **$30** chỉ trong 1 ngày (hoặc vài giờ).
  - *Tự hồi phục:* Service phải chờ đến **đầu tháng sau** (có thể mất đến 20-30 ngày) mới tự reset hạn mức, hoặc cần admin can thiệp thủ công.
- **Với hạn mức $1/ngày:**
  - *Thiệt hại tối đa:* Khoanh vùng thiệt hại tối đa chỉ đúng **$1.0** trong ngày xảy ra sự cố. Khi chạm mốc $1.0, các request tiếp theo lập tức bị chặn với 402 Payment Required.
  - *Tự hồi phục:* Service **tự động hồi phục vào 00:00 UTC ngày hôm sau** khi key Redis ngày mới được tạo (`spend:client_id:YYYY-MM-DD`).
- *Kết luận:* Ngân sách theo ngày giúp khoanh vùng rủi ro (blast radius) nhỏ hơn 30 lần và có khả năng self-healing tự động hàng ngày mà không cần con người can thiệp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự các sự kiện xảy ra khi Redis mất kết nối 30s:
1. **Redis mất kết nối (0s):** Cả 3 container ứng dụng vẫn đang chạy nhưng không kết nối được tới Redis.
2. **Health check gộp bị thất bại:** Orchestrator (Docker/Kubernetes/Cloud) gọi endpoint check duy nhất. Vì endpoint này check Redis, nó trả về 503 Service Unavailable.
3. **Orchestrator đánh dấu Dead và restart container:** Orchestrator hiểu nhầm rằng process ứng dụng bị sập/hung, tiến hành `kill` và khởi động lại cả 3 container.
4. **Boot storm & Thất bại dây chuyền:** 3 container mới khởi động lại, tiêu tốn CPU/RAM để boot. Boot xong check Redis vẫn sập -> tiếp tục bị kill và restart (CrashLoopBackOff).
5. **Đứt gãy request:** Mọi request hợp lệ đang xử lý hoặc không cần Redis đều bị đứt gãy đột ngột do container bị kill liên tục.
6. **Khi Redis hồi phục (sau 30s):** Các container vẫn tốn thêm thời gian khởi động lại từ đầu, khiến thời gian gián đoạn dịch vụ kéo dài hơn rất nhiều so với 30s sập Redis.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** App không đọc biến môi trường `$PORT` do Cloud Platform (Render/Railway) tự động cấp phát, dẫn đến Health Check Timeout / Deployment Failure.
- **Thông báo lỗi:**
  `Health check failed: timed out after 30s connecting to http://0.0.0.0:8000/healthz`
  `Exited with status 1 - Container failed to start`
- **Cách tìm nguyên nhân:** Kiểm tra Runtime Logs trên dashboard của Cloud thấy Uvicorn báo listening tại port 8000 (`http://0.0.0.0:8000`), trong khi Cloud gán biến `PORT=10000` (hoặc cổng ngẫu nhiên) và thực hiện healthcheck tới port ngẫu nhiên đó.
- **Cách sửa:** Cập nhật lệnh khởi chạy trong `Dockerfile` để Uvicorn nhận biến `PORT` động:
  `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
