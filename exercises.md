# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Mạnh Đoan. Mã sinh viên: 2A202602839

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu tôi quên đặt `AGENT_API_KEY` thì `Settings`
> phát sinh `ValidationError` ngay lúc container khởi động. Nhờ vậy deployment
> không được nhận traffic và tôi nhìn thấy lỗi trong runtime log để bổ sung
> secret. Nếu code có mặc định `"changeme"`, service vẫn lên xanh nhưng bất kỳ
> ai đoán được khóa mặc định đều có thể gọi `/ask`, tiêu quota và ngân sách của
> tôi. Fail fast biến một lỗi cấu hình âm thầm thành lỗi triển khai nhìn thấy
> ngay, trước khi service được công khai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log JSON tôi thu được khi gọi `/ask` là:
>
> ```json
> {"event":"ask_completed","level":"info","timestamp":"2026-09-28T09:25:13+00:00","user_id":"sv-test","tokens_in":3,"tokens_out":35,"cost_usd":0.00002145}
> ```
>
> Từ các trường có cấu trúc này, tôi có thể lọc và cộng `cost_usd` theo
> `user_id` để tìm user tiêu nhiều ngân sách nhất. Tôi cũng có thể đếm số event
> theo thời gian, mức log hoặc loại sự kiện để làm dashboard/cảnh báo. Dòng
> `print("đã trả lời xong")` không chứa user, timestamp, token hay chi phí nên
> không hỗ trợ hai việc trên.

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
| 1 stage (bản đầu) | 1.68 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build hai image trên cùng máy và đo được `single_bytes=1679393000`, còn
> `multi_bytes=271180859`, tương ứng Docker hiển thị 1.68 GB và 271 MB. Chênh
> lệch chủ yếu đến từ base image `python:3.11` đầy đủ của bản một stage, gồm
> nhiều package hệ điều hành và công cụ không cần cho runtime. Bản production
> dùng `python:3.11-slim`; builder cài dependency vào `/install`, rồi stage cuối
> chỉ copy kết quả chạy cần thiết nên không mang theo môi trường build và các
> thành phần dư thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa `app/main.py` rồi build lại, các layer base image, `WORKDIR`,
> `COPY requirements.txt` và `RUN pip install` vẫn lấy từ cache vì
> `requirements.txt` không đổi. Layer `COPY app ./app` phải chạy lại; các layer
> đứng sau nó như `COPY utils` và tạo `appuser` cũng được dựng lại theo chuỗi
> cache. Tôi đã thấy bước cài dependency báo `CACHED` khi rebuild. Nếu đặt
> `COPY . .` trước `RUN pip install`, mọi thay đổi source đều làm layer copy
> đổi, kéo theo việc cài lại toàn bộ dependency dù requirements không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu API Python có lỗ hổng thực thi lệnh, kẻ tấn công có thể chạy lệnh với
> quyền của process trong container. Khi process là root, họ có quyền sửa file
> hệ thống trong container, cài công cụ và khai thác tiếp cấu hình mount hoặc
> lỗ hổng container runtime để tác động tới host với mức thiệt hại lớn hơn.
> Dockerfile của tôi tạo UID 10001 và đặt `USER appuser` trước khi chạy
> Uvicorn. Vì vậy mã bị khai thác chỉ có quyền của user thường; đây không loại
> bỏ lỗ hổng nhưng cắt chuỗi leo thang ở bước ứng dụng có quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong khoảng hai giây: gửi 10 request ngay
> trước khi phút cũ kết thúc, ví dụ 10:00:59, rồi gửi thêm 10 request ngay sau
> khi bộ đếm reset ở 10:01:00. Fixed window coi chúng thuộc hai phút khác nhau
> nên cả hai nhóm đều hợp lệ. Sliding window nhìn lại đúng 60 giây gần nhất,
> vì vậy nhóm request đầu vẫn còn trong cửa sổ và nhóm thứ hai sẽ bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số lượng request trong 60 giây, còn cost guard
> giới hạn tổng tiền một user đã tiêu trong tháng. Nếu user gửi một request
> rất dài mỗi phút thì vẫn dưới rate limit, nhưng khi tổng chi phí vượt 10 USD
> cost guard phải trả 402. Ngược lại, một user mới chỉ tiêu vài phần cent nhưng
> gửi 15 request rất ngắn liên tiếp thì cost guard vẫn cho phép về ngân sách,
> còn rate limiter chặn từ request thứ 11 bằng 429. Trong kiểm tra cloud của
> tôi, 10 request đầu trả 200 và 5 request cuối trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu gộp hai endpoint là: Redis mất kết nối; cả ba container
> cùng không ping được Redis; endpoint health của cả ba trả 503; orchestrator
> hiểu nhầm rằng ba process đã chết và restart cả ba gần như đồng thời; mọi
> request đang xử lý bị gián đoạn và cụm không còn instance khỏe để phục vụ.
> Nếu Redis quay lại trong lúc container vẫn đang restart, outage vẫn kéo dài
> hơn sự cố Redis ban đầu. Khi tách đúng, `/health` vẫn 200 vì process còn sống,
> còn `/ready` trả 503 để load balancer chỉ tạm ngừng gửi traffic mà không
> restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi hai instance dùng chung Redis, tôi quan sát request đầu có
> `history_length=0` và request kế tiếp cùng `X-User-Id` có
> `history_length=2` (một message user và một message assistant), dù dữ liệu
> được đọc qua instance/store khác. Nếu dùng dict Python, mỗi container có
> một bản history riêng: request rơi luân phiên vào ba container sẽ cho số liệu
> kiểu 0, 0, 0 rồi 2, 2, 2 thay vì tăng đều 0, 2, 4, 6. Khi container restart,
> history trong dict còn trở về 0 hoàn toàn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thực tế tôi gặp là deployment Railway báo `SUCCESS` nhưng gọi domain trả
> `502 Application failed to respond`. Tôi đọc runtime log và thấy Uvicorn đang
> lắng nghe tại `0.0.0.0:8080`, trong khi lúc tạo domain tôi đã chỉ định target
> port 8000. Tôi cập nhật target port của domain Railway từ 8000 sang 8080.
> Sau đó `/health` trả 200, `/ready` trả 200 với `redis=true`, `/ask` không key
> trả 401, `/ask` có key trả 200 và bài test rate limit trả mười mã 200 rồi năm
> mã 429. Qua lỗi này tôi hiểu rằng ứng dụng phải đọc `$PORT` và domain phải
> chuyển traffic tới đúng cổng runtime mà platform cấp.
