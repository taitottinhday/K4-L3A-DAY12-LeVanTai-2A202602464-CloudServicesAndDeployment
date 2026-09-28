# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng placeholder bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Văn Tài  Mã học viên: 2A202602464

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy lên Render nhưng quên khai báo
> `AGENT_API_KEY`. Nếu khóa bắt buộc, process dừng ngay và log chỉ rõ lỗi cấu
> hình trước khi nhận traffic. Nếu code dùng mặc định `"changeme"`, service
> vẫn báo healthy; người biết khóa mặc định có thể gọi `/ask` và tiêu ngân
> sách của tôi trước khi tôi phát hiện cấu hình sai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thực tế thu được khi gọi service trong Docker Compose:
>
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:47:43.307360+00:00", "user_id": "sv-local", "tokens_in": 4, "tokens_out": 36, "cost_usd": 2.22e-05}`
>
> Tôi có thể (1) lọc chính xác các event `ask_completed` của `sv-local` trong
> một khoảng thời gian và (2) cộng `tokens_in`, `tokens_out`, `cost_usd` để
> làm dashboard hoặc cảnh báo chi phí. Câu `print("đã trả lời xong")` không có
> timestamp, user, mức log hay số liệu dạng trường nên không làm tin cậy được
> hai việc đó.

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
| 1 stage (bản đầu) | 1.7 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build thật bằng hai tag `day12-agent:single` và `day12-agent:prod`;
> `docker images` lần lượt báo `1.7GB` và `271MB`. Phần chênh lệch chủ yếu là
> base image `python:3.11` đầy đủ (nhiều thư viện hệ thống và công cụ phục vụ
> phát triển) so với `python:3.11-slim`. Runtime stage chỉ nhận dependency đã
> cài và source cần chạy, không mang theo thư mục làm việc của builder hay dữ
> liệu trung gian. `--no-cache-dir` cũng tránh giữ cache tải package trong
> layer cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một thay đổi nhỏ vào `app/main.py` rồi build lại. Log BuildKit cho
> thấy các bước tạo user, `WORKDIR`, `COPY requirements.txt`, `RUN pip install`
> và `COPY --from=builder` đều là `CACHED`; hai layer `COPY app` và `COPY utils`
> cùng layer export chạy lại (layer `COPY utils` đứng sau layer source đã đổi).
> Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi trong source làm
> checksum của layer COPY đổi, kéo theo layer cài dependency phía sau chạy lại
> dù `requirements.txt` không đổi. Build sẽ chậm hơn và tải/cài package không
> cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro có thể là: lỗi Python cho phép thực thi lệnh từ xa; lệnh đó chạy
> với UID 0 trong container; kẻ tấn công tiếp tục lợi dụng lỗi kernel/container
> runtime hoặc một mount/socket Docker cấu hình sai để thoát khỏi namespace;
> khi đó quyền root trong container làm hậu quả trên host nghiêm trọng hơn.
> `USER app` cắt chuỗi ngay tại bước payload chạy trong container: kiểm tra
> thực tế cho kết quả `uid=999(app) gid=999(app)`, nên payload chỉ có quyền của
> user thường. Đây là lớp giảm thiểu phạm vi ảnh hưởng, không thay thế việc vá
> kernel và tránh mount đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là **20 request trong 2 giây**. User gửi 10 request ở cuối phút cũ,
> chẳng hạn 10:00:59.x. Bộ đếm reset ở 10:01:00, sau đó user gửi thêm 10
> request ở đầu phút mới, chẳng hạn 10:01:00.x. Mỗi bucket riêng vẫn chỉ có
> 10 request nhưng hai nhóm nằm trong một khoảng 2 giây. Sliding window 60
> giây vẫn nhìn thấy nhóm cũ nên chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit bảo vệ **tốc độ/số request trong cửa sổ ngắn**, còn cost guard bảo
> vệ **tổng tiền của từng user trong cả tháng**. Nếu một user đã dùng gần hết
> ngân sách tháng rồi nghỉ một giờ, request kế tiếp vẫn qua rate limit nhưng
> cost guard phải trả 402. Ngược lại, một user còn nguyên ngân sách nhưng gửi
> request thứ 11 trong chưa đầy 60 giây sẽ bị rate limiter trả 429, dù cost
> guard vẫn cho phép về mặt tiền.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Theo thứ tự: Redis mất kết nối; endpoint gộp bắt đầu thất bại trên cả ba
> container; orchestrator hiểu nhầm rằng ba process bị hỏng và loại/restart
> chúng; các container mới vẫn không gọi được Redis nên tiếp tục fail probe và
> tạo restart loop. Traffic mất cả ba replica dù bản thân code web vẫn sống,
> còn restart không thể sửa dependency Redis. Khi Redis trở lại, cụm còn phải
> chờ container khởi động và probe xanh lại. Tách `/health` và `/ready` giúp
> process vẫn sống, chỉ rút instance khỏi traffic trong lúc dependency lỗi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy ba agent qua Nginx và gọi sáu lần bằng cùng user. Log cho thấy
> request được chia cho `agent-1`, `agent-2`, `agent-3`, còn `history_length`
> vẫn tăng đều `0, 2, 4, 6, 8, 10` vì cả ba đọc cùng Redis. Nếu dùng dict
> Python, mỗi container có một bản riêng: request chuyển sang replica khác sẽ
> thấy lịch sử ngắn hơn hoặc quay về 0; dãy số sẽ tách thành ba dãy cục bộ và
> thay đổi thất thường theo replica được Nginx chọn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy thật lên Railway, `/health` trả 200 nhưng `/ready` trả 500 và
> `/ask` vẫn trả 401 dù tôi đã gửi API key. Tôi gọi riêng từng endpoint để tách
> lỗi process khỏi lỗi dependency, sau đó kiểm tra Variables của `day12-agent`.
> Nguyên nhân là `REDIS_URL` đang rỗng và `AGENT_API_KEY` trên Railway chưa khớp
> với key dùng để kiểm thử. Tôi đặt `REDIS_URL` bằng reference
> `${{day12-redis.REDIS_URL}}`, đồng bộ `AGENT_API_KEY`, rồi redeploy. Sau đó
> `/health` và `/ready` đều trả 200, request không key trả 401, request có key
> trả 200; 15 request cùng user còn cho đúng dãy mười mã 200 rồi năm mã 429.
