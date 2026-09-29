# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Mười câu dưới đây được trả lời từ kết quả chạy bài và triển khai thực tế.
>
> Họ và tên: Phạm Hoàng Trọng  Mã học viên: 2A202602765

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy Render mà quên đặt `AGENT_API_KEY`, app dừng ngay khi khởi động với `ValidationError` ở trường `agent_api_key`; mình đã thử chạy thiếu biến này và thấy tiến trình thoát với mã 1. Nhờ vậy mình biết cấu hình đang thiếu trước khi service nhận request. Nếu để mặc định `"changeme"`, service vẫn chạy và bất kỳ ai biết hoặc đoán được khóa mặc định đều có thể gọi `/ask`.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng mình lấy từ log của agent khi gọi `/ask`:
>
> `{"user_id": "cp4-shared-ae7655c0", "tokens_in": 93, "tokens_out": 48, "cost_usd": 4.275e-05, "event": "ask_completed", "level": "info", "timestamp": "2026-09-29T02:44:15.057723+00:00"}`
>
> Mình có thể lọc chính xác các event `ask_completed` theo `user_id` và thời gian để xem request của một người dùng. Mình cũng có thể cộng `cost_usd` hoặc `tokens_out` từ các dòng log để theo dõi chi phí và mức sử dụng. Dòng `print("đã trả lời xong")` không có các field này để máy lọc hoặc tính toán.

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
| 1 stage (lệnh của bản đầu, dùng `python:3.11-slim` để so cùng base) | 319 MB |
| Multi-stage | 309 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Mình đo được `agent:single` 319 MB và `day12-agent:prod` 309 MB bằng `docker images`. Để so tác động của cách build, mình dùng lại các lệnh của Dockerfile một stage ban đầu nhưng đổi base sang cùng `python:3.11-slim`; bản gốc dùng `python:3.11` đầy đủ chưa có số đo vì tải base image từ Docker Hub quá chậm. Chênh lệch **10 MB** trong phép đo cùng base đến từ việc image một stage giữ cả nội dung của bước `pip install` (gồm cache tải gói) trong runtime; bản multi-stage chỉ chép virtualenv từ `builder`. Nếu dùng đúng base đầy đủ của bản gốc, image một stage còn lớn hơn do có thêm các thành phần của base đó, nhưng mình không ghi số ước lượng vào bảng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi mình build lại sau khi sửa source ở các checkpoint, bước copy `requirements.txt` và `RUN pip install` của stage `builder` hiện `CACHED` vì hai đầu vào đó không đổi. Bước `COPY app` phải chạy lại; các bước runtime phía sau nó cũng có thể bị mất cache do layer cha thay đổi. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ một thay đổi ở `app/main.py` cũng làm layer copy đổi, kéo theo việc cài lại toàn bộ dependency dù `requirements.txt` y nguyên. Dockerfile hiện tại đặt cài dependency trước copy source để tránh việc đó.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Ví dụ code xử lý request có lỗ hổng cho phép chạy lệnh tùy ý. Kẻ tấn công có thể dùng lỗ hổng để chạy lệnh trong container; nếu tiến trình là root và container có mount nhạy cảm hoặc quyền truy cập Docker socket, lệnh đó có thể sửa dữ liệu trên host hoặc tạo container có đặc quyền cao. `USER` chuyển tiến trình sang tài khoản thường (UID 10001), nên mã bị chiếm quyền chỉ có quyền của tài khoản này và khó sửa file hệ thống hay file trên mount không cho phép ghi. Đây là một lớp giảm thiệt hại; nó không tự ngăn mọi kiểu thoát container hay cấu hình mount quá rộng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi **20 request** trong khoảng 2 giây: 10 request ngay trước mốc đổi phút, rồi 10 request ngay sau giây 00. Bộ đếm theo phút coi đó là hai phút khác nhau nên cho qua cả hai nhóm. Sliding window nhìn lại đúng 60 giây tính từ từng request; khi nhóm đầu vẫn còn trong cửa sổ, request thứ 11 sẽ bị chặn bằng 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tốc độ** gọi (10 request trong 60 giây), còn cost guard giới hạn **chi phí tích lũy trong tháng UTC** (10 USD). Nếu một người mới gửi request đầu tiên trong phút nhưng đã tiêu 10,01 USD trong tháng, rate limit cho qua còn cost guard trả 402. Ngược lại, nếu họ đã gửi 10 request trong 60 giây nhưng mới tiêu 0,001 USD, cost guard vẫn cho qua còn rate limit trả 429 và `Retry-After`. Cả hai được kiểm tra trước khi gọi LLM.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng ping Redis, Redis mất kết nối sẽ làm cả ba agent trả 503 ở health check. Sau vài lần kiểm tra thất bại, orchestrator có thể đánh dấu cả ba container không khỏe và restart chúng; Redis vẫn lỗi nên container mới lại không khỏe, gây vòng lặp restart mà không sửa được nguyên nhân. Trong lần mình dừng Redis để thử, cả ba `/health` vẫn trả 200, còn `/ready` trả 503; khi Redis chạy lại, `/ready` trở về 200. Cách này giữ tiến trình sống nhưng tạm ngừng nhận traffic cho tới khi dependency sẵn sàng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy ba agent ở cổng 8000, 8001, 8002 và gọi `/ask` với cùng `X-User-Id`. `history_length` quan sát được lần lượt là **0, 2, 4**: mỗi câu hỏi thêm hai message (user và assistant), và instance sau đọc được lịch sử của instance trước qua Redis. Nếu lưu bằng dict Python, mỗi container có dict riêng; khi request đi sang container khác, giá trị sẽ bắt đầu lại từ 0 hoặc tăng theo lịch sử riêng của container đó, nên có thể thấy 0, 0, 0 thay vì 0, 2, 4.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi tạo Blueprint trên Render, màn hình chọn repository báo **“No repositories found”**. Mình kiểm tra lại phần kết nối GitHub và thấy Render chưa có quyền truy cập repository trong tài khoản này, dù repository đã được push lên GitHub. Mình nhập URL của public repository vào mục **Public Git Repository**; Render đọc được `render.yaml`, tạo web service cùng Key Value và deploy thành công. Đây là lỗi ở bước chọn nguồn deploy, không phải lỗi build của ứng dụng.
