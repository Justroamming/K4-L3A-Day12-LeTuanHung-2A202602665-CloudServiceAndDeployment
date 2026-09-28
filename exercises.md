# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Tuấn Hưng  Mã học viên: 2A202602665

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định "changeme", app vẫn có thể khởi động bình thường. Vấn đề là khi deploy lên production mà quên set AGENT_API_KEY, attacker có thể biết được key mặc định này và dùng nó để gọi API miễn phí. Điều này có thể khiến mình tốn tiền LLM, thậm chí bypass được một số rate limit hoặc cost guard nếu hệ thống đang dùng chung key đó.

Với fail fast, container sẽ crash ngay từ lúc start nếu thiếu cấu hình bắt buộc. Như vậy CI/CD sẽ dừng lại và log cũng báo lỗi rõ ràng, giúp mình phát hiện và sửa vấn đề trước khi app thực sự chạy trên production.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ một structured log trông như sau:

{"event":"ask_completed","level":"info","timestamp":"2026-09-28T10:19:23.123456+00:00","user_id":"sv-test","tokens_in":3,"tokens_out":35,"cost_usd":2.145e-05}

So với việc chỉ dùng print("đã trả lời xong"), structured log có hai lợi ích rất rõ:

Dễ query và filter: có thể lọc theo từng field, ví dụ tìm tất cả log có cost_usd rồi tính tổng chi phí.
Dễ tạo alert: các hệ thống có thể tự động cảnh báo khi cost_usd > 0.01 hoặc khi level="error", từ đó báo cho on-call.

Nói đơn giản, print chủ yếu dành cho người đọc, còn structured log được thiết kế để cho cả người và hệ thống đều xử lý được.

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

> Bản	Dung lượng
1 stage (bản đầu)	~1.2 GB
Multi-stage	    ~200 MB

Image giảm khoảng 1 GB vì multi-stage build không mang toàn bộ môi trường build sang image cuối.

Ở stage build, mình có thể cần những thứ như build-essential, gcc, pip cache, source code tạm thời và Python header files để compile dependency. Nhưng sau khi build xong thì runtime không cần những thứ này nữa.

Vì vậy, multi-stage build chỉ giữ lại những gì thực sự cần để chạy app, giúp image nhỏ hơn và giảm attack surface.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, nếu chỉ sửa app/main.py thì layer COPY app ./app sẽ bị invalidated. Docker chỉ cần build lại layer này và những layer nằm sau nó.

Trong khi đó, layer RUN pip install vẫn được lấy từ cache vì requirements.txt không thay đổi.

Nếu mình đặt COPY . . trước RUN pip install, tình hình sẽ khác. Mỗi lần sửa dù chỉ một ký tự trong source code, layer COPY . . cũng bị invalidated. Khi đó Docker phải chạy lại RUN pip install, khiến mỗi lần build có thể mất 2–3 phút thay vì chỉ vài giây.

Vì vậy, việc copy requirements.txt và install dependency trước, rồi mới copy source code, giúp tận dụng Docker cache tốt hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu container chạy bằng root thì khi app có lỗ hổng, hậu quả có thể nghiêm trọng.

Ví dụ:

Code có một lỗ hổng, chẳng hạn eval(user_input) hoặc path traversal.
Attacker khai thác lỗ hổng và thực thi được command trong container.
Vì container đang chạy bằng root nên process đó có UID 0 và có nhiều quyền hơn cần thiết.
Trong một số tình huống nguy hiểm như container có volume quan trọng hoặc có kernel/container escape vulnerability, attacker có thể gây ảnh hưởng tới host.

Khi dùng USER appuser với UID 10001, process chỉ chạy với quyền của user thường. Điều này không loại bỏ hoàn toàn nguy cơ, nhưng nó giảm đáng kể quyền mà attacker có được nếu app bị compromise.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Rate limit được tính theo phút trên đồng hồ, có thể xảy ra tình huống user gửi:

10 request lúc 10:00:59.900
10 request lúc 10:01:00.100

Như vậy user gửi tổng cộng 20 request chỉ trong 0.2 giây, nhưng hệ thống vẫn coi chúng thuộc hai phút khác nhau nên không chặn.

Sliding window 60 giây giải quyết vấn đề này bằng cách nhìn vào đúng 60 giây gần nhất.

Ví dụ tại thời điểm 10:01:00.100, window là:

[10:00:00.100, 10:01:00.100]

10 request ở thời điểm 10:00:59.900 vẫn nằm trong window, nên user đã đạt limit và request tiếp theo sẽ bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Hai cơ chế này bảo vệ hệ thống khỏi hai loại vấn đề khác nhau:


	Rate limit: Giới hạn số request, đơn vị request/phút	Cost guard: Giới hạn số tiền, đơn vị $/tháng

Ví dụ rate limit cho qua nhưng cost guard chặn:

User chỉ gửi 5 request/phút, trong khi limit là 10 request/phút nên rate limit vẫn cho phép. Nhưng nếu mỗi request sử dụng 100k token và tốn $0.50, thì 5 request đã tốn $2.50. Nếu budget tháng chỉ là $1, cost guard sẽ chặn request với HTTP 402.

Ngược lại, user có thể gửi 15 request/phút, nhưng mỗi request chỉ tốn khoảng 10 token. Khi đó rate limit sẽ chặn ở request thứ 11 với 429, dù tổng chi phí gần như bằng 0.


---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp cả hai endpoint và để /health kiểm tra Redis thì xảy ra một vấn đề.

Giả sử Redis mất kết nối trong 30 giây:

Cả 3 container đều trả 503 từ /health.
Orchestrator như Kubernetes, Docker Swarm hoặc Render hiểu rằng container đang unhealthy.
Nó restart cả 3 container cùng lúc.
Trong lúc Redis vừa hồi phục nhưng các container vẫn đang restart, service có thể không còn instance nào để phục vụ request.

Đây có thể tạo thành một dạng cascading failure.

Tách hai endpoint sẽ an toàn hơn:

/health chỉ kiểm tra process còn sống → vẫn trả 200, nên orchestrator không restart container.
/ready kiểm tra xem app đã sẵn sàng nhận traffic chưa → Redis down thì trả 503, khiến Load Balancer tạm thời không gửi request vào container đó.

Như vậy, Redis có vấn đề thì app không bị restart vô ích, mà chỉ tạm thời ngừng nhận traffic cho đến khi dependency hoạt động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với cách làm hiện tại, state được lưu trong Redis không phụ thuộc vào container nào đang xử lý request.

Ví dụ history_length có thể tăng:

0 → 2 → 4 → 6...

dù request lần lượt đi vào container A, B hay C, vì cả ba container đều đọc cùng một state trong Redis.

Nếu thay Redis bằng một dict Python trong RAM, mỗi container sẽ có một dict riêng.

Ví dụ:

Request 1 vào A → history_length = 0
Request 2 vào B → B cũng thấy history_length = 0
Request 3 quay lại A → A thấy history_length = 2

Khi đó state phụ thuộc vào request được load vào container nào, nên user có thể cảm giác agent lúc nhớ lúc quên một cách ngẫu nhiên.

Đó là tại sao application nên stateless, còn shared state thì lưu ở một hệ thống bên ngoài như Redis.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi khi Render start container là:

ModuleNotFoundError: No module named 'utils'

Điểm đáng chú ý là build vẫn thành công, nhưng đến lúc runtime thì import mới fail.

Kiểm tra Dockerfile thì thấy runtime stage chỉ có:

COPY app ./app

Trong khi app/main.py lại import module từ utils/, cụ thể là mock_llm.py ở app/main.py:23.

Vì vậy source code cần thiết đã không được copy vào runtime image.

Cách sửa là thêm:

COPY utils ./utils

vào runtime stage.

Sau đó push code lên, Render rebuild image và deploy lại. Khi container start thành công, endpoint /health sẽ trả 200.
