---
title : "Giới thiệu"
date : 2026-07-13 
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

# đồ án và kiến trúc triển khai

## Giới thiệu

**Live Auction** là nền tảng đấu giá trực tuyến được phát triển nhằm cung cấp môi trường đấu giá minh bạch, thuận tiện và hỗ trợ cập nhật dữ liệu theo thời gian thực.

Hệ thống cung cấp hai giao diện riêng biệt dành cho người dùng và quản trị viên. Người dùng có thể tạo tài khoản, đăng nhập, xem thông tin cá nhân, theo dõi các phiên đấu giá, tạo và quản lý phiên của mình, thêm một hoặc nhiều vật phẩm vào phiên và tham gia đặt giá. Quản trị viên có thể quản lý tài khoản người dùng, quản lý danh mục sản phẩm, duyệt phiên đấu giá và tạo thêm tài khoản quản trị viên.

Trong quá trình thực hiện đồ án, nhóm triển khai hệ thống trên nền tảng **Amazon Web Services (AWS)** theo kiến trúc **serverless**. Toàn bộ hạ tầng được xây dựng và quản lý bằng **Terraform (Infrastructure as Code)**, giúp tự động hóa quá trình tạo và cấu hình các tài nguyên AWS, đồng thời bảo đảm tính nhất quán giữa các lần triển khai.

Sau khi hạ tầng được triển khai, các dịch vụ AWS được tích hợp để xây dựng hệ thống hoàn chỉnh, bao gồm lưu trữ và phân phối hai giao diện frontend, xác thực và phân quyền tài khoản, xử lý API nghiệp vụ, lưu trữ dữ liệu, xử lý tuần tự yêu cầu đặt giá và cập nhật trạng thái đấu giá theo thời gian thực.

## Mục tiêu

Workshop được thực hiện nhằm các mục tiêu sau:

* Tìm hiểu quy trình triển khai một hệ thống thực tế trên nền tảng AWS.
* Triển khai hạ tầng bằng Terraform theo mô hình Infrastructure as Code.
* Tích hợp các dịch vụ AWS để xây dựng hệ thống Live Auction.
* Triển khai riêng giao diện dành cho người dùng và giao diện dành cho quản trị viên.
* Xây dựng chức năng đấu giá theo thời gian thực trên kiến trúc serverless.
* Kiểm thử và đánh giá khả năng hoạt động của hệ thống sau khi triển khai.

## Kiến trúc triển khai

Hệ thống Live Auction được triển khai theo mô hình kiến trúc serverless trên AWS. Hạ tầng được quản lý bằng Terraform, trong khi các dịch vụ AWS được kết hợp để cung cấp các chức năng từ xác thực tài khoản, phân quyền, xử lý nghiệp vụ và lưu trữ dữ liệu đến truyền thông tin đấu giá theo thời gian thực.

User Frontend và Admin Frontend được triển khai độc lập nhằm tách biệt chức năng dành cho người dùng và quản trị viên. Cả hai giao diện cùng kết nối đến các dịch vụ backend thông qua Amazon API Gateway và được kiểm soát quyền truy cập dựa trên token do Amazon Cognito cấp.


Sơ đồ dưới đây mô tả kiến trúc triển khai thực tế của hệ thống Live Auction trên nền tảng AWS. Sơ đồ thể hiện mối liên hệ giữa người dùng, hai giao diện frontend, các dịch vụ xác thực, API, xử lý nghiệp vụ, lưu trữ dữ liệu, xử lý đấu giá theo thời gian thực, giám sát và quy trình CI/CD.

{{< figure
    src="/images/5-Workshop/5.1-Workshop-overview/live-auction-deployment-architecture.jpg"
    title="Hình 5.1.1: Kiến trúc triển khai hệ thống Live Auction trên AWS"
    width="100%"
>}}

## Các dịch vụ AWS sử dụng

Bảng dưới đây trình bày các công cụ và dịch vụ AWS được sử dụng trong quá trình triển khai hệ thống.

| Công cụ/Dịch vụ        | Vai trò                                                                            |
| ---------------------- | ---------------------------------------------------------------------------------- |
| **Terraform**          | Quản lý và triển khai hạ tầng theo mô hình Infrastructure as Code.                 |
| **AWS IAM**            | Quản lý Role, Policy và quyền truy cập giữa các dịch vụ AWS.                       |
| **Amazon Cognito**     | Xác thực tài khoản và phân biệt quyền User với Admin.                              |
| **Amazon S3**          | Lưu trữ User Frontend, Admin Frontend và các tài nguyên tĩnh.                      |
| **Amazon CloudFront**  | Phân phối hai giao diện frontend thông qua CDN.                                    |
| **AWS Lambda**         | Thực thi các nghiệp vụ của người dùng, quản trị viên và đấu giá.                   |
| **Amazon API Gateway** | Cung cấp REST API và WebSocket API cho hệ thống.                                   |
| **Amazon DynamoDB**    | Lưu trữ tài khoản, danh mục, phiên, vật phẩm, lượt đặt giá và trạng thái hệ thống. |
| **Amazon SQS FIFO**    | Tiếp nhận và xử lý tuần tự các yêu cầu đặt giá.                                    |

Các nội dung cấu hình và triển khai từng thành phần được trình bày chi tiết trong các mục tiếp theo của Workshop.