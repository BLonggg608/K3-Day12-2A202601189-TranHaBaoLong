# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.**
> Họ và tên: Trần Hà Bảo Long - Mã học viên: 2A202601189

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy, nếu `AGENT_API_KEY` bị thiếu thì `Settings` ném `ValidationError` ngay khi app đọc cấu hình. Nhờ vậy tôi phát hiện lỗi trước khi mở service công khai. Nếu dùng `changeme`, app vẫn chạy nhưng khóa rất dễ bị đoán.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log JSON có event, level, timestamp, user, token và cost nên cloud có thể lọc và tổng hợp tự động. `print` chỉ là text tự do, thiếu schema và dữ liệu có cấu trúc.

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
| 1 stage (bản đầu) | 1.73 GB disk usage; 446 MB content size |
| Multi-stage | 270 MB disk usage; 63.7 MB content size |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Multi-stage giảm khoảng 1.46 GB disk usage và 382.3 MB content size vì image runtime không chứa build environment, cache và file trung gian của stage builder.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Docker có thể cache `COPY requirements.txt` và `RUN pip install` khi requirements không đổi. Nếu `COPY . .` đặt trước `RUN pip install`, mỗi lần sửa source sẽ làm mất cache pip và phải cài lại dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app bị khai thác khi chạy root, attacker có thể ghi file, cài công cụ hoặc thực hiện thao tác đặc quyền trong container. `USER appuser` giảm quyền và giới hạn blast radius của lỗi khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với cách đếm theo phút đồng hồ, có thể gửi 20 request trong 2 giây: 10 request ngay trước khi chuyển phút và 10 request ngay sau đó. Sliding window đếm đúng 60 giây gần nhất nên request thứ 11 bị trả 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lượng request theo thời gian, còn cost guard giới hạn tổng chi phí theo tháng. User chưa vượt 10 request/phút nhưng đã tiêu 9,99 USD và request mới tốn 0,02 USD sẽ bị cost guard trả 402. Ngược lại, user còn ngân sách nhưng gửi quá nhanh sẽ bị rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> `/health` chỉ kiểm tra process còn sống và không phụ thuộc Redis. `/ready` kiểm tra Redis. Khi Redis mất, `/ready` trả 503 để load balancer ngừng gửi traffic, còn `/health` vẫn có thể trả 200.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu history nằm trong dict Python, mỗi container có RAM riêng nên request vào container khác có thể không thấy lịch sử và `history_length` tăng không đều. Redis giúp cả ba container dùng chung state.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần đầu Railway chạy literal `$PORT`, khiến Uvicorn báo `'$PORT' is not a valid integer`. Tôi xem runtime log, bỏ start command lỗi và dùng shell form `${PORT:-8000}` trong Dockerfile. Sau đó bổ sung `AGENT_API_KEY` và `REDIS_URL` trên Railway; `/health` và `/ready` đều trả 200.
