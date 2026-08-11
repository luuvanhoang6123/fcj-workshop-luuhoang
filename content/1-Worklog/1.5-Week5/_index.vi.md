---
title: "Worklog Tuần 5"
date: 2026-07-20
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Thời gian

**20/07/2026 – 24/07/2026**

### Mục tiêu cá nhân

- Phát triển backend serverless: Lambda handlers và schema DynamoDB.
- Triển khai logic đặt giá, lưu lịch sử bid và validate dữ liệu đầu vào.
- Phối hợp tích hợp API với frontend do thành viên khác xây dựng.

### Công việc đã thực hiện

| Thứ | Ngày | Nội dung | Tài liệu |
| --- | --- | --- | --- |
| Thứ Hai | 20/07/2026 | Thống nhất cấu trúc repo (`Infrastructure/`, `backend/`, `frontend/`); em nhận module auction và bid. | Tài liệu nhóm |
| Thứ Ba | 21/07/2026 | Thiết kế bảng DynamoDB: Users, Categories, Sessions, Items, Bids; chọn partition/sort key phù hợp truy vấn. | [DynamoDB Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) |
| Thứ Tư | 22/07/2026 | Viết Lambda REST cho CRUD phiên đấu giá và vật phẩm; thêm validation giá khởi điểm, thời gian kết thúc. | [FastAPI Tutorial](https://fastapi.tiangolo.com/tutorial/) |
| Thứ Năm | 23/07/2026 | Implement handler nhận bid, đẩy message vào SQS FIFO; draft consumer Lambda cập nhật giá cao nhất. | Mã nguồn nhóm |
| Thứ Sáu | 24/07/2026 | Test Postman với frontend staging; sửa lỗi format response; liệt kê API còn thiếu cho tuần 6. | Tài liệu test nội bộ |

### Kết quả và ghi chú

- Hoàn thành skeleton Lambda cho quản lý phiên, vật phẩm và tiếp nhận bid.
- Schema DynamoDB hỗ trợ truy vấn theo `sessionId` và lịch sử bid theo thời gian.
- Luồng bid local chạy end-to-end trước khi deploy AWS (mock queue).
- **Khó khăn:** Xử lý điều kiện bid phải cao hơn giá hiện tại — em dùng conditional write DynamoDB để tránh ghi đè sai.
- **Phối hợp nhóm:** Thành viên frontend tích hợp API list/detail; em hỗ trợ fix CORS sơ bộ trên API Gateway mock.
