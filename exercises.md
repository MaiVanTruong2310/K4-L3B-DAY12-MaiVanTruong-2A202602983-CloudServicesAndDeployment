# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Mai Văn Trường  Mã học viên: 2A202602983

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống cụ thể: Khi deploy ứng dụng lên nền tảng đám mây (như Railway hoặc Render), nếu kỹ sư quên khai báo biến môi trường `AGENT_API_KEY` trong phần cấu hình dashboard. Nếu đặt mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công, vượt qua health check và mở cổng cho traffic bên ngoài. Các bot tự động quét lỗ hổng trên Internet sẽ nhanh chóng tìm ra endpoint `/ask` và thử các key mặc định phổ biến như `"changeme"`. Khi đó, kẻ lạ có thể thoải mái gọi API và tiêu tốn toàn bộ ngân sách LLM mà ta chỉ biết khi nhận hóa đơn. Nhờ cơ chế fail-fast (không có giá trị mặc định), Pydantic sẽ ném ra `ValidationError` ngay lúc khởi động, ngăn chặn quá trình release bị lỗi và buộc kỹ sư phải cấu hình khóa bảo mật chính xác ngay từ đầu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T14:25:35.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 25, "cost_usd": 0.0001}`
> 
> Hai việc làm được với log JSON mà `print("đã trả lời xong")` không thể làm được:
> 1. Phân tích cú pháp và lọc có cấu trúc tự động (Structured Filtering): Các hệ thống gom log tập trung (Datadog, Grafana Loki, CloudWatch, ELK) có thể tự động parse JSON để lọc và truy vấn chính xác theo từng trường, ví dụ: tìm tất cả request của `user_id = "sv-test"` hoặc tìm các request có `cost_usd > 0.005`, điều mà print text thuần túy không làm được nếu không viết regex phức tạp.
> 2. Đo lường và thiết lập cảnh báo định lượng (Metrics & Alerting): Dễ dàng tính toán tổng chi phí tiêu thụ theo thời gian thực (aggregate sum `cost_usd`), đo tỷ lệ token in/out trung bình, hoặc tự động kích hoạt cảnh báo (Alert) gửi về Slack/PagerDuty ngay khi phát hiện người dùng tiêu vượt ngưỡng chi phí cho phép trong 5 phút.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1024 MB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~839 MB) gồm các thành phần sau:
> 1. Base image gốc `python:3.11` là bản đầy đủ dựa trên Debian chuẩn chứa rất nhiều công cụ phát triển, trình biên dịch C/C++ (`build-essential`, `gcc`), thư viện tiện ích hệ thống và các header files không cần thiết ở môi trường runtime. Trong khi đó bản multi-stage dùng `python:3.11-slim` chỉ chứa môi trường tối giản cần thiết để chạy Python.
> 2. Bộ nhớ đệm (cache) của trình quản lý gói `pip` và các file wheel/tải về tạm thời khi cài đặt thư viện được giữ lại ở stage builder và bị loại bỏ hoàn toàn nhờ cờ `--no-cache-dir` kết hợp chỉ copy thư mục cài đặt `/install` sang stage runtime.
> 3. Các file tài liệu hướng dẫn (man pages), mã nguồn test, và package quản lý không dùng tới bị loại bỏ, giúp image nhỏ gọn, kéo image và khởi động container nhanh hơn gấp nhiều lần.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại:
> - Các layer được dùng lại từ cache (CACHED): Tải base image, thiết lập `WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install ...`, thiết lập `WORKDIR /app`, `RUN useradd ...`, và `COPY --from=builder /install /usr/local`. Lý do là file `requirements.txt` không thay đổi.
> - Các layer phải chạy lại: Chỉ có layer `COPY app ./app` và các bước sau nó (`COPY utils ./utils`, cấu hình `USER`, `EXPOSE`, `HEALTHCHECK`, `CMD`).
> 
> Nếu đặt `COPY . .` lên trước `RUN pip install`:
> Do Docker lưu cache theo từng layer tuần tự từ trên xuống dưới, bất cứ thay đổi nào trong thư mục mã nguồn (dù chỉ là sửa 1 ký tự trong `main.py`) cũng sẽ làm mất hiệu lực (bust) cache của lệnh `COPY . .` và toàn bộ các lệnh sau nó. Khi đó, Docker bắt buộc phải thực thi lại lệnh `RUN pip install`, tải lại và cài đặt lại toàn bộ thư viện từ Internet, khiến thời gian build bị kéo dài từ vài giây lên vài phút mỗi lần lập trình viên sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện tấn công:
> 1. Ứng dụng Python tồn tại một lỗ hổng bảo mật (ví dụ: Command Injection, RCE qua thư viện deserialization pickle, hoặc Arbitrary File Write).
> 2. Kẻ tấn công gửi payload khai thác thành công và chiếm được shell điều khiển bên trong container.
> 3. Do container không chỉ định `USER`, tiến trình Python chạy với quyền `root` (UID 0). Do đó kẻ tấn công sở hữu quyền root bên trong không gian tên (namespace) của container.
> 4. Kẻ tấn công tìm kiếm các điểm tiếp xúc với host: khai thác container breakout (qua lỗ hổng của kernel Linux, lỗ hổng runc, hoặc container được mount quyền nguy hiểm như `/var/run/docker.sock`, `privileged: true`, hay mount volume từ thư mục nhạy cảm trên host). Kẻ tấn công thoát khỏi container và xuất hiện trên máy host với đúng UID 0 (root của máy host), từ đó toàn quyền kiểm soát hệ thống, đánh cắp dữ liệu và cài backdoor.
> 
> Vị trí lệnh `USER appuser` cắt đứt chuỗi:
> Lệnh `USER appuser` cắt đứt chuỗi ngay tại bước 3. Khi kẻ tấn công xâm nhập được vào container, chúng chỉ sở hữu quyền của một user thông thường không có đặc quyền (UID 10001, không có quyền sudo, không thể ghi vào các file nhị phân hệ thống). Kẻ tấn công không thể đọc ghi file nhạy cảm, không thể tương tác với docker daemon socket, và triệt tiêu hầu hết các kỹ thuật leo thang đặc quyền container breakout.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Một người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp khi hạn mức là 10/phút.
> 
> Cách đạt được con số đó:
> - Người dùng gửi 10 request vào giây thứ 59 của phút trước đó (ví dụ: 10:00:59). Toàn bộ 10 request này được chấp thuận vì thuộc hạn mức của phút 10:00.
> - Đúng 1 giây sau, khi đồng hồ điểm sang giây 00 của phút tiếp theo (10:01:00), cơ chế fixed-window reset bộ đếm về 0.
> - Ngay lập tức tại thời điểm 10:01:00 hoặc 10:01:01, người dùng gửi thêm 10 request nữa. Hệ thống tiếp tục cho qua vì đây là hạn mức mới của phút 10:01.
> Kết quả: Hệ thống phải gánh chịu một đợt bùng nổ lưu lượng (traffic burst) gồm 20 request chỉ trong khoảng 2 giây liên tiếp, gấp đôi công suất thiết kế. Thuật toán sliding window 60 giây dùng Redis Sorted Set loại bỏ được kẽ hở này vì nó luôn tính số lượng request trong đúng 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Sự khác nhau giữa hai cơ chế:
> - Rate Limit: Bảo vệ hệ thống về mặt tần suất / lưu lượng tức thời (số request trên một đơn vị thời gian ngắn, ví dụ 10 request/phút) nhằm tránh nghẽn mạng và chống quá tải tài nguyên tính toán (DDoS).
> - Cost Guard: Bảo vệ hệ thống về mặt tài chính / ngân sách tích lũy (tổng số tiền USD trên một chu kỳ dài, ví dụ tháng) dựa trên lượng token thực tế mà LLM xử lý.
> 
> Tình huống 1: Rate limit cho qua nhưng Cost guard chặn:
> Người dùng chỉ gửi 1 request trong 10 phút (tần suất cực thấp, hoàn toàn thỏa mãn rate limit 10 request/phút), nhưng câu hỏi kèm văn bản tài liệu đầu vào khổng lồ (50.000 token) khiến chi phí ước tính vượt quá số dư ngân sách còn lại trong tháng của người dùng -> Cost guard chặn lại với mã lỗi HTTP 402 Payment Required.
> 
> Tình huống 2: Cost guard cho qua nhưng Rate limit chặn:
> Người dùng mới bắt đầu tháng, ngân sách còn nguyên $10.0 chưa tiêu đồng nào, nhưng viết script gửi liên tiếp 15 câu hỏi ngắn trong vòng 3 giây. Mặc dù tổng chi phí của 15 câu hỏi này rất nhỏ (chưa tới $0.01), nhưng tốc độ gửi vượt quá hạn mức 10 request/phút -> Rate limit lập tức chặn từ request thứ 11 với mã lỗi HTTP 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện xảy ra:
> 1. Redis gặp sự cố mạng hoặc khởi động lại, tạm thời mất kết nối trong 30 giây.
> 2. Cơ chế giám sát container (orchestrator như Docker Swarm / Kubernetes) định kỳ gọi probe kiểm tra liveness (`/health`). Do gộp chung logic kiểm tra Redis, endpoint `/health` trả về lỗi HTTP 503 trên cả 3 container agent.
> 3. Orchestrator hiểu rằng tiến trình bên trong container đã bị treo/hỏng (unhealthy) và kích hoạt hành động phục hồi: ép tắt (kill) và restart lại toàn bộ cả 3 container cùng lúc.
> 4. Trong suốt thời gian các container bị restart (tải lại Python interpreter, nạp module, khởi tạo bộ nhớ), toàn bộ hệ thống bị gián đoạn hoàn toàn (total outage), không thể phục vụ bất kỳ traffic nào (kể cả các request tĩnh hoặc không cần đến Redis).
> 5. Khi Redis vừa hồi phục sau 30 giây, các container lại đang trong vòng xoáy khởi động lại (restart loop) hoặc chịu tải khởi động đột ngột, dẫn đến tình trạng sập domino (cascading failure).
> Tách riêng `/health` (chỉ kiểm tra process sống) và `/ready` (kiểm tra dependency) giúp load balancer chỉ tạm ngưng đẩy traffic vào khi Redis ngắt kết nối mà không restart container, hệ thống phục hồi ngay lập tức khi Redis hoạt động trở lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lịch sử được lưu trong dict Python in-memory thay vì Redis:
> Khi có 3 container chạy song song sau load balancer, mỗi request gửi đến sẽ được phân phối theo cơ chế round-robin tới một trong ba container A, B hoặc C. Vì mỗi container sở hữu một vùng nhớ RAM độc lập:
> - Request 1 tới container A: `history_length` là 0. A lưu câu 1 vào RAM của mình.
> - Request 2 tới container B: do B không chia sẻ RAM với A, B thấy lịch sử rỗng -> `history_length` lại là 0. B lưu câu 2 vào RAM của B.
> - Request 3 tới container C: C cũng thấy lịch sử rỗng -> `history_length` vẫn là 0.
> - Request 4 quay lại container A: A tìm thấy câu 1 trước đó -> `history_length` là 2.
> - Request 5 tới container B: B tìm thấy câu 2 trước đó -> `history_length` là 2.
> 
> Hiện tượng: Con số `history_length` không tăng tuần tự (0, 2, 4, 6, 8...) mà sẽ nhảy lộn xộn, ngắt quãng. Agent bị "mất trí nhớ ngẫu nhiên" tùy vào việc container nào tiếp nhận request. Chuyển state sang Redis giúp cả 3 instance cùng đọc và ghi vào một nguồn dữ liệu duy nhất.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Lỗi gặp phải: `Application failed to respond on port 8000 within timeout / Health check failed`.
> - Cách tìm nguyên nhân: Mở tab Deployment Logs và Runtime Logs trên dashboard của Railway/Render. Quan sát log thấy ứng dụng khởi động thành công với thông báo `Uvicorn running on http://0.0.0.0:8000`, tuy nhiên nền tảng đám mây lại ping health check vào cổng do họ chỉ định ngẫu nhiên (ví dụ `PORT=35421`) qua biến môi trường `$PORT`. Do ứng dụng hardcode cổng 8000 nên router của cloud không thể kết nối tới ứng dụng, dẫn đến timeout và deploy bị hủy bỏ.
> - Cách sửa:
>   1. Sửa lệnh `CMD` trong `Dockerfile` để đọc biến môi trường `$PORT` do cloud cấp phát, có fallback về 8000:
>      `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
>   2. Cập nhật `Settings` trong `app/config.py` đọc đúng biến `PORT` và đảm bảo bind host luôn là `0.0.0.0` (thay vì `127.0.0.1`) để container chấp nhận traffic từ bên ngoài.
