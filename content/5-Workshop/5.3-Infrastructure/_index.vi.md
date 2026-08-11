---
title: "Triển khai hạ tầng bằng Terraform"
date: 2026-07-13
weight: 3
chapter: false
pre: "<b>5.3. </b>"
---

# Triển khai hạ tầng bằng Terraform

## Giới thiều

Sau khi hoàn tất việc chuẩn bị môi trường, nhóm tiến hành triển khai hạ tầng của hệ thống **Live Auction** trên nền tảng **Amazon Web Services (AWS)** bằng **Terraform**.

Terraform được sử dụng theo mô hình **Infrastructure as Code (IaC)**, cho phép định nghĩa toàn bộ tài nguyên AWS bằng mã nguồn thay vì cấu hình thủ công trên AWS Console. Điều này giúp quá trình triển khai trở nên tự động, nhất quán và dễ dàng quản lý khi hệ thống được mở rộng hoặc cập nhật.

Trong phần này, nhóm sẽ trình bày quá trình khởi tạo Terraform, cấu trúc thư mục Infrastructure, kiểm tra kế hoạch triển khai, tạo hạ tầng AWS và kiểm tra kết quả sau khi triển khai.

#### Nội dung

1. [Giới thiệu Terraform](5.3.1-Terraform-Overview/)
2. [Cấu trúc thư mục Infrastructure](5.3.2-Infrastructure-Structure/)
3. [Khởi tạo môi trường Terraform](5.3.3-Terraform-Init/)
4. [Kiểm tra kế hoạch triển khai](5.3.4-Terraform-Plan/)
5. [Triển khai hạ tầng](5.3.5-Terraform-Apply/)