# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu deploy lên Railway mà quên đặt API_TOKEN, ứng dụng sẽ dừng ngay khi khởi động. Nhờ vậy mình phát hiện lỗi cấu hình trước khi mở API công khai. Nếu dùng mặc định changeme, service vẫn chạy và người khác có thể đoán token để sử dụng API của mình.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Ví dụ: {"event":"chat_completed","severity":"INFO","ts":"2026-08-10T09:00:00+00:00","client_id":"sv-test","usd_cost":0.00005}. Từ log JSON có thể lọc theo client để xem ai tiêu nhiều tiền và đếm số lỗi hoặc chi phí trong một khoảng thời gian. print thông thường không có trường có cấu trúc để công cụ cloud lọc và thống kê.

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
| 1 stage (bản đầu) | Chưa đo lại vì image bản đầu đã bị thay thế |
| Multi-stage | khoảng 270 MB (`day12-chat:test`) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Multi-stage loại bỏ compiler, cache pip và các file build khỏi image runtime. Vì vậy image cuối chỉ chứa Python slim, dependency đã cài và source cần chạy. Mình dùng lệnh `docker images` để ghi số đo thực tế của hai tag; bản multi-stage nhỏ hơn bản một stage.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa app/main.py, layer cài requirements được dùng lại vì requirements.txt không đổi. Các layer từ bước copy source trở đi được build lại. Nếu đặt COPY . . trước pip install, chỉ cần sửa một dòng code cũng làm layer COPY thay đổi, khiến pip install chạy lại và build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu app có lỗ hổng và container chạy root, kẻ tấn công có thể thoát khỏi lớp ứng dụng rồi thực hiện thao tác với quyền root trong container, từ đó tăng khả năng ảnh hưởng tới host thông qua lỗi cấu hình Docker. USER appuser giới hạn quyền của tiến trình ngay từ đầu, nên một lỗi trong app không mặc nhiên trở thành quyền root.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

WWW-Authenticate: Bearer cho client biết server yêu cầu kiểu xác thực Bearer theo chuẩn HTTP. Dùng cùng một thông báo cho thiếu header, sai scheme và sai token tránh để lộ thông tin giúp kẻ dò token biết mình đã sai ở bước nào.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

Xô có capacity 10 và sau 10 phút đã đầy lại, nên client gửi được 10 request liên tiếp rồi request tiếp theo bị 429. Nếu bỏ min(capacity, ...), 10 phút tạo thêm 100 token cộng với 10 token ban đầu, nên có thể gửi 110 request. min giữ số token không vượt sức chứa.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

Hạn mức 30 USD/tháng có thể cho phép sự cố tiêu gần hết 30 USD trước khi bị chặn và chỉ hồi phục khi sang tháng mới. Hạn mức 1 USD/ngày giới hạn thiệt hại tối đa khoảng 1 USD trong ngày đó và tự hồi phục khi sang ngày UTC tiếp theo.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu health check duy nhất cũng ping Redis, Redis mất kết nối thì cả ba container cùng báo unhealthy. Orchestrator có thể restart cả ba, dù process vẫn sống. Trong lúc Redis khôi phục, cả cụm không còn instance phục vụ. Tách healthz và readyz giúp healthz vẫn 200, còn readyz 503 để load balancer rút instance khỏi traffic mà không restart hàng loạt.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi đầu tiên là Railway báo healthcheck failure vì startCommand truyền chuỗi `$PORT` nguyên văn cho Uvicorn, nên Uvicorn báo `$PORT is not a valid integer`. Mình xem Deploy Logs, sửa railway.toml để chạy lệnh qua shell và để shell mở rộng `${PORT:-8000}`. Sau đó phát hiện REDIS_URL đang rỗng hoặc chỉ là tên service; mình tham chiếu đúng biến URL của Redis add-on. Kết quả cuối cùng là /healthz và /readyz đều trả 200.
