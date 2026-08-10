# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Cao Nhật Minh  Mã học viên: 2A202601721

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy mà quên cấu hình `AGENT_API_KEY`, app dừng ngay và báo lỗi để tôi bổ sung secret trước khi nhận traffic. Nếu có khóa mặc định `"changeme"`, app vẫn chạy bình thường, khiến người lạ dễ đoán khóa, gọi API trái phép và làm phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T03:04:23+00:00","user_id":"sv01","cost_usd":0.0001}`. Từ các trường JSON, tôi có thể lọc request theo `user_id` và cộng `cost_usd` để theo dõi chi phí. Dòng `print("đã trả lời xong")` không chứa các trường riêng biệt nên hệ thống log khó tìm kiếm, thống kê hoặc cảnh báo tự động.

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
| 1 stage (bản đầu) | 1690 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản multi-stage nhỏ hơn khoảng 1420 MB. Phần chênh lệch chủ yếu đến từ base image Python đầy đủ cùng các công cụ, cache và thành phần phục vụ build. Stage runtime chỉ dùng image `slim` và sao chép thư viện đã cài cùng source cần chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Vì `requirements.txt` không đổi, Docker dùng lại cache của layer copy requirements và cài dependency. Chỉ layer copy source cùng các layer phía sau phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, một thay đổi nhỏ trong code cũng làm cache bị mất và Docker phải tải, cài lại toàn bộ thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Kẻ tấn công có thể lợi dụng lỗ hổng Python để thực thi lệnh trong container. Nếu process chạy bằng root, lệnh đó có quyền sửa file hệ thống hoặc lợi dụng volume/cấu hình sai để tác động đến host. `USER appuser` chuyển process sang UID thường, nên mã bị khai thác chỉ có quyền hạn chế và giảm mức độ thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong 2 giây: gửi 10 request vào giây 59 của phút trước, sau đó gửi tiếp 10 request vào giây 00 của phút mới. Bộ đếm theo phút đã reset nên vẫn xem cả hai nhóm là hợp lệ, còn sliding window sẽ nhìn lại đúng 60 giây gần nhất và chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong một khoảng thời gian, còn cost guard giới hạn tổng tiền đã tiêu trong tháng. Một request rất dài có thể vẫn qua rate limit nhưng bị cost guard chặn vì vượt ngân sách. Ngược lại, nhiều request rất rẻ gửi dồn dập có thể còn ngân sách nhưng vẫn bị rate limit chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint gộp trả 503 ở cả 3 container. Orchestrator hiểu đây là lỗi liveness nên lần lượt restart cả 3, dù các process vẫn còn sống. Trong lúc Redis phục hồi, toàn bộ agent cũng đang khởi động lại nên service mất khả năng phục vụ. Tách `/health` và `/ready` giúp load balancer chỉ ngừng gửi traffic mà không restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi dùng Redis chung, `history_length` tăng đều vì container nào cũng đọc được lịch sử của cùng user. Nếu dùng dict Python, mỗi container giữ một bản riêng nên con số sẽ tăng không ổn định, có thể quay về 0 khi request tiếp theo được chuyển sang container chưa từng gặp user đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
