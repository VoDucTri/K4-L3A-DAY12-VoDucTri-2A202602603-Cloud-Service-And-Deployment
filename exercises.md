# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Võ Đức Trí  Mã học viên: 2A202602603

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi triển khai dịch vụ lên môi trường Cloud (chẳng hạn Railway, Render hoặc tạo container mới trong hệ thống CI/CD), người quản trị hoặc lập trình viên vô tình quên cấu hình biến môi trường `AGENT_API_KEY` trong bảng Environment Variables của dashboard. 
- Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công và liveness probe trả về 200 OK. Đội ngũ phát triển tưởng rằng dịch vụ đã hoạt động an toàn. Tuy nhiên, các bot quét bảo mật tự động trên Internet liên tục rà soát các endpoint công khai và thử những chuỗi khóa mặc định phổ biến như `"changeme"`, `"admin"`, `"password"`. Khi đó, kẻ tấn công dễ dàng gọi được vào endpoint `/ask` hoàn toàn miễn phí, tiêu tốn sạch token và ngân sách LLM của hệ thống, mà ta chỉ phát hiện khi nhận hóa đơn chi phí tăng đột biến.
- Nhờ cơ chế "Fail fast" (không gán giá trị mặc định), Pydantic sẽ ném ngay ngoại lệ `ValidationError` ngay trong mili-giây đầu tiên khi ứng dụng tải cấu hình. Container sẽ lập tức dừng lại (CrashLoopBackOff hoặc Deploy Failed). Việc ứng dụng "chết sớm" cảnh báo ngay cho ta biết cấu hình đang thiếu sót trầm trọng và buộc ta phải bổ sung khóa bí mật an toàn trước khi dịch vụ kịp mở cổng đón bất kỳ traffic nào từ bên ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:44:00.125634+00:00", "user_id": "sv-test", "tokens_in": 14, "tokens_out": 52, "cost_usd": 0.0000333}`

Hai việc làm được với dòng log JSON mà `print("đã trả lời xong")` không thể làm được:
1. **Lọc, tìm kiếm và truy vấn có cấu trúc (Structured Querying & Filtering):** Các hệ thống thu thập và phân tích log tập trung (như Datadog, Grafana Loki, CloudWatch, ELK Stack) có thể tự động phân tách JSON và đánh chỉ mục (index) từng trường dữ liệu. Ta có thể dễ dàng lọc ra toàn bộ request của một người dùng cụ thể (`user_id == "sv-test"`), hoặc tìm các request tiêu tốn chi phí cao bất thường (`cost_usd > 0.001`) trong một khung giờ nhất định mà không cần viết các biểu thức chính quy (regex) phức tạp và dễ vỡ.
2. **Tổng hợp số liệu và kích hoạt cảnh báo tự động (Metrics Aggregation & Alerting):** Hệ thống giám sát có thể thực hiện các phép toán thời gian thực như tính tổng chi phí (`SUM(cost_usd)`) hoặc lượng token tiêu thụ theo từng phút/giờ để vẽ đồ thị theo dõi. Đồng thời, ta có thể thiết lập cảnh báo tự động gửi thông báo về Slack hoặc email nếu tổng token hoặc chi phí của một user vượt ngưỡng định mức trong 5 phút. Lệnh `print("đã trả lời xong")` hoàn toàn không có cấu trúc dữ liệu, thiếu thông tin định danh và timestamp chuẩn để máy tính thực hiện các tác vụ tự động hóa này.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 184 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~850 MB) bao gồm:
1. **Hệ điều hành cơ sở (Base OS Footprint):** Bản 1-stage sử dụng image `python:3.11` đầy đủ (dựa trên bản phân phối Debian hoàn chỉnh), tích hợp sẵn hàng loạt thư viện hệ thống, trình biên dịch C/C++ (gcc, g++, make, build-essential), tài liệu hướng dẫn man pages, và các tiện ích dòng lệnh không dùng tới ở môi trường production. Ngược lại, bản multi-stage ở stage `runtime` sử dụng `python:3.11-slim`, chỉ giữ lại các gói nhị phân tối thiểu vừa đủ để chạy trình thông dịch Python.
2. **Công cụ biên dịch và file tạm (Build Artifacts & Cache):** Ở bản multi-stage, toàn bộ quá trình tải dependencies, giải nén bánh xe (wheels), cache tải về của pip (`/root/.cache/pip`), và các file tiêu đề (headers) chỉ diễn ra ở stage `builder`. Stage `runtime` cuối cùng chỉ copy đúng thư mục nhị phân đã biên dịch `/install` sang `/usr/local`. Toàn bộ công cụ biên dịch và cache dư thừa của stage `builder` đều bị loại bỏ hoàn toàn, không lưu lại trong image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
  - Các layer được dùng lại từ cache (`CACHED`): `FROM python:3.11-slim AS builder`, `WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install ...`, `FROM python:3.11-slim AS runtime`, `WORKDIR /app`, `COPY --from=builder /install /usr/local`, và `RUN useradd ...`. Lý do là file `requirements.txt` và các tập lệnh môi trường hoàn toàn không thay đổi hash.
  - Các layer phải chạy lại: Bắt đầu từ lệnh `COPY . .` do file `app/main.py` đã bị thay đổi một ký tự (thay đổi checksum context), kéo theo các chỉ thị tiếp theo (`USER appuser`, `EXPOSE 8000`, `HEALTHCHECK`, `CMD`) cũng được đánh giá lại. Thời gian build lại chỉ mất khoảng 1-2 giây vì không phải tải lại package.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  Mỗi khi sửa dù chỉ một ký tự trong mã nguồn `app/main.py`, hash của layer `COPY . .` sẽ bị thay đổi ngay lập tức. Theo cơ chế layer caching của Docker, khi một layer bị vô hiệu hóa cache (cache miss), toàn bộ các layer tiếp theo nằm sau nó bắt buộc phải chạy lại từ đầu. Do đó, Docker sẽ phải thực thi lại lệnh `RUN pip install -r requirements.txt`, tải về và cài đặt lại toàn bộ các thư viện Python, làm thời gian build kéo dài từ vài giây lên vài phút một cách không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện tấn công:
1. **Khai thác lỗ hổng ứng dụng (App Vulnerability):** Ứng dụng Python tồn tại một lỗ hổng thực thi mã từ xa (RCE) — ví dụ qua hàm `eval()`, việc deserialize dữ liệu không an toàn bằng module `pickle`, hoặc command injection qua `os.popen`. Kẻ tấn công gửi payload độc hại để chiếm quyền điều khiển và mở reverse shell vào bên trong container.
2. **Quyền hạn trong container:** Vì container mặc định không khai báo `USER`, tiến trình Python chạy dưới quyền tài khoản `root` (UID 0). Kẻ tấn công ngay lập tức có toàn quyền quản trị tối cao bên trong filesystem của container: chỉnh sửa cấu hình hệ thống, đọc các biến môi trường nhạy cảm, và can thiệp vào mọi socket mạng.
3. **Leo thang đặc quyền và vượt rào ra host (Container Escape):** Do UID 0 bên trong container mặc định ánh xạ trực tiếp với UID 0 trên kernel của máy host (nếu không bật user namespace remapping). Kẻ tấn công lợi dụng quyền root này để khai thác các lỗ hổng nhân Linux (như Dirty Pipe, cgroup escape) hoặc can thiệp vào các tài nguyên hệ thống được mount từ host (chẳng hạn volume mount `/var/run/docker.sock` hoặc thư mục `/proc`), từ đó thoát khỏi ranh giới container và kiểm soát máy host với quyền root.
- **Lệnh `USER appuser` cắt đứt chuỗi ở đâu:** Lệnh `USER` cắt đứt chuỗi tấn công ngay tại bước 2. Khi kẻ tấn công khai thác thành công lỗi RCE, shell thu được chỉ thuộc về người dùng thường `appuser` (UID 10001). Người dùng này không thể ghi vào thư mục hệ thống của container, bị tước bỏ hầu hết các Linux capabilities nguy hiểm (như `CAP_SYS_ADMIN`), không thể truy cập socket của Docker daemon, do đó ngăn chặn hoàn toàn khả năng thực hiện container escape để leo thang lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Số request tối đa có thể gửi trong 2 giây liên tiếp là: **20 request**.

Cách đạt được con số đó:
- Cơ chế đếm theo phút đồng hồ (fixed window) sẽ đặt lại bộ đếm về 0 vào mỗi đầu phút (giây 00).
- Giả sử hạn mức là 10 request/phút. Người dùng có thể chờ đến giây cuối cùng của phút trước, ví dụ `10:00:59`, và gửi liên tiếp **10 request**. Hệ thống ghi nhận 10/10 request cho phút 10:00, hoàn toàn hợp lệ.
- Đúng 1 giây sau, khi đồng hồ chuyển sang `10:01:00`, bộ đếm của hệ thống tự động reset về 0.
- Ngay trong giây `10:01:00` (hoặc `10:01:01`), người dùng tiếp tục bắn liên tiếp thêm **10 request** nữa. Hệ thống ghi nhận 10/10 request cho phút 10:01, vẫn được tính là hợp lệ.
- Kết quả là chỉ trong 2 giây liên tiếp (từ `10:00:59` đến `10:01:00`), người dùng đã gửi thành công tổng cộng 20 request (gấp đôi hạn mức quy định), gây ra đột biến tải có thể làm nghẽn dịch vụ.
- Trong khi đó, thuật toán sliding window 60 giây bằng Redis Sorted Set luôn tính lùi đúng 60 giây từ thời điểm request hiện tại, đảm bảo trong bất kỳ cửa sổ 60 giây liên tục nào, tổng số request không bao giờ vượt quá 10.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt cốt lõi:
- **Rate Limiting:** Giới hạn *tần suất và số lượng yêu cầu* trong một khoảng thời gian ngắn (ví dụ 10 request/phút) nhằm bảo vệ tính sẵn sàng của hạ tầng server, chống nghẽn đường truyền, DoS và brute-force.
- **Cost Guard:** Giới hạn *tổng chi phí tài chính phát sinh* (tính bằng USD hoặc số lượng token tiêu thụ) trong một chu kỳ dài (ví dụ 10.0 USD/tháng) nhằm bảo vệ ngân sách trước chi phí sử dụng API của các mô hình LLM bên thứ ba.

Hai tình huống minh họa:
1. **Rate limit cho qua nhưng Cost guard chặn:** Một người dùng chỉ gửi duy nhất 1 request trong vòng 15 phút (tần suất rất thấp, vượt qua dễ dàng ngưỡng 10 request/phút). Tuy nhiên, người này đã tích lũy chi tiêu đạt 9.99 USD trên tổng ngân sách 10.00 USD của tháng. Request mới này gửi kèm một tài liệu khổng lồ 50.000 token với chi phí ước tính 0.05 USD. Cost guard sẽ chặn ngay lập tức và trả về mã lỗi `402 Payment Required` vì tổng chi phí sẽ vượt quá ngân sách tháng.
2. **Cost guard cho qua nhưng Rate limit chặn:** Một người dùng mới toanh chưa tiêu đồng nào (ngân sách còn nguyên 10.00 USD), nhưng sử dụng script gửi tự động 15 request câu hỏi siêu ngắn (mỗi câu chỉ 5 token, chi phí cực nhỏ chỉ 0.000001 USD) trong vòng 3 giây. Về mặt ngân sách, số tiền này hoàn toàn không đáng kể và Cost guard cho phép, nhưng Rate limiter sẽ lập tức can thiệp và chặn từ request thứ 11 với mã lỗi `429 Too Many Requests` để tránh làm server bị quá tải do dồn dập request.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện diễn ra:
1. **Giây 0:** Redis gặp sự cố mạng tạm thời, quá tải hoặc đang khởi động lại trong vòng 30 giây.
2. **Giây 1 - 5:** Liveness probe (vốn bị gộp chung kiểm tra Redis) của orchestrator (Docker daemon, Kubernetes kubelet hoặc cloud platform) định kỳ gọi vào cả 3 container agent. Do không kết nối được Redis, endpoint trả về mã lỗi 503 hoặc timeout.
3. **Giây 5 - 10:** Orchestrator nhận thấy liveness probe thất bại liên tiếp vượt quá số lần retry cho phép. Nó đưa ra kết luận sai lầm rằng toàn bộ tiến trình ứng dụng của cả 3 container đều "đã chết hoặc treo" (unhealthy).
4. **Giây 10 - 20:** Orchestrator tiến hành **gửi tín hiệu hủy và khởi động lại (restart) đồng loạt cả 3 container agent**.
5. **Giây 20 - 30:** Các container mới được bật lên, nhưng Redis vẫn chưa hoàn tất 30 giây hồi phục. Khi vừa khởi động xong, probe lại tiếp tục kiểm tra và thất bại, khiến orchestrator tiếp tục khởi động lại cụm container lần nữa (rơi vào trạng thái CrashLoopBackOff / restart loop).
6. **Hậu quả:** Toàn bộ hệ thống bị sập hoàn toàn (100% downtime), tất cả các kết nối đang xử lý bị ngắt đột ngột và client nhận lỗi 502/503. Dù thực chất tiến trình Python FastAPI hoàn toàn khỏe mạnh và chỉ cần tạm dừng nhận request mới trong 30 giây chờ Redis khôi phục (nhiệm vụ đúng đắn của `/ready`), việc gộp vào `/health` đã biến một sự cố phụ thuộc tạm thời thành thảm họa sập toàn bộ cụm dịch vụ.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi chạy cụm 3 instance agent, các request gửi tới sẽ được load balancer (hoặc cơ chế định tuyến mạng của Docker/Nginx) phân phối theo thuật toán round-robin hoặc ngẫu nhiên vào 1 trong 3 container:
- Nếu lịch sử hội thoại được lưu trong Redis: Cả 3 container đều kết nối và chia sẻ chung một cơ sở dữ liệu Redis. Bất kể request rơi vào container nào, nó đều đọc và ghi cùng một key `history:<user_id>`, giúp `history_length` tăng đều đặn, tuần tiến qua các lần hỏi: `0 -> 2 -> 4 -> 6...`.
- Nếu lịch sử hội thoại được lưu trong một biến dict nội bộ trong bộ nhớ RAM của tiến trình Python:
  - Mỗi container A, B, C sẽ sở hữu một dict riêng biệt trong RAM của nó.
  - Request 1 rơi vào container A: dict của A lưu 2 tin nhắn, trả về `history_length: 0`.
  - Request 2 rơi vào container B: dict của B hoàn toàn rỗng, container B trả lời như người mới gặp và trả về `history_length: 0` thay vì 2!
  - Request 3 rơi vào container C: dict của C cũng rỗng, tiếp tục trả về `history_length: 0`.
  - Request 4 rơi lại vào container A: A nhớ được lần trao đổi đầu tiên nên trả về `history_length: 2`.
  - Kết quả là giá trị `history_length` sẽ nhảy lộn xộn, thất thường (lúc 0, lúc 2, lúc 0), và AI Agent sẽ liên tục "mất trí nhớ", quên ngữ cảnh câu hỏi vừa trao đổi tùy thuộc vào việc request bị đẩy ngẫu nhiên vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:** `Container failed to start: Port 8000 is not responding` hoặc `Application failed to respond on PORT` kèm theo việc health check timeout và service bị crash loop trên nền tảng cloud.
- **Cách tìm ra nguyên nhân:** Khi mở tab Logs / Deployment Events trên dashboard của dịch vụ cloud (như Railway hoặc Render), ta quan sát thấy hệ thống cloud tự động gán một cổng ngẫu nhiên thông qua biến môi trường `PORT` (ví dụ `PORT=10000` hoặc một cổng động khác) và bộ proxy định tuyến của cloud gửi health check vào cổng đó. Tuy nhiên, lệnh khởi chạy mặc định trong Dockerfile lại hardcode cứng cổng `--port 8000`. Do uvicorn chỉ lắng nghe trên 8000 trong khi proxy của cloud lại ping vào cổng `$PORT`, kết nối bị từ chối (Connection Refused), khiến cloud đánh giá container không khởi động được.
- **Cách sửa:** Chỉnh sửa lệnh `CMD` trong `Dockerfile` để đọc giá trị động từ biến môi trường `$PORT` do platform cấp, đồng thời giữ giá trị dự phòng 8000 khi chạy local:
  ```dockerfile
  CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
  ```
  Sau khi cập nhật cú pháp này và rebuild deploy lại, uvicorn tự động nhận cổng của platform cấp và vượt qua bước kiểm tra sức khỏe liveness probe ngay lập tức.
