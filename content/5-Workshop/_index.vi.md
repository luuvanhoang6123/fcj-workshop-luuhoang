---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Xây dựng và triển khai hệ thống Live Auction trên nền tảng AWS

#### Tổng quan

**Live Auction** là nền tảng đấu giá trực tuyến cho phép người dùng đăng ký tài khoản, theo dõi các phiên đấu giá và thực hiện đặt giá theo thời gian thực.

Hệ thống cung cấp hai giao diện riêng biệt dành cho người dùng và quản trị viên. Người dùng có thể quản lý thông tin cá nhân, tạo phiên đấu giá, thêm một hoặc nhiều vật phẩm, theo dõi phiên và tham gia đặt giá. Quản trị viên có thể quản lý tài khoản người dùng, quản lý danh mục sản phẩm, duyệt phiên đấu giá và tạo thêm tài khoản quản trị viên.

Trong Workshop này, nhóm trình bày quá trình xây dựng và triển khai hệ thống **Live Auction** trên nền tảng **Amazon Web Services (AWS)**. Hạ tầng được triển khai bằng **Terraform**, giúp tự động hóa việc tạo và quản lý tài nguyên AWS, đồng thời bảo đảm tính nhất quán giữa các lần triển khai.

User Frontend và Admin Frontend được lưu trữ riêng biệt trên **Amazon S3** và phân phối thông qua **Amazon CloudFront**. Chức năng xác thực và phân quyền tài khoản sử dụng **Amazon Cognito** kết hợp với **AWS IAM**. Các API nghiệp vụ được triển khai bằng **AWS Lambda** và **Amazon API Gateway**.

Đối với chức năng đấu giá theo thời gian thực, hệ thống sử dụng **API Gateway WebSocket**, **Amazon SQS FIFO** và **Amazon DynamoDB** để tiếp nhận yêu cầu đặt giá, xử lý thông điệp theo đúng thứ tự, lưu trạng thái đấu giá và gửi dữ liệu cập nhật đến những người dùng đang kết nối.

Workshop tập trung vào quá trình chuẩn bị môi trường, triển khai hạ tầng bằng Terraform, kiểm tra các dịch vụ AWS, kiểm thử hệ thống, hướng dẫn dọn dẹp tài nguyên và tổng kết kết quả đạt được.

#### Nội dung

1. [Giới thiệu đồ án và kiến trúc triển khai](5.1-Workshop-overview/)
2. [Chuẩn bị môi trường](5.2-Preparation/)
3. [Triển khai hạ tầng bằng Terraform](5.3-Infrastructure/)
4. [Các dịch vụ AWS được triển khai](5.4-AWS-Services/)
5. [Kiểm thử hệ thống](5.5-Testing/)
6. [Dọn dẹp tài nguyên](5.6-Cleanup/)
7. [Kết quả đạt được](5.7-Results/)