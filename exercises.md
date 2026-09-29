# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần giữ chỗ dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thị Bảo Trang  Mã học viên: 2A202602580

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, nếu tôi quên đặt `AGENT_API_KEY` thì ứng dụng dừng ngay
> lúc khởi động và log chỉ rõ biến cấu hình bắt buộc đang thiếu. Nhờ vậy tôi sửa
> cấu hình trước khi service nhận traffic. Nếu code dùng khóa mặc định
> `"changeme"`, service vẫn có vẻ hoạt động nhưng bất kỳ ai đoán được khóa này đều
> có thể gọi `/ask`, làm phát sinh chi phí và truy cập API trái phép.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi lấy được khi chạy code là:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T15:02:28.414008+00:00", "user_id": "sv-test", "tokens_in": 4, "tokens_out": 8, "cost_usd": 1.2e-05}`.
> Với log JSON này, tôi có thể lọc tất cả sự kiện `ask_completed` của một
> `user_id` và tổng hợp `cost_usd` hoặc số token để theo dõi chi phí. Tôi cũng có
> thể tạo cảnh báo theo `level`, thời gian hay mức chi phí. Một câu `print` tự do
> không có các trường ổn định để hệ thống log truy vấn và tổng hợp như vậy.

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
| 1 stage (bản đầu) | khoảng 1.02 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản một stage dùng image `python:3.11` đầy đủ và giữ toàn bộ công cụ, cache cài
> package cùng source trong image cuối. Bản multi-stage dùng `python:3.11-slim`;
> stage runtime chỉ nhận các dependency đã cài từ builder cùng `app/` và `utils/`,
> nên không mang theo phần dư của image đầy đủ hay môi trường build. Vì vậy image
> runtime giảm còn 271 MB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer của builder gồm base image, `COPY
> requirements.txt` và `RUN pip install` vẫn được lấy từ cache vì
> `requirements.txt` không đổi. Ở runtime, các layer trước `COPY app ./app` vẫn
> dùng lại; layer copy `app` và các layer đứng sau nó phải được tạo lại. Nếu đặt
> `COPY . .` trước `RUN pip install`, mọi thay đổi source sẽ làm checksum của layer
> copy thay đổi và buộc Docker chạy lại bước cài dependency dù dependency không
> hề đổi, làm build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng thực thi lệnh từ xa, kẻ tấn công trước tiên chạy lệnh
> với UID của process trong container. Khi process là root, họ có toàn quyền sửa
> file và tiến trình trong container; nếu đồng thời khai thác được lỗ hổng thoát
> container hoặc có mount nhạy cảm như Docker socket, quyền đó có thể dẫn đến
> root trên host. `USER appuser` cắt chuỗi ở bước đầu bằng cách để process chỉ có
> UID 10001 không đặc quyền, nên giảm đáng kể tác hại và khả năng leo thang. Đây
> vẫn là lớp giảm thiểu rủi ro, không thay thế việc vá lỗ hổng hay cấu hình mount
> an toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong hai giây: gửi 10 request ở giây
> 59 của phút trước, rồi gửi thêm 10 request ở giây 00 hoặc 01 của phút sau. Bộ
> đếm theo phút đồng hồ đã reset nên cả hai nhóm đều hợp lệ, dù thực tế chúng nằm
> sát nhau. Sliding window 60 giây vẫn nhìn thấy nhóm đầu khi nhóm sau tới và sẽ
> chặn request vượt quá 10.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit bảo vệ tốc độ gọi trong một cửa sổ ngắn, còn cost guard bảo vệ tổng
> tiền đã dùng theo user trong cả tháng. Nếu user đã dùng gần hết 10 USD nhưng đây
> mới là request đầu tiên trong phút, rate limit cho qua còn cost guard phải chặn
> bằng 402. Ngược lại, nếu user gửi request rẻ thứ 11 trong 60 giây khi tổng chi
> phí tháng còn rất thấp, cost guard vẫn cho phép nhưng rate limit chặn bằng 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai endpoint và bắt health kiểm tra Redis, khi Redis mất kết nối thì cả
> ba container cùng trả health lỗi. Orchestrator đánh dấu cả ba là không khỏe và
> khởi động lại chúng dù process ứng dụng vẫn sống. Các container mới vẫn không
> kết nối được Redis nên lại fail, tạo vòng lặp restart và làm mất toàn bộ capacity.
> Khi tách riêng, `/health` vẫn báo process còn sống nên không restart; `/ready`
> trả 503 để load balancer tạm ngừng đưa traffic vào. Khi Redis trở lại, readiness
> tự thành công và các instance nhận traffic tiếp mà không cần restart hàng loạt.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, các request dù rơi vào instance nào cũng đọc cùng một lịch
> sử nên `history_length` tăng đều theo số message trước đó, ví dụ 0, 2, 4, 6.
> Nếu dùng dict Python, mỗi instance giữ một bản riêng. Khi load balancer chuyển
> request giữa ba instance, tôi sẽ thấy số này nhảy hoặc lặp như 0, 0, 2, 0, 2 thay
> vì tăng liên tục; restart một instance còn làm phần lịch sử của nó trở về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi tạo Blueprint trên Render, cấu hình ban đầu dùng `type: redis` nên Render
> không tạo đúng dịch vụ Redis theo schema hiện tại. Tôi kiểm tra phần Blueprint
> và dashboard deploy, đối chiếu loại dịch vụ Render đang hỗ trợ, rồi đổi cả tham
> chiếu `fromService.type` và service từ `redis` thành `keyvalue`; đồng thời đặt
> cùng region Singapore. Sau khi redeploy, service chuyển sang `Live`, `/ready`
> trả `200` với `{"status":"ready","redis":true}` và request có API key trả 200.
