# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

Họ và tên: Nguyen Thanh Hoa  
Mã học viên: 2A202602559

### Câu 1 — Fail fast (CP1)

Nếu deploy lên Railway mà quên đặt `AGENT_API_KEY`, ứng dụng sẽ dừng ngay khi khởi động. Nhờ vậy lỗi được phát hiện trong deployment log, thay vì service chạy với key mặc định như `changeme` và cho phép người khác gọi API trái phép.

### Câu 2 — Log cho máy đọc (CP1)

Ví dụ log:

```json
{"event":"ask_completed","level":"info","timestamp":"2026-09-28T13:42:12.968698+00:00","user_id":"sv-test","tokens_in":48,"tokens_out":52,"cost_usd":0.0000384}
```

Từ log này có thể lọc user tiêu tốn nhiều chi phí nhất và thống kê request trong một khoảng thời gian. `print()` thông thường chỉ là chuỗi văn bản nên khó lọc và phân tích tự động.

### Câu 3 — Kích thước image (CP2)

| Bản | Dung lượng |
|---|---:|
| 1 stage | ~390 MB |
| Multi-stage | 271 MB |

Multi-stage image nhỏ hơn vì stage builder đảm nhận việc tải, biên dịch bánh xe wheel và cài đặt dependencies. Stage runtime cuối cùng chỉ copy kết quả thư mục packages (`site-packages`) và mã nguồn cần thiết, loại bỏ hoàn toàn pip cache, compiler cũng như các file tạm trung gian trong quá trình build.

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Khi chỉ sửa `app/main.py`, layer copy và cài `requirements.txt` vẫn được Docker dùng lại từ cache. Các layer copy source và các layer phía sau nó phải chạy lại.

Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần sửa một dòng code cũng làm layer `COPY` thay đổi, khiến Docker phải chạy lại `pip install` và build chậm hơn.

### Câu 5 — Vì sao không chạy bằng root (CP2)

Nếu ứng dụng có lỗ hổng, kẻ tấn công có thể thực thi mã bên trong container. Nếu process chạy bằng root, mã độc có quyền cao và có thể đọc hoặc sửa nhiều file, thậm chí tìm cách ảnh hưởng host. Lệnh `USER appuser` giới hạn process ở user thường, giảm quyền của kẻ tấn công ngay cả khi ứng dụng bị khai thác.

### Câu 6 — Cửa sổ trượt (CP3)

Nếu rate limit reset theo phút đồng hồ, user có thể gửi tối đa 20 request trong 2 giây: 10 request ngay trước thời điểm chuyển phút và 10 request ngay sau khi sang phút mới. Sliding window kiểm tra đúng 60 giây gần nhất nên không cho phép trường hợp vượt quota này.

### Câu 7 — Rate limit và cost guard (CP3)

Rate limit giới hạn số lượng request, còn cost guard giới hạn tổng chi phí. User có thể gửi ít request nhưng mỗi request rất lớn, khiến rate limit cho qua nhưng cost guard chặn vì vượt ngân sách. Ngược lại, user còn ngân sách nhưng gửi quá nhiều request trong một phút thì rate limit chặn.

### Câu 8 — `/health` khác `/ready` (CP4)

Nếu `/health` cũng kiểm tra Redis, khi Redis mất kết nối thì cả ba container có thể bị đánh dấu không khỏe và bị restart, dù process agent vẫn hoạt động. Thiết kế đúng là `/health` chỉ kiểm tra process, còn `/ready` kiểm tra Redis. Khi Redis lỗi, `/ready` trả `503` để load balancer ngừng gửi traffic mới.

### Câu 9 — Stateless (CP4)

Nếu history nằm trong dict Python, mỗi container có một bản ghi nhớ riêng. Khi request chuyển từ container A sang B, `history_length` có thể quay về `0` hoặc tăng không liên tục. Khi lưu trong Redis, cả ba container cùng đọc một nguồn dữ liệu nên lịch sử nhất quán dù request đi qua instance nào.

### Câu 10 — Deploy thật (CP5)

Khi deploy, tôi gặp lỗi service không khởi động vì ứng dụng chưa đọc đúng biến `PORT` và health check không truy cập được endpoint. Tôi kiểm tra deployment log trên Railway và thấy process dùng port cố định `8000`, trong khi Railway cấp port qua biến môi trường. Tôi sửa Dockerfile và `railway.toml` để chạy Uvicorn với `${PORT:-8000}`, cấu hình health check `/health`, rồi deploy lại. Sau đó `/health`, `/ready` và `/ask` đều hoạt động.
