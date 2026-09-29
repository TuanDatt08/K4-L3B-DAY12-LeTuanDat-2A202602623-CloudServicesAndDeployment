# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.> Cách trả lời: thay dòng placeholder dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Tuấn Đạt  Mã học viên: 2A202602623

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ mình deploy lên Render mà quên nhập `AGENT_API_KEY` trong dashboard.
> Nếu code để mặc định `"changeme"` thì service vẫn lên Live, `/health` vẫn 200,
> dashboard xanh hết nên mình tưởng ổn. Nhưng khóa `"changeme"` ai cũng đoán
> được (nó còn nằm công khai trong repo), nên bất kỳ ai tìm thấy URL đều gọi
> `/ask` thoải mái — mỗi request là tiền LLM của mình, và mình chỉ biết khi
> nhìn hóa đơn.
>
> Vì không có mặc định, thiếu biến là `Settings()` ném `ValidationError` ngay lúc
> khởi động (test `test_thieu_api_key_thi_fail_fast` kiểm tra đúng điều này).
> Trên cloud, deploy sẽ fail và log ghi rõ `agent_api_key Field required` — lỗi
> hiện ra đúng lúc mình đang nhìn màn hình deploy, sửa mất 1 phút, thay vì lộ ra
> sau vài ngày dưới dạng hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật khi gọi `/ask` lần thứ hai với user `sv01`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:48:09.058612+00:00", "user_id": "sv01", "tokens_in": 48, "tokens_out": 52, "cost_usd": 3.84e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc và cộng theo trường:** lọc `event == "ask_completed"` rồi nhóm theo
>    `user_id`, cộng `cost_usd` → biết user nào tiêu nhiều tiền nhất hôm nay.
>    Chuỗi text tự do không có `user_id` hay `cost_usd` để mà cộng.
> 2. **Đếm và cảnh báo theo thời gian:** vì có `timestamp` và `level`, hệ thống
>    log trên cloud đếm được số dòng `level == "error"` trong 5 phút gần nhất và
>    bắn cảnh báo khi vượt ngưỡng. Mỗi log nằm gọn trên một dòng nên không bị
>    cắt thành nhiều mảnh khi platform gom log theo dòng.

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
| 1 stage (bản đầu, `python:3.11`, file `Dockerfile.single`) | 1.73 GB (~1770 MB) |
| Multi-stage (`python:3.11-slim`) | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật trên máy mình: bản 1 stage **1.73 GB**, bản multi-stage
> **271 MB** — nhỏ hơn khoảng 6,5 lần, chênh gần 1,5 GB. Phần chênh lệch gồm:
>
> - **Base image:** `python:3.11` bản đầy đủ dựa trên Debian đầy đủ, kèm sẵn
>   gcc, build-essential, header file, git và rất nhiều thư viện hệ thống để
>   *biên dịch* package. Bản `-slim` chỉ giữ phần tối thiểu để *chạy* Python.
>   Đây là phần lớn nhất (~1 GB).
> - **Rác lúc cài đặt:** bản 1 stage chạy `pip install` không có
>   `--no-cache-dir` nên cache của pip nằm lại trong image.
> - **File không cần thiết:** `COPY . .` chép luôn `tests/`, file `.md`,
>   `.pytest_cache`… vào image. Bản multi-stage chỉ copy `app/`, `utils/` và
>   thư mục `/install` đã cài xong từ stage `builder`; bản thân stage builder
>   bị bỏ đi, không nằm trong image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thêm một dòng vào `app/main.py` rồi `docker build` lại, cả lần build mất
> khoảng **3 giây**:
>
> - **Dùng lại cache (`CACHED`):** `WORKDIR`, `COPY requirements.txt`,
>   `RUN pip install ...`, `COPY --from=builder /install`, `COPY utils`.
> - **Chạy lại:** `COPY app ./app` (vì nội dung `app/` đổi) và mọi lệnh đứng sau
>   nó — ở Dockerfile của mình là `RUN useradd ...`.
>
> Bước tốn thời gian nhất là `pip install` vẫn được cache vì `requirements.txt`
> không đổi. (Mình cũng nhận ra `useradd` nên đặt *trước* `COPY app` để khỏi
> phải chạy lại mỗi lần sửa code.)
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: sửa bất kỳ file nào (kể cả một
> dấu phẩy trong `main.py`) cũng làm layer `COPY . .` đổi, Docker hủy cache từ
> đó trở xuống nên phải cài lại toàn bộ thư viện — mỗi lần build mất vài phút
> thay vì vài giây, deploy lên cloud cũng chậm theo.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy bằng root:
>
> 1. Code Python có lỗ hổng (ví dụ một thư viện bị lỗi cho phép thực thi lệnh,
>    hoặc lỡ đưa input của user vào `subprocess`) → kẻ tấn công chạy được lệnh
>    shell bên trong container.
> 2. Process đó là **root (uid 0)** trong container → đọc/ghi được mọi file,
>    cài thêm công cụ, sửa code của app, đọc biến môi trường chứa secret.
> 3. uid 0 trong container cũng là uid 0 trên kernel của host. Chỉ cần thêm một
>    cấu hình sai (mount volume từ host, mount `docker.sock`, chạy
>    `--privileged`) hoặc một lỗ hổng container escape là kẻ tấn công bước ra
>    host **với quyền root** → chiếm toàn bộ máy.
>
> Lệnh `USER appuser` cắt chuỗi ở **bước 2**. Mình kiểm tra bằng
> `docker compose exec agent whoami` → `appuser` (uid 10001). Kẻ tấn công vẫn
> vào được app nhưng chỉ là user thường: không cài được gói, không ghi được thư
> mục hệ thống, và nếu có thoát ra host thì cũng chỉ là một uid không có quyền
> gì, thay vì root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**.
>
> Cách đạt được: gửi 10 request lúc 10:00:59 — vẫn trong hạn mức của phút
> 10:00. Đến 10:01:00 bộ đếm reset về 0, gửi tiếp 10 request lúc 10:01:00–01.
> Cả 20 request đều "đúng luật" vì mỗi phút đồng hồ chỉ có 10, nhưng thực tế là
> gấp đôi hạn mức trong 2 giây.
>
> Với sliding window, lúc 10:01:01 limiter nhìn lại 60 giây trước đó (từ
> 10:00:01) và thấy đã có 10 request → chặn 429 ngay. Mình kiểm tra trên bản
> deploy: gọi 15 lần liên tiếp nhận được `200` × 10 rồi `429` × 5.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau:** rate limit giới hạn **số request trong khoảng thời gian ngắn**
> (10 request/60 giây) — chống spam, chống một user chiếm hết tài nguyên. Cost
> guard giới hạn **tổng số tiền trong cả tháng** ($10/user/tháng) — chống cháy
> ngân sách. Một cái đếm lượt, một cái đếm tiền.
>
> - **Rate limit cho qua nhưng cost guard chặn:** user gửi đều đặn 5
>   request/phút (dưới hạn mức), nhưng mỗi request là một prompt rất dài, tốn
>   nhiều token. Sau vài ngày tổng chi phí vượt $10 → cost guard trả **402**
>   dù user chưa bao giờ gửi nhanh.
> - **Cost guard cho qua nhưng rate limit chặn:** một script gửi 50 câu "hi"
>   trong 10 giây. Mỗi câu chỉ tốn vài phần triệu đô, ngân sách còn gần như
>   nguyên, nhưng từ request thứ 11 rate limit đã trả **429**.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối (ví dụ đang restart).
> 2. Cả 3 container cùng gọi Redis trong health check → cả 3 cùng trả 503.
> 3. Orchestrator thấy liveness fail vài lần liên tiếp → kết luận container
>    "chết" → **restart cả 3 container** cùng lúc.
> 4. Trong lúc restart không còn instance nào phục vụ → mọi user nhận lỗi 502,
>    kể cả những request không cần Redis; request đang xử lý dở bị cắt ngang.
> 5. Redis quay lại sau 30 giây nhưng các container vẫn đang khởi động lại
>    (free tier như Render có thể mất cả phút) → sự cố 30 giây của Redis biến
>    thành sự cố lâu hơn của toàn hệ thống.
>
> Khi tách ra: `/health` (liveness) không chạm Redis nên vẫn 200, container
> **không bị restart**; `/ready` trả 503 nên load balancer tạm **ngừng gửi
> request** vào. Redis quay lại → `/ready` về 200 → nhận traffic tiếp, không
> container nào phải khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy 3 container agent (`--scale agent=3`) và gọi `/ask` với cùng
> `X-User-Id: sv-scale`, cố ý gọi lần lượt vào từng container khác nhau. Kết quả
> thật:
>
> | Lượt | Container | `history_length` |
> |---|---|---|
> | 1 | agent-1 | 0 |
> | 2 | agent-2 | 2 |
> | 3 | agent-3 | 4 |
> | 4 | agent-1 | 6 |
> | 5 | agent-2 | 8 |
>
> Con số tăng đều 2 mỗi lượt dù request rơi vào container khác, vì cả 3 cùng đọc
> và ghi một list `history:sv-scale` trong Redis.
>
> Nếu lưu trong dict Python: mỗi container có dict riêng trong RAM, nên con số
> nhảy lung tung theo container nhận request — lượt 1 (agent-1) → 0, lượt 2
> (agent-2) → 0, lượt 3 (agent-3) → 0, lượt 4 quay lại agent-1 mới thấy 2. Agent
> "mất trí nhớ" ngẫu nhiên, và container restart là mất sạch lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** khi chạy `docker compose up` ở máy để kiểm tra image trước khi đẩy
> lên Render, Docker không kéo được image Redis:
>
> ```
> failed to resolve reference "docker.io/library/redis:7-alpine": failed to
> authorize: failed to fetch oauth token ... lookup auth.docker.io:
> getaddrinfow: This is usually a temporary error during hostname resolution
> ```
>
> Trước đó còn gặp `'docker' is not recognized as an internal or external
> command` ngay sau khi cài Docker Desktop.
>
> **Tìm nguyên nhân:** đọc kỹ thông báo — lỗi nằm ở bước `lookup` (phân giải tên
> miền), không phải Dockerfile hay code, vì `docker build` dùng cache vẫn chạy
> được. Thử `docker pull redis:7-alpine` lại thì qua được bước xác thực nhưng
> lỗi DNS ở bước tải layer → mạng/DNS lúc đó chập chờn. Còn lỗi
> `'docker' is not recognized` là do terminal/VS Code được mở trước khi cài
> Docker nên vẫn giữ biến PATH cũ.
>
> **Sửa:** đợi mạng ổn định rồi chạy lại (có thể đặt DNS `8.8.8.8` trong
> Docker Engine settings); tắt hẳn VS Code mở lại để nhận PATH mới. Sau đó
> `docker compose up -d` chạy được, `/ready` trả `redis: true`. Nhờ đã kiểm tra
> image chạy đúng ở máy, lần deploy lên Render qua Blueprint thành công ngay:
> `/health` 200, `/ready` 200 (nối Key Value `day12-redis`), `/ask` không có key
> trả 401.

---