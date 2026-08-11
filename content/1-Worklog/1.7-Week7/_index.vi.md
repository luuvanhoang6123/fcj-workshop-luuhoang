---
title: "Worklog Tuần 7"
date: 2026-08-03
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Thời gian

**03/08/2026 – 07/08/2026**

### Mục tiêu cá nhân

- Lập và thực thi test case cho luồng đấu giá thời gian thực (phần em phụ trách).
- Kiểm tra tích hợp WebSocket, SQS FIFO và DynamoDB trên môi trường AWS thật.
- Khắc phục lỗi phát sinh; rà soát IAM và cấu hình Terraform trước tổng duyệt.

### Công việc đã thực hiện

| Thứ | Ngày | Nội dung | Tài liệu |
| --- | --- | --- | --- |
| Thứ Hai | 03/08/2026 | Soạn ma trận test: auth, CRUD phiên, bid hợp lệ/không hợp lệ, reconnect WebSocket. | Tài liệu test nhóm |
| Thứ Ba | 04/08/2026 | Test Cognito: đăng ký, xác nhận email, login User vs Admin; kiểm tra token trên request API. | [Cognito User Pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html) |
| Thứ Tư | 05/08/2026 | Test REST Lambda trên AWS: tạo phiên, thêm item, gửi bid; đối chiếu dữ liệu DynamoDB. | [Lambda Testing](https://docs.aws.amazon.com/lambda/latest/dg/testing-guide.html) |
| Thứ Năm | 06/08/2026 | Mô phỏng 3 client bid liên tiếp; xác minh thứ tự SQS FIFO và broadcast qua WebSocket. | [WebSocket API](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html) |
| Thứ Sáu | 07/08/2026 | Fix CORS, env var sai tên bảng DynamoDB; chạy lại `terraform plan` sau chỉnh IAM; demo end-to-end cho nhóm. | Mã nguồn & Terraform |

### Kết quả và ghi chú

- **Pass:** Luồng đăng nhập → tham gia phiên → đặt giá → cập nhật UI real-time hoạt động ổn định với 2–3 client.
- **Pass:** Bid thấp hơn giá hiện tại bị từ chối; lịch sử bid hiển thị đúng thứ tự thời gian.
- **Đã sửa:** CORS thiếu header `Authorization`; Lambda consumer đọc sai tên biến `BIDS_TABLE`.
- **Đã sửa:** WebSocket disconnect không xóa connection ID — thêm cleanup trong handler `$disconnect`.
- Em hoàn thành phần nháp Workshop mục 5.5 (Testing) cho các scenario em đã test trực tiếp.
- **Còn lại cho tuần 8:** Kiểm thử upload ảnh S3/CloudFront và viết báo cáo tổng kết cá nhân.
