---
title: "Worklog Tuần 6"
date: 2026-07-27
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Thời gian

**27/07/2026 – 31/07/2026**

### Mục tiêu cá nhân

- Triển khai hạ tầng AWS bằng Terraform (phần em phụ trách chính).
- Deploy các dịch vụ serverless và lấy output endpoint phục vụ tích hợp.
- Viết tài liệu Workshop mục 5.3 (Infrastructure) kèm screenshot và lệnh thực tế.

### Công việc đã thực hiện

| Thứ | Ngày | Nội dung | Tài liệu |
| --- | --- | --- | --- |
| Thứ Hai | 27/07/2026 | Cài Terraform 1.x, kiểm tra `terraform -version`; cấu hình AWS profile cho tài khoản nhóm. | [Install Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) |
| Thứ Ba | 28/07/2026 | Tạo cấu trúc `Infrastructure/modules/`; khai báo provider, variables, locals; tách module IAM và Cognito. | [Terraform Language](https://developer.hashicorp.com/terraform/language) |
| Thứ Tư | 29/07/2026 | Chạy `terraform init`, `validate`, `plan`; sửa lỗi trùng tên resource và thiếu dependency giữa Lambda–IAM Role. | [terraform init](https://developer.hashicorp.com/terraform/cli/commands/init) |
| Thứ Năm | 30/07/2026 | `terraform apply` deploy S3, CloudFront, DynamoDB, SQS FIFO, API Gateway, Lambda; kiểm tra trên Console. | [terraform apply](https://developer.hashicorp.com/terraform/cli/commands/apply) |
| Thứ Sáu | 31/07/2026 | Gắn biến môi trường Lambda; deploy frontend lên S3; soạn draft Workshop 5.3.1–5.3.5 từ log triển khai thực tế. | Mã nguồn & log deploy |

### Kết quả và ghi chú

- Hạ tầng chính được provision bằng Terraform; output gồm REST URL, WebSocket URL, CloudFront domain.
- Em hoàn thành bản nháp tài liệu Infrastructure cho Workshop (init → plan → apply).
- Phát hiện và sửa lỗi IAM policy thiếu quyền `sqs:SendMessage` cho Lambda bid producer.
- **Khó khăn:** CloudFront invalidate cache sau khi upload frontend — em ghi thêm bước này vào hướng dẫn deploy.
- **Phối hợp nhóm:** Thành viên khác tích hợp Cognito UI; em cung cấp endpoint và bảng mapping biến môi trường.
