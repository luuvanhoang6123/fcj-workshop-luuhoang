---
title: "Worklog Tuần 4"
date: 2026-07-13
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Thời gian

**13/07/2026 – 17/07/2026**

### Mục tiêu cá nhân

- Tham gia chọn đề tài và phân tích yêu cầu nghiệp vụ đấu giá.
- Đề xuất giải pháp xử lý bid đồng thời và cập nhật giá theo thời gian thực.
- Soạn phần kiến trúc và lộ trình triển khai trong bản Proposal.

### Công việc đã thực hiện

| Thứ | Ngày | Nội dung | Ghi chú |
| --- | --- | --- | --- |
| Thứ Hai | 13/07/2026 | Thảo luận nhóm, thống nhất đề tài **Live Auction Platform on AWS**; em phụ trách tóm tắt phạm vi MVP. | Meeting nhóm |
| Thứ Ba | 14/07/2026 | Vẽ use case cho User/Admin: đăng ký, tạo phiên, thêm vật phẩm, đặt giá, duyệt phiên. | Tài liệu nội bộ |
| Thứ Tư | 15/07/2026 | Phân tích pain point: race condition khi nhiều người bid; đề xuất SQS FIFO + WebSocket push. | [AWS Architecture](https://aws.amazon.com/architecture/) |
| Thứ Năm | 16/07/2026 | Nghiên cứu Well-Architected; ánh xạ Cognito, API Gateway, Lambda, DynamoDB, S3, CloudFront vào sơ đồ. | [Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) |
| Thứ Sáu | 17/07/2026 | Hoàn thiện sơ đồ kiến trúc; đề xuất Terraform cho IaC; em soạn mục "Luồng đặt giá" trong Proposal. | [Terraform AWS](https://developer.hashicorp.com/terraform/tutorials/aws-get-started) |

### Kết quả và ghi chú

- Em hoàn thành phần mô tả luồng đấu giá: client gửi bid → API Gateway → SQS FIFO → Lambda xử lý → DynamoDB cập nhật → WebSocket broadcast.
- Xác định hai frontend (User/Admin) và vai trò Cognito phân quyền.
- Proposal được mentor góp ý: cần làm rõ cơ chế timeout phiên đấu giá — em ghi vào backlog tuần 5.
- **Vai trò cá nhân tuần này:** Phân tích nghiệp vụ + kiến trúc luồng real-time (không trùng với phần frontend do thành viên khác phụ trách).
