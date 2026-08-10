# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Quang Tùng   Mã học viên: 2A202601545

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Ví dụ cụ thể của mình là nếu quên set `AGENT_API_KEY` trên cloud mà app vẫn
khởi động được, người lạ có thể gọi `/ask` miễn phí và mình chỉ phát hiện ra
sau khi hóa đơn hoặc log đã tăng bất thường. Chết sớm giúp mình biết ngay lúc
deploy rằng cấu hình còn thiếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Mình có thể dùng log JSON để lọc theo `event="ask_completed"`, đếm số request
thành công, hoặc tìm tất cả request của một `user_id` cụ thể để debug. Ngoài
ra mình còn có thể parse log bằng máy để cảnh báo khi `cost_usd` tăng bất
thường, trong khi `print("đã trả lời xong")` chỉ là chuỗi người đọc, không dễ
thống kê hay truy vết.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Với bản 1 stage, image thường lớn hơn vì nó mang theo cả dependency cài đặt,
tool build và nhiều thứ không cần cho runtime. Với multi-stage, phần chênh lệch
là compiler, package build và các file trung gian chỉ dùng ở stage builder;
stage runtime chỉ giữ đúng những gì app cần để chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi mình sửa một ký tự trong `app/main.py`, Docker chỉ phải build lại các layer
copy code và các layer phía sau nó. Layer cài dependency từ `requirements.txt`
vẫn được cache nên build nhanh hơn. Nếu đặt `COPY . .` trước `RUN pip install`,
mỗi lần đổi code là Docker coi cả context thay đổi và cài lại toàn bộ thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu container chạy root và app có lỗ hổng thực thi lệnh hoặc đọc/ghi file tùy
ý, kẻ tấn công có thể chiếm quyền root bên trong container. Từ đó họ dễ sửa
thêm payload, đọc dữ liệu nhạy cảm và trong một số tình huống có thể lợi dụng
sai cấu hình để chạm sang tài nguyên của host. `USER` chuyển app xuống user
thường nên dù bị khai thác, quyền cũng bị giới hạn hơn nhiều.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Đếm theo phút đồng hồ có thể bị lợi dụng ở ranh giới phút: người dùng gửi 10
request ngay trước phút mới và 10 request ngay sau khi phút đổi, tổng cộng là
20 request trong khoảng 2 giây. Sliding window 60 giây chặn được kiểu “vượt
ranh giới phút” này vì nó đếm đúng 60 giây gần nhất, không phụ thuộc đồng hồ
chia phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit chặn số lượng request trong một khoảng thời gian, còn cost guard
chặn tổng tiền đã tiêu trong tháng. Ví dụ, user gọi quá nhanh thì rate limit
trả 429 dù tiền vẫn còn. Ngược lại, user gọi không quá nhanh nhưng mỗi request
đắt khiến vượt ngân sách thì cost guard trả 402.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp `/health` và `/ready` lại rồi bắt nó kiểm tra Redis, khi Redis mất
kết nối thì endpoint đó sẽ trả lỗi. Load balancer sẽ tưởng instance chết và
ngừng hoặc restart container; với 3 container thì cả 3 có thể lần lượt bị loại
khỏi traffic dù process vẫn còn sống. Tách riêng giúp `/health` luôn nhẹ và chỉ
báo process còn sống, còn `/ready` mới kiểm tra phụ thuộc.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi state nằm trong dict Python, mỗi container có một bản nhớ riêng. Vì vậy
request vào container A không xuất hiện ở container B. Nếu dùng Redis thì hai
instance nhìn cùng một lịch sử, nên `history_length` tăng đều qua các request.
Nếu để trong RAM, con số đó sẽ bị lệch hoặc reset tùy container nào nhận request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Một lỗi mình gặp là Render probe vào `/` và trả `404 Not Found` trong log.
Mình kiểm tra `Settings` của service, xác nhận `Health Check Path` phải là
`/health`, rồi redeploy lại. Sau đó Render bắt đầu gọi đúng `/health` và
service lên `Live` thành công.
