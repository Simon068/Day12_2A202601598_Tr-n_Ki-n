# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Kiên  Mã học viên: 2A202601598

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định `changeme`, service vẫn lên cloud nhưng bất cứ ai đoán hoặc biết giá trị đó đều có thể gọi `/chat` và tạo chi phí. Khi `API_TOKEN` bị thiếu, Settings báo lỗi ngay lúc khởi động; dashboard deploy sẽ đỏ nên tôi phát hiện thiếu secret trước khi service nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ một log là `{"event":"chat_completed","severity":"INFO","ts":"2026-08-10T07:00:00+00:00","client_id":"sv-test","usd_cost":0.00002}`. Tôi có thể lọc riêng các request của `sv-test` và cộng/truy vấn trường `usd_cost` để theo dõi chi phí. Một câu `print` tự do không có cấu trúc trường ổn định cho hai việc này.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chưa đo được số MB thực tế vì Docker daemon trên máy chưa chạy. Multi-stage bỏ khỏi runtime compiler, cache/build artifact và dependency chỉ phục vụ build; runtime chỉ còn Python slim, thư viện đã cài, mã `app/` và `utils/`, nên nhỏ hơn đáng kể so với image một stage.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, layer copy `requirements.txt` và layer `pip install` được dùng lại từ cache; layer copy source phía sau phải chạy lại. Nếu `COPY . .` đặt trước `pip install`, thay đổi một ký tự trong source làm Docker mất cache của layer copy, khiến toàn bộ dependency phải cài lại.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng có thể cho phép thực thi lệnh trong process Python. Nếu process là root, kẻ tấn công có quyền root trong container và có thể khai thác cấu hình/mount sai để tăng ảnh hưởng tới host. `USER appuser` hạ đặc quyền của process trước khi chạy app, nên lệnh thực thi chỉ có quyền của user thường và giảm đáng kể phạm vi thiệt hại.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Header `WWW-Authenticate: Bearer` cho client biết cơ chế xác thực cần dùng, đúng quy ước của response 401. Tôi trả cùng một thông báo cho thiếu header, sai scheme và sai token để không tiết lộ cho người thử dò rằng token có đúng nhưng cách gửi sai, hay token đã gần đúng.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Client gửi được 10 request liên tiếp trước khi bị 429 vì xô mới chỉ đầy tối đa 10 token. Nếu bỏ `min(capacity, ...)`, sau 10 phút nó tích thêm 100 token và có thể gửi khoảng 100 request liên tiếp; việc đó phá vỡ giới hạn burst mà token bucket cần bảo đảm.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức tháng $30 có thể để sự cố lúc 2 giờ sáng đốt tới $30 trước khi chặn và chỉ tự hồi phục khi sang tháng mới. Hạn mức $1/ngày chặn tối đa khoảng $1 trong ngày đó, rồi tự mở lại vào ngày UTC tiếp theo; mức thiệt hại nhỏ hơn và thời gian phục hồi ngắn hơn.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint chung kiểm tra Redis, Redis mất kết nối sẽ làm cả ba container đều báo không khỏe. Orchestrator lần lượt loại chúng khỏi load balancer rồi restart; trong 30 giây không còn instance phục vụ dù process app vẫn chạy. Tách `/healthz` giúp container còn sống không bị restart, còn `/readyz` chỉ tạm ngừng nhận traffic cho đến khi Redis trở lại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Chưa thực hiện deploy cloud vì chưa có quyền truy cập tài khoản Railway/Render và Redis add-on của người nộp. Khi triển khai, lỗi tôi sẽ kiểm tra đầu tiên là health check timeout do app không đọc `$PORT`; log cho biết cổng đang listen, và cách sửa là chạy Uvicorn với `--port ${PORT:-8000}`, sau đó redeploy và gọi lại `/healthz`.
