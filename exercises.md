# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu (trích dẫn in nghiêng) bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tiến Phát  Mã học viên: 2A202602387

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Fail fast là nguyên tắc để hệ thống dừng ngay khi thiếu cấu hình bắt buộc, thay vì chạy tiếp với giá trị mặc định không an toàn. Lỗi lộ ra lúc khởi động hoặc deploy, khi sửa còn rẻ.

Giả sử code có `AGENT_API_KEY` mặc định là "changeme" và tôi quên set biến này trên cloud. App vẫn chạy bình thường, dashboard báo Online, nhưng ai biết chuỗi "changeme" cũng gọi được API của tôi. Họ có thể gọi liên tục và tôi chỉ biết khi nhận hóa đơn LLM.

Nếu không có giá trị mặc định thì thiếu key là app chết ngay khi khởi động. Tôi thấy lỗi lúc deploy chứ không phải lúc nhận hóa đơn.

Tôi đã tự gặp chuyện tương tự trên Railway. App vẫn Online vì config chỉ được đọc khi có request, nên lỗi chỉ lộ ra khi gọi `/ready` và `/ask`.

**Reference:** https://12factor.net/config

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T17:34:19.908790+00:00", "user_id": "sv01", "tokens_in": 46, "tokens_out": 54, "cost_usd": 3.93e-05}
```

Hai việc làm được với dòng log này mà `print()` không làm được:

- Lọc hoặc cộng `cost_usd` theo `user_id`, ví dụ tìm ngay user nào tiêu nhiều tiền nhất trong ngày. Với `print()` tôi phải viết regex để tách chuỗi, và chỉ cần đổi câu chữ là regex hỏng.
- Đặt cảnh báo theo trường, ví dụ báo khi số dòng `event = "ask_failed"` vượt ngưỡng trong 5 phút. Ngoài ra có thể gộp log từ nhiều container rồi sắp xếp theo `timestamp` để dựng lại đúng thứ tự sự việc.

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
| 1 stage (bản đầu) | 1.73 GB (~1730 MB) |
| Multi-stage | 297 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

| Base image | Kích thước | Có gì |
|---|---|---|
| Bản đầy đủ | 1.73 GB | Compiler, header `-dev`, git, tài liệu |
| Bản slim | 297 MB | Chỉ phần tối thiểu để chạy Python |

Phần chênh khoảng 1.4 GB là các công cụ build. Thư viện Python ở hai bản gần như bằng nhau. Runtime không cần compiler hay git, nên bản slim vừa nhẹ hơn vừa ít thứ để kẻ tấn công lợi dụng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

| Layer | Build lại sau khi sửa code |
|---|---|
| `COPY requirements.txt` | Dùng lại từ cache |
| `pip install` | Dùng lại từ cache |
| `useradd` | Dùng lại từ cache |
| `COPY app` | Chạy lại |
| `COPY utils` | Chạy lại |

Build lại mất khoảng 1 giây. Nếu đặt `COPY . .` lên trước, sửa một dòng code cũng làm mất cache của `pip install`, phải cài lại toàn bộ thư viện (12 giây, tổng 14 giây).

> Nguyên tắc: layer đầu tiên có thay đổi sẽ làm mất cache của nó và mọi layer phía sau. Vì vậy phần ít đổi (dependency) đặt trước, phần hay đổi (code) đặt sau.

**Reference:** https://docs.docker.com/build/cache/, https://docs.docker.com/build/building/best-practices/

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:

1. Code có lỗ hổng, ví dụ cho phép chạy lệnh tùy ý. Kẻ tấn công có shell trong container, với quyền của user đang chạy app.
2. Nếu user đó là root, họ cài được công cụ, đọc mọi file, và nếu thoát được ra ngoài container (container escape) thì họ là root trên máy host.
3. `USER appuser` cắt chuỗi này ngay ở bước 2: container của tôi chạy bằng `appuser`, không cài thêm được gì, không sửa được file hệ thống. Image slim lại không có gcc và git, nên kẻ tấn công cũng không có sẵn công cụ để leo thang.

**Reference:** https://docs.docker.com/build/building/best-practices/

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Giả sử giới hạn là 10 request mỗi phút theo cửa sổ cố định. Tôi gửi 10 request ở giây :59, rồi 10 request nữa ở giây :00 của phút tiếp theo. Hai nhóm rơi vào hai cửa sổ khác nhau nên đều được cho qua. Tổng cộng là **20 request trong 2 giây**, gấp đôi mức cho phép.

> Cửa sổ trượt 60 giây luôn đếm số request trong 60 giây tính ngược từ thời điểm hiện tại, thay vì đếm theo phút cố định.

Ở giây :00, cửa sổ trượt vẫn còn 10 request của giây :59 nên đã đầy, các request mới bị chặn. Cửa sổ trượt chỉ mở lại khi các request cũ trôi ra khỏi 60 giây.

**Reference:** https://blog.cloudflare.com/counting-things-a-lot-of-different-things/

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tốc độ (số request mỗi phút), còn cost guard giới hạn tổng tiền (USD mỗi tháng).

- **Rate limit cho qua nhưng cost guard chặn:** ít request nhưng mỗi request rất đắt (prompt dài, nhiều token), hoặc gọi đều đặn dưới ngưỡng suốt cả tháng cho tới khi chạm trần USD.
- **Rate limit chặn nhưng cost guard cho qua:** gửi dồn dập nhiều request rẻ trong vài giây, như lúc tôi gọi 15 lần và thấy 429. Tổng chi phí còn rất nhỏ nhưng tốc độ đã vượt ngưỡng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Liveness (`/health`) trả lời câu hỏi "process còn sống không", nếu fail thì orchestrator restart container. Readiness (`/ready`) trả lời câu hỏi "sẵn sàng nhận request chưa", nếu fail thì load balancer chỉ ngừng gửi request vào, không restart.

Nếu gộp chung, chuỗi sự kiện như sau:

1. Redis mất kết nối.
2. Cả 3 container cùng báo lỗi health.
3. Orchestrator restart cả 3, lúc đó không còn container nào phục vụ.
4. Khi Redis quay lại, các container vẫn đang khởi động, nên sự cố nhỏ (một phụ thuộc chập chờn) biến thành sự cố toàn hệ thống.

Nếu tách riêng thì `/ready` trả 503 khi Redis mất. Load balancer chỉ ngừng gửi request vào, không container nào bị restart. Redis quay lại là `/ready` trả 200 và phục vụ được ngay.

**Reference:** https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Ứng dụng stateless không giữ trạng thái trong bộ nhớ của process. Mọi dữ liệu cần dùng lại giữa các request đều nằm ở dịch vụ bên ngoài như Redis.

Thí nghiệm gửi 6 lượt chat liên tiếp, 3 container chạy sau load balancer, mỗi lượt thêm 2 tin nhắn (hỏi và đáp):

| Lượt | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|
| Dùng Redis | 0 | 2 | 4 | 6 | 8 | 10 |
| Dùng dict trong process | 0 | 0 | 2 | 0 | 2 | 4 |

Với dict, load balancer chia 6 lượt lần lượt cho agent-2, agent-3, agent-2, agent-1, agent-3, agent-2. Mỗi container chỉ nhớ những lượt nó tự xử lý nên con số nhảy lung tung. Với Redis, cả 3 container cùng đọc một nguồn nên `history_length` tăng đều, và restart hay scale container không làm mất hội thoại.

**Reference:** https://12factor.net/processes

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi đã gặp trên Railway, theo 3 phần:

- **Lỗi là gì:** dashboard báo Online và `/health` trả 200, nhưng `/ready` và `/ask` trả 500.
- **Tìm ra bằng cách nào:** so sánh `/health` với `/ready` (một cái chỉ kiểm tra process còn sống, một cái kiểm tra phụ thuộc), rồi mở tab Variables của service agent. Ở đó thấy thiếu `AGENT_API_KEY`, còn `REDIS_URL` đang tham chiếu tới `DATABASE_URL` thay vì `REDIS_URL` của service Redis.
- **Sửa thế nào:** thêm biến trong tab Variables, đổi tham chiếu sang `${{day12-redis.REDIS_URL}}`, redeploy, rồi gọi lại `/ready` để xác nhận trả 200.

Bài học: `/health` trả 200 chỉ chứng minh process còn sống, không chứng minh app dùng được. Vì vậy cần `/ready` kiểm tra cả cấu hình lẫn kết nối tới Redis.
