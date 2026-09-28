# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Ho Nam  Mã học viên: 2A202602788

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: Khi deploy lên production nhưng quên truyền biến môi trường `AGENT_API_KEY`. Nếu có giá trị mặc định "changeme", ứng dụng vẫn chạy và bất kỳ ai cũng có thể dùng key "changeme" để xài API chùa (gây rò rỉ dữ liệu và tốn kém chi phí). Việc "chết sớm" giúp ta phát hiện ngay lỗi cấu hình lúc khởi động, đảm bảo app không bao giờ bị public với một key mặc định yếu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:49:03.959666+00:00", "user_id": "sv-test", "tokens_in": 4, "tokens_out": 36, "cost_usd": 2.22e-05}`
> Hai việc làm được: 
> 1. Dùng các tool như Elasticsearch/Datadog dễ dàng parse cú pháp JSON, tự động cộng tổng số tiền (cost_usd) đã tiêu theo từng `user_id`.
> 2. Đặt cảnh báo tự động nếu `tokens_out` hoặc `cost_usd` của một request tăng đột biến so với ngưỡng, do giá trị kiểu số được trích xuất thẳng thay vì phải dùng regex parse chuỗi text lộn xộn.

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
| 1 stage (bản đầu) | ~ 1.01 GB |
| Multi-stage | ~ 150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch dung lượng chủ yếu nằm ở các build tools (gcc, make), pip cache, header C++ được tải về trong quá trình cài đặt dependencies. Với multi-stage, image runtime chỉ rinh nguyên phần code và thư viện đã được biên dịch xong sang, bỏ lại hoàn toàn môi trường build nặng nề.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, layer cài đặt (từ `COPY requirements.txt .` đến `RUN pip install...`) không bị đổi nên được dùng lại từ cache. Docker chỉ chạy lại từ bước `COPY . .` trở xuống. 
> Nếu đặt `COPY . .` lên trước, bất kỳ thay đổi nào trong code (dù chỉ 1 ký tự) cũng sẽ làm invalidate layer `COPY` đó, kéo theo lệnh `RUN pip install` phía sau cũng mất cache và phải tải cài lại toàn bộ thư viện từ đầu, khiến thời gian build tăng cực kỳ đáng kể.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: Ứng dụng Python có lỗ hổng thực thi mã từ xa (RCE) -> Hacker chạy được shell trong container, do container chạy quyền root nên hacker nghiễm nhiên có quyền lực tối cao -> Từ quyền root trong container, hacker có thể tìm cách escape (thoát ra) môi trường container hoặc tương tác với các volume mount của host để chiếm trọn máy chủ.
> Lệnh `USER appuser` cắt đứt chuỗi này: hacker có thực thi RCE cũng chỉ lấy được shell của user thường (`appuser`), hoàn toàn bị vô hiệu hóa việc vọc vạch sâu vào hệ thống.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong vỏn vẹn 2 giây liên tiếp (cụ thể ở giây 59 và giây 00 kế tiếp).
> Giải thích: Người đó có thể "ém" và gửi dồn 10 request lúc 10:00:59 (thuộc hạn mức phút 00), sau đó gửi dồn tiếp 10 request lúc 10:01:00 (vừa reset sang hạn mức phút 01). Dù chỉ cách nhau 1 giây nhưng đếm tĩnh sẽ vẫn cho qua hết 20 request, gây tải tức thời lớn cho máy chủ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - Khác biệt: Rate limit chặn tần suất (số lần gọi trong thời gian ngắn là phút/giây), còn Cost Guard chặn tổng chi phí (tiêu dùng tích lũy trong khoảng thời gian dài là tháng).
> - Rate Limit cho qua nhưng Cost Guard chặn: Người dùng hỏi rất chậm (chỉ 1 request/phút) nhưng nội dung câu hỏi đẩy vào nguyên 1 triệu token, đốt sạch tiền trong tháng.
> - Rate Limit chặn nhưng Cost Guard cho qua: Bắt đầu tháng mới chưa tiêu đồng nào, nhưng user xài tool spam 100 API calls "Xin chào" trong 1 giây để quấy rối.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Mạng chập chờn khiến kết nối từ container tới Redis bị gián đoạn, hàm ping trả về False.
> 2. Probe gộp (đóng vai trò liveness) sẽ lập tức báo lỗi (status 503).
> 3. Hệ thống Orchestrator (K8s/Docker Swarm) thấy liveness báo lỗi nên tưởng container app bị treo cứng -> thẳng tay kill cả 3 container và khởi động lại chúng.
> 4. Container mới dựng lên vẫn thấy Redis tịt -> lại bị kill -> Gây ra chuỗi CrashLoop. Toàn bộ app bị sập (downtime) thay vì chỉ tạm ngừng nhận traffic chờ Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu dùng dict Python (nhớ trên RAM), `history_length` sẽ nhảy loạn xạ (ví dụ: request đầu là 0, request tiếp có thể là 0, sau đó nhảy lên 2, rồi lại về 0) do các request bị load balancer rải ngẫu nhiên (round-robin) vào 3 container độc lập, ai có trí nhớ của người nấy.
> Nhờ đưa vào Redis làm chỗ lưu trữ chung, mọi container nhìn thấy cùng một nguồn lịch sử, nên `history_length` sẽ tăng đều và chuẩn xác (0, 2, 4, 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thông báo lỗi (mô phỏng Fallback): `httpx.RemoteProtocolError: Server disconnected without sending a response` khi tool chấm gọi vào `http://localhost:8000/health`.
> Tìm nguyên nhân: Em mở terminal và gõ `docker compose ps` cũng như xem log bằng `docker compose logs agent`, phát hiện app chưa được build và run đúng cách ở port 8000, do thiếu cấu hình biến môi trường trong `.env` dẫn đến service sập hoặc chưa map port chính xác.
> Khắc phục: Xóa container cũ, chạy `cp .env.example .env`, điền `LOCAL_FALLBACK=true`, rồi tiến hành `docker compose build` và `docker compose up -d` lại để service binding đúng chuẩn 0.0.0.0:8000. Lỗi kết nối chấm dứt lập tức.
