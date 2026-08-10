# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần giữ chỗ bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Minh Hạnh  Mã học viên: 2A202601232

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ, khi deploy lên Railway tôi quên khai báo `API_TOKEN`. Nếu code có
> token mặc định `changeme`, container vẫn healthy và public URL vẫn mở, nên
> người lạ có thể đoán token và gọi `/chat` trước khi tôi phát hiện. Việc
> `api_token` không có mặc định làm Pydantic báo lỗi ngay lúc startup; deployment
> chuyển sang failed thay vì chạy ở trạng thái không an toàn. Tôi thà bị lỗi rõ
> ràng khi deploy còn hơn nhận hóa đơn do một secret mặc định.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được khi chạy ba replica là:
>
> ```json
> {"event":"chat_completed","severity":"INFO","ts":"2026-08-10T09:06:29.043467+00:00","client_id":"cp4-shared-history","prompt_tokens":4,"completion_tokens":38,"usd_cost":0.0000234}
> ```
>
> Từ log JSON này, hệ thống log cloud có thể lọc theo `client_id` để truy vết
> toàn bộ request của một client, và có thể cộng `usd_cost` hoặc tạo cảnh
> báo khi chi phí vượt ngưỡng. `print("đã trả lời xong")` không có field cố
> định, timestamp, token usage hay chi phí, nên máy không thể lọc và tổng hợp
> tin cậy.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 309 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại Dockerfile gốc thành `chat:single` và xem bằng `docker images`.
> Bản single-stage dùng image `python:3.11` đầy đủ, mang theo các gói hệ thống,
> công cụ build và cache không cần cho runtime. Nó cũng `COPY . .` trước khi cài
> dependency và cài bằng root trong cùng stage. Bản multi-stage dùng
> `python:3.11-slim`; builder cài dependency vào `/opt/venv`, còn runtime chỉ copy
> virtualenv, `app` và `utils`. Vì vậy image nộp bài giảm khoảng 1.42 GB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, sửa một ký tự trong `app/main.py` không làm thay đổi
> layer `COPY requirements.txt` và `RUN pip install`, nên các layer base image, tạo
> virtualenv và cài dependency được lấy từ cache. Docker chỉ chạy lại `COPY app
> ./app`, các layer source phía sau và layer `chown`. Nếu đặt `COPY . .` trước
> `RUN pip install`, mọi thay đổi source sẽ làm hash của layer copy thay đổi; tất
> cả layer sau nó, bao gồm cài dependency, phải chạy lại dù
> `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: lỗ hổng Python cho phép remote code execution, kẻ tấn công
> chiếm process trong container, process đó đang là root nên có toàn quyền trong
> container, sau đó kẻ tấn công lợi dụng container escape, kernel bug, socket
> Docker bị mount hoặc volume/quyền runtime cấu hình sai để tác động lên host.
> Root trong container không tự động là root trên host, nhưng làm hậu quả của
> bước escape nghiêm trọng hơn. Lệnh `USER app` cắt chuỗi ngay sau khi chiếm
> process: mã tấn công chỉ có UID quyền hạn chế, không thể ghi file hệ thống
> hay thực hiện nhiều thao tác đòi quyền root trong container.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> HTTP 401 nghĩa là request chưa có credential hợp lệ, nên
> `WWW-Authenticate: Bearer` cho client biết cơ chế xác thực cần dùng theo RFC
> 6750. Nếu thiếu header này, client hoặc thư viện HTTP không biết nên cung cấp
> credential kiểu gì. Tôi trả cùng thông báo `invalid or missing bearer token`
> cho thiếu header, sai scheme và sai token để không cho người dò biết họ đã
> đoán đúng scheme hay một phần credential. Phía server vẫn có thể ghi lý do
> chi tiết vào log nội bộ nếu cần chẩn đoán.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Client mới hoặc client im lặng 10 phút chỉ có tối đa `capacity=10`, nên
> gửi liên tiếp được 10 request; request thứ 11 bị 429. Tôi cũng quan sát
> đúng chuỗi `200` mười lần rồi `429` trên Railway. Nếu bỏ
> `min(capacity, ...)`, sau 10 phút bucket nạp thêm 100 token. Nếu trước đó
> bucket đầy 10 token thì client có thể có 110 token và bắn 110 request,
> làm mất ý nghĩa giới hạn burst của `capacity`.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức 30 USD/tháng, sự cố từ 2 giờ sáng có thể tiêu tối đa
> phần ngân sách tháng còn lại, tối đa 30 USD nếu tháng chưa tiêu gì.
> Sau khi chạm trần, client có thể bị khóa tới đầu tháng sau, nên một sự
> cố có thể làm gián đoạn nhiều ngày. Với 1 USD/ngày, thiệt hại của
> ngày sự cố bị chặn quanh 1 USD và key chi phí chuyển sang ngày UTC mới,
> nên service tự có ngân sách lại vào 00:00 UTC hôm sau. Cần lưu ý guard
> kiểm tra trước request nên chi phí thực có thể vượt rất nhỏ trần bởi
> request cuối cùng.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp liveness và readiness thành một endpoint có ping Redis, trình tự khi
> Redis mất kết nối 30 giây là: cả ba container ping thất bại; probe chung
> trả 503; orchestrator coi cả ba process là hỏng và restart chúng; container mới
> lên vẫn không ping được Redis nên lại fail và bị restart, tạo restart loop
> cho toàn cụm. Khi Redis hồi phục, app còn phải chờ các container khởi động
> lại. Tách probe thì `/healthz` vẫn 200 vì process còn sống, còn `/readyz`
> trả 503 để load balancer tạm ngừng traffic. Khi Redis trở lại, `/readyz` tự
> chuyển 200 mà không cần restart ba process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thật tôi gặp trên Railway là deployment lặp lại thông báo
> `Invalid value for '--port': '$PORT' is not a valid integer`. Tôi mở Deploy Logs và
> thấy Uvicorn nhận nguyên chuỗi `$PORT`, trong khi Docker chạy local bình thường.
> Nguyên nhân là `startCommand` trong `railway.toml` override Docker CMD, nhưng
> Railway không chạy chuỗi này qua shell nên không nội suy `$PORT`. Tôi xóa
> `startCommand` để dùng Docker CMD `sh -c "... --port ${PORT:-8000}"`, set
> service và domain cùng port 8000 rồi redeploy. Deployment sau đó `SUCCESS`;
> public `/healthz` và `/readyz` đều trả 200, `/chat` không token trả 401 và
> token hợp lệ trả 200.
