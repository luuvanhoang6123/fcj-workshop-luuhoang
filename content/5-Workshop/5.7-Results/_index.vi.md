---
title: "Kết quả đạt được"
date: 2026-08-09
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

## Tổng quan

Sau quá trình phân tích, phát triển, triển khai và kiểm thử, nhóm đã hoàn thành hệ thống **Live Auction** trên nền tảng **Amazon Web Services (AWS)**.

Hệ thống được xây dựng theo kiến trúc serverless, sử dụng Terraform để quản lý hạ tầng dưới dạng mã nguồn. Các thành phần frontend, backend, xác thực tài khoản, lưu trữ dữ liệu và đấu giá theo thời gian thực đã được tích hợp thành một hệ thống hoàn chỉnh.

Kết quả kiểm thử chức năng và các hình ảnh minh chứng chi tiết đã được trình bày tại mục **5.5 — Kiểm thử hệ thống**. Vì vậy, mục này tập trung tổng kết kết quả triển khai, kiến thức đạt được, hạn chế và hướng phát triển của đồ án.

## Kết quả triển khai

Toàn bộ hạ tầng của hệ thống đã được xây dựng bằng **Terraform** và triển khai tại Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Các kết quả chính gồm:

- Hạ tầng được chia thành các Terraform module theo từng nhóm chức năng.
- Terraform State được lưu trữ và quản lý tập trung.
- Các thay đổi hạ tầng được kiểm tra bằng `terraform plan`.
- Các tài nguyên AWS được triển khai bằng `terraform apply`.
- Tên tài nguyên tuân theo quy ước của dự án.
- IAM Role và IAM Policy được cấu hình theo nguyên tắc đặc quyền tối thiểu.
- User Frontend và Admin Frontend được triển khai độc lập.
- Hệ thống có thể được cập nhật hoặc triển khai lại từ mã nguồn Terraform.
- Các thành phần giám sát, sao lưu và CI/CD được tích hợp vào hạ tầng.

Các dịch vụ AWS chính đã được triển khai:

| Dịch vụ | Kết quả đạt được |
| --- | --- |
| **AWS IAM** | Quản lý Role, Policy và quyền truy cập giữa các dịch vụ. |
| **Amazon Cognito** | Xác thực tài khoản và phân quyền User, Admin. |
| **Amazon S3** | Lưu trữ hai frontend và hình ảnh vật phẩm. |
| **Amazon CloudFront** | Phân phối nội dung frontend thông qua CDN. |
| **AWS Lambda** | Xử lý nghiệp vụ của người dùng, quản trị viên và đấu giá. |
| **Amazon API Gateway** | Cung cấp REST API cho hai giao diện frontend. |
| **API Gateway WebSocket** | Cung cấp kết nối hai chiều theo thời gian thực. |
| **Amazon DynamoDB** | Lưu dữ liệu nghiệp vụ, kết nối và trạng thái đấu giá. |
| **Amazon SQS FIFO** | Tiếp nhận và xử lý yêu cầu đặt giá theo đúng thứ tự. |
| **Amazon CloudWatch** | Theo dõi Logs, Metrics và tình trạng hoạt động. |
| **AWS Backup** | Sao lưu các bảng dữ liệu quan trọng. |

## Kết quả chức năng

Hệ thống đã cung cấp hai giao diện riêng biệt dành cho người dùng và quản trị viên.

### User Frontend

Người dùng có thể:

- Đăng ký, đăng nhập và đăng xuất.
- Xem và cập nhật thông tin cá nhân.
- Xem danh sách và thông tin chi tiết của phiên đấu giá.
- Tạo và quản lý phiên đấu giá của mình.
- Thêm một hoặc nhiều vật phẩm vào phiên.
- Tham gia phiên đấu giá đang hoạt động.
- Gửi yêu cầu đặt giá.
- Nhận giá mới và trạng thái đấu giá theo thời gian thực.

### Admin Frontend

Quản trị viên có thể:

- Xem trang tổng quan hệ thống.
- Quản lý tài khoản người dùng.
- Khóa hoặc mở khóa tài khoản.
- Kiểm duyệt phiên và vật phẩm đấu giá.
- Duyệt, từ chối hoặc hủy phiên.
- Quản lý danh mục sản phẩm.
- Theo dõi lịch sử thao tác quản trị.
- Mời và quản lý các tài khoản quản trị viên.

### Đấu giá theo thời gian thực

Luồng đấu giá theo thời gian thực được triển khai bằng:

```text
API Gateway WebSocket
AWS Lambda
Amazon SQS FIFO
Amazon DynamoDB
```

Khi người dùng đặt giá:

1. Frontend gửi yêu cầu qua WebSocket API.
2. Lambda WebSocket Handler kiểm tra yêu cầu.
3. Yêu cầu hợp lệ được đưa vào Amazon SQS FIFO.
4. Lambda Bid Processor xử lý thông điệp theo thứ tự.
5. Giá mới và lịch sử đặt giá được lưu vào DynamoDB.
6. Broadcast Lambda gửi kết quả đến các WebSocket Client.
7. Giao diện người dùng cập nhật giá mà không cần tải lại trang.

Kết quả kiểm thử chi tiết của các chức năng này được trình bày tại mục **5.5 — Kiểm thử hệ thống**.

## Kiến thức và kinh nghiệm đạt được

Thông qua quá trình thực hiện đồ án, nhóm đã tích lũy được các kiến thức và kinh nghiệm:

- Phân tích yêu cầu của hệ thống đấu giá trực tuyến.
- Thiết kế kiến trúc serverless trên AWS.
- Tổ chức Terraform theo nhiều module.
- Quản lý hạ tầng bằng Infrastructure as Code.
- Sử dụng Terraform State và Backend.
- Cấu hình IAM Role và IAM Policy.
- Xác thực và phân quyền bằng Amazon Cognito.
- Triển khai website bằng Amazon S3 và CloudFront.
- Xây dựng REST API bằng API Gateway và Lambda.
- Xây dựng chức năng thời gian thực bằng WebSocket API.
- Xử lý thông điệp có thứ tự bằng Amazon SQS FIFO.
- Thiết kế dữ liệu NoSQL trên Amazon DynamoDB.
- Sử dụng Global Secondary Index, TTL, Stream và Point-in-Time Recovery.
- Theo dõi hệ thống bằng CloudWatch Logs và Metrics.
- Sao lưu dữ liệu và chuẩn bị kế hoạch khôi phục.
- Kiểm tra và xử lý lỗi trong quá trình triển khai.
- Phối hợp làm việc nhóm và quản lý mã nguồn bằng GitHub.

## Hạn chế

Hệ thống vẫn còn một số hạn chế:

- Chưa kiểm thử tải với số lượng lớn người dùng đồng thời.
- Chưa đánh giá đầy đủ chi phí vận hành trong thời gian dài.
- Một số CloudWatch Logs chưa ghi chi tiết Route Key và Connection ID.
- Giao diện và trải nghiệm người dùng vẫn có thể tiếp tục cải thiện.
- Chưa tích hợp tên miền tùy chỉnh.
- Chưa triển khai đầy đủ cơ chế chống gian lận trong đấu giá.
- Việc kiểm thử bảo mật chuyên sâu vẫn còn hạn chế.
- Chưa đánh giá hệ thống với lưu lượng thực tế trong môi trường production.

## Hướng phát triển

Trong tương lai, hệ thống có thể được phát triển thêm:

- Tích hợp tên miền tùy chỉnh và chứng chỉ SSL.
- Hoàn thiện quy trình CI/CD cho frontend, backend và hạ tầng.
- Bổ sung thông báo qua email hoặc thiết bị di động.
- Bổ sung lịch sử giao dịch và chức năng thanh toán.
- Tăng cường phát hiện gian lận trong quá trình đặt giá.
- Tích hợp AWS WAF để bảo vệ CloudFront và API Gateway.
- Cải thiện CloudWatch Dashboard và cảnh báo tự động.
- Tích hợp AWS X-Ray để theo dõi request giữa các dịch vụ.
- Thực hiện kiểm thử tải và đánh giá khả năng mở rộng.
- Tối ưu chi phí dựa trên dữ liệu sử dụng thực tế.
- Hoàn thiện cơ chế sao lưu và khôi phục sau sự cố.
- Cải thiện giao diện trên thiết bị di động.

## Kết luận

Nhóm đã hoàn thành quá trình xây dựng và triển khai hệ thống **Live Auction** theo kiến trúc serverless trên AWS.

Terraform giúp tự động hóa quá trình triển khai và bảo đảm tính nhất quán của hạ tầng. Các dịch vụ AWS được kết hợp để đáp ứng yêu cầu lưu trữ frontend, xác thực tài khoản, phân quyền, xử lý API, lưu trữ dữ liệu, xử lý yêu cầu đặt giá theo thứ tự và cập nhật trạng thái theo thời gian thực.

Kết quả của Workshop cho thấy hệ thống có thể hỗ trợ các nghiệp vụ chính dành cho người dùng và quản trị viên, đồng thời cung cấp nền tảng để nhóm tiếp tục cải tiến và mở rộng sản phẩm trong tương lai.