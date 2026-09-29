# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hồng Khoa  Mã học viên: 2A202602534

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy tôi quên đặt `AGENT_API_KEY` trên dashboard. Nếu
> code có mặc định `"changeme"`, service vẫn khởi động và endpoint công khai có
> thể bị gọi bằng khóa dễ đoán. Khi trường này không có mặc định, Pydantic báo
> `ValidationError` ngay lúc startup, nên bản deploy lỗi trước khi nhận traffic
> và trước khi phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> CP3 chưa hoàn thành nên tôi chưa có log sinh ra từ một lần gọi `/ask`. Khi
> kiểm tra hàm logging trực tiếp, tôi thu được một dòng cùng cấu trúc sau:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:17:44.597981+00:00", "user_id": "sv-test", "cost_usd": 0.0001}
> ```
>
> Từ JSON này, hệ thống log có thể lọc hoặc nhóm theo `user_id` để tìm người gọi
> nhiều nhất, đồng thời cộng `cost_usd` để theo dõi chi phí và đặt cảnh báo.
> Chuỗi `print("đã trả lời xong")` không có trường dữ liệu ổn định để thực hiện
> hai việc đó. Tôi sẽ thay dòng minh họa bằng log `/ask` thực tế sau CP3.

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
| 1 stage (bản đầu) | 1617.8 MB |
| Multi-stage | 258.6 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi đo bằng `docker image inspect` sau khi build cả hai image. Phần chênh lệch
> chủ yếu đến từ image `python:3.11` đầy đủ của bản một stage, chứa nhiều gói hệ
> thống, công cụ và dữ liệu không cần khi chạy ứng dụng. Bản mới dùng
> `python:3.11-slim`; stage builder chỉ tạo bộ dependency trong `/install`, rồi
> stage runtime nhận kết quả đó cùng mã nguồn cần chạy. Vì vậy image cuối không
> mang toàn bộ môi trường build và base image đầy đủ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một comment vào `app/main.py` rồi build lại. Docker báo các layer
> `WORKDIR /build`, `COPY requirements.txt`, `pip install`, `WORKDIR /app` và
> `COPY --from=builder /install /usr/local` đều `CACHED`. Từ `COPY app ./app`
> trở đi phải chạy lại, gồm cả `COPY utils` và bước tạo/chown user vì chúng nằm
> sau layer đã thay đổi. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi
> trong source sẽ làm mất cache của layer copy và khiến `pip install` chạy lại,
> dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công trước hết có
> quyền của process trong container. Khi process chạy root, họ có toàn quyền
> trong container và có thể lợi dụng thêm lỗi container escape hoặc cấu hình
> nguy hiểm như mount Docker socket/host path để tác động tới host với quyền
> cao. `USER appuser` làm process chạy bằng UID 10001, nên bước đầu chỉ nhận
> quyền của user thường; phạm vi ghi file và khả năng khai thác tiếp bị thu hẹp.
> Lệnh này giảm thiệt hại nhưng không tự thay thế việc vá lỗ hổng hay cấu hình
> container an toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong hai giây: 10 request ở cuối
> phút thứ nhất, ví dụ 10:00:59, rồi 10 request ngay đầu phút tiếp theo, ví dụ
> 10:01:00. Bộ đếm theo phút thấy mỗi nhóm thuộc một bucket khác nhau nên đều
> cho qua. Sliding window 60 giây nhìn cả 20 request trong cùng một cửa sổ nên
> chặn kẽ hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ request trong một khoảng thời gian ngắn; cost guard
> giới hạn tổng tiền đã dùng theo user trong cả tháng. Một user gửi mỗi phút một
> request rất dài có thể không vượt rate limit nhưng vẫn bị cost guard chặn khi
> hết ngân sách. Ngược lại, một user chưa tốn đáng kể ngân sách nhưng gửi 11
> request rất rẻ trong vài giây có thể còn ngân sách và vẫn bị rate limit chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu Redis mất kết nối, endpoint đã gộp sẽ trả lỗi ở cả ba container. Bộ điều
> phối hiểu đó là lỗi liveness nên loại rồi restart cả ba process, dù bản thân
> ứng dụng vẫn chạy. Trong lúc restart, cụm không còn instance phục vụ request.
> Nếu Redis vẫn chưa trở lại, các container mới tiếp tục fail health check và có
> thể tạo vòng lặp restart. Tách `/health` và `/ready` cho phép process vẫn sống,
> còn load balancer chỉ tạm ngừng gửi traffic cho tới khi Redis hoạt động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> **Cần cập nhật sau CP4:** hiện tại `/ask` và Redis conversation store chưa
> hoàn thành nên tôi chưa chạy được phép thử ba instance. Kết quả dự kiến khi
> dùng Redis là `history_length` tăng nhất quán dù request vào container nào.
> Nếu dùng dict Python, mỗi container có một lịch sử riêng nên số này sẽ tăng
> theo từng chuỗi rời rạc, có thể quay về 0 hoặc giá trị thấp khi load balancer
> chuyển request sang instance khác.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Cần cập nhật sau CP5:** chưa có bản deploy cloud nên chưa có lỗi cloud thực
> tế để ghi nhận. Lỗi startup cục bộ gần nhất là
> `NotImplementedError: TODO (CP4): cài đặt install`, do lifespan gọi
> `lifecycle.install()` khi hàm còn để trống. Tôi lần theo traceback tới
> `app/lifecycle.py`, cài đặt đăng ký `SIGTERM`/`SIGINT` và gọi lại handler cũ;
> sau đó Compose khởi động thành công và container chuyển sang trạng thái
> `healthy`. Tôi sẽ thay đoạn này bằng lỗi từ lần deploy cloud thực tế.
