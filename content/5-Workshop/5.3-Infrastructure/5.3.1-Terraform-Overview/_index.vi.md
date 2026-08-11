---
title: "Giới thiệu Terraform"
date: 2026-07-13
weight: 1
chapter: false
pre: "<b>5.3.1. </b>"
---

## Terraform là gì?

Terraform là công cụ **Infrastructure as Code (IaC)** do HashiCorp phát triển, cho phép định nghĩa và quản lý hạ tầng thông qua các tệp cấu hình thay vì thao tác thủ công trên giao diện quản trị.

Với Terraform, toàn bộ tài nguyên của hệ thống như IAM, Amazon S3, Amazon CloudFront, Amazon Cognito, AWS Lambda, Amazon API Gateway, Amazon DynamoDB và Amazon SQS được mô tả dưới dạng mã nguồn. Khi cần triển khai, Terraform sẽ tự động tạo hoặc cập nhật các tài nguyên theo đúng cấu hình đã định nghĩa.

---

## Lợi ích của Terraform

Trong quá trình thực hiện đồ án, nhóm lựa chọn Terraform vì các ưu điểm sau:

- Triển khai hạ tầng tự động bằng mã nguồn.
- Đảm bảo tính nhất quán giữa các lần triển khai.
- Dễ dàng quản lý và theo dõi sự thay đổi của hạ tầng.
- Hỗ trợ làm việc nhóm thông qua hệ thống quản lý mã nguồn Git.
- Có thể mở rộng và tái sử dụng cấu hình cho nhiều môi trường khác nhau.

---

## Terraform trong hệ thống Live Auction

Đối với hệ thống **Live Auction**, Terraform được sử dụng để triển khai và quản lý các dịch vụ AWS phục vụ hệ thống.

Các thành phần hạ tầng được quản lý bao gồm:

- AWS Identity and Access Management (IAM)
- Amazon Cognito
- Amazon S3
- Amazon CloudFront
- AWS Lambda
- Amazon API Gateway
- API Gateway WebSocket
- Amazon DynamoDB
- Amazon SQS FIFO

Việc sử dụng Terraform giúp nhóm triển khai toàn bộ hạ tầng AWS chỉ với một số lệnh, đồng thời đảm bảo các thành viên đều sử dụng chung một cấu hình trong quá trình phát triển và triển khai.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-overview.png" alt="Terraform Overview" width="60%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.1.</b> Thư mục Infrastructure (infra) sử dụng Terraform để quản lý hạ tầng AWS.
    </figcaption>
</figure>

---

## Kết quả

Sau khi tìm hiểu Terraform và chuẩn bị các tệp cấu hình, nhóm tiến hành tổ chức mã nguồn hạ tầng trong thư mục **Infrastructure**, nội dung sẽ được trình bày ở mục tiếp theo.