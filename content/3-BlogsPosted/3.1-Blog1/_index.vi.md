---
title: "Blog 1 - Live Auction trên AWS Serverless"
date: 2026-08-09
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# LIVE AUCTION: XÂY DỰNG NỀN TẢNG ĐẤU GIÁ THỜI GIAN THỰC TRÊN AWS SERVERLESS

Trong quá trình thực hiện đồ án, nhóm em đã nghiên cứu và chia sẻ bài viết **“Live Auction: Xây dựng nền tảng đấu giá thời gian thực trên AWS Serverless”** trên cộng đồng AWS Study Group.

Bài viết trình bày quá trình xây dựng hệ thống đấu giá trực tuyến có khả năng xử lý yêu cầu đặt giá đồng thời, cập nhật giá theo thời gian thực, xác thực người dùng và triển khai hạ tầng bằng Terraform.

## Bài toán của hệ thống

Một trong những yêu cầu quan trọng của nền tảng đấu giá trực tuyến là duy trì được mức giá chính xác khi có nhiều người cùng đặt giá trong khoảng thời gian rất ngắn.

Ban đầu, hệ thống được xây dựng theo mô hình:

```text
React/Vite Frontend
        ↓
FastAPI Backend
        ↓
MySQL Database
```

Mô hình này vẫn được giữ lại để hỗ trợ quá trình phát triển và kiểm thử trong môi trường cục bộ. Tuy nhiên, khi phân tích sâu hơn, nhóm nhận thấy hệ thống còn phải giải quyết nhiều yêu cầu như:

* Cập nhật trạng thái đấu giá theo thời gian thực.
* Xử lý nhiều yêu cầu đặt giá đồng thời.
* Bảo đảm thứ tự xử lý lượt đặt giá.
* Xác thực và phân quyền tài khoản.
* Lưu trữ hình ảnh vật phẩm.
* Tự động thay đổi trạng thái phiên theo thời gian.
* Theo dõi lỗi và hỗ trợ khôi phục dữ liệu.
* Triển khai hạ tầng một cách nhất quán và có thể lặp lại.

Do đó, nhóm nghiên cứu và triển khai kiến trúc serverless trên AWS cho các nghiệp vụ chính của hệ thống.

## Kiến trúc tổng quát

Sơ đồ dưới đây mô tả kiến trúc AWS được sử dụng cho nền tảng Live Auction.

![Kiến trúc AWS Serverless của hệ thống Live Auction](/images/blog1/live-auction-serverless-architecture.jpg)

Kiến trúc kết hợp các dịch vụ chính:

* Amazon S3.
* Amazon CloudFront.
* Amazon Cognito.
* Amazon API Gateway.
* API Gateway WebSocket.
* AWS Lambda.
* Amazon DynamoDB.
* Amazon SQS FIFO.
* Amazon EventBridge.
* Amazon CloudWatch.
* AWS CloudTrail.
* AWS Backup.
* AWS IAM.
* AWS CodeBuild và các thành phần CI/CD.
* Terraform.

## Phân phối frontend bằng Amazon S3 và CloudFront

Hai giao diện User Frontend và Admin Frontend được build từ React/Vite thành các tệp tĩnh.

Luồng phân phối frontend:

```text
Mã nguồn React/Vite
        ↓
Build thành static asset
        ↓
Amazon S3
        ↓
Amazon CloudFront
        ↓
Người dùng
```

Các S3 Bucket được sử dụng để lưu trữ tệp HTML, CSS, JavaScript và những tài nguyên tĩnh khác. Amazon CloudFront lấy nội dung từ S3 Origin và phân phối đến người dùng thông qua HTTPS.

User Frontend và Admin Frontend có tài nguyên triển khai riêng, giúp tách biệt chức năng dành cho thành viên và quản trị viên. Hình ảnh vật phẩm cũng được lưu trữ và phân phối thông qua tài nguyên riêng để tránh ảnh hưởng đến các request API.

## Xác thực bằng Amazon Cognito

Amazon Cognito User Pool được sử dụng để quản lý:

* Đăng ký tài khoản.
* Xác nhận tài khoản.
* Đăng nhập.
* Khôi phục mật khẩu.
* Làm mới token.
* Phân biệt quyền thành viên và quản trị viên.

Sau khi đăng nhập thành công, Amazon Cognito phát hành JWT Token. Token này được sử dụng khi frontend gửi yêu cầu đến REST API hoặc thiết lập kết nối WebSocket.

Luồng xác thực:

```text
User/Admin Frontend
        ↓
Amazon Cognito
        ↓
Nhận JWT Token
        ↓
Gửi request đến API Gateway
        ↓
Kiểm tra token
        ↓
Cho phép hoặc từ chối yêu cầu
```

Lambda Post Confirmation được sử dụng để hoàn thành bước khởi tạo dữ liệu người dùng sau khi tài khoản được xác nhận.

## Xử lý nghiệp vụ bằng AWS Lambda

Thay vì tập trung toàn bộ nghiệp vụ vào một backend lớn, kiến trúc serverless tách các chức năng thành những Lambda Function có trách nhiệm riêng.

Một số nhóm xử lý gồm:

* Quản lý phiên đấu giá và các quy tắc của phiên.
* Quản lý vật phẩm và tải hình ảnh.
* Truy vấn danh mục và lịch sử đặt giá.
* Xử lý các thao tác quản trị.
* Quản lý kết nối WebSocket.
* Xử lý lượt đặt giá.
* Gửi kết quả mới đến người tham gia.

Các Lambda Function tiêu biểu gồm:

```text
session-service
item-service
query-service
admin-command
ws-authorizer
ws-handler
bid-processor
broadcast
```

Cách tổ chức này giúp từng thành phần có phạm vi rõ ràng, dễ kiểm tra và có thể được cập nhật độc lập.

## Đấu giá thời gian thực bằng WebSocket

Hệ thống sử dụng API Gateway WebSocket để duy trì kết nối giữa người tham gia và phòng đấu giá.

Luồng kết nối:

```text
Người dùng
    ↓
API Gateway WebSocket
    ↓
WebSocket Authorizer
    ↓
Lambda WebSocket Handler
    ↓
Lưu Connection ID và Room Membership
    ↓
Nhận cập nhật đấu giá theo thời gian thực
```

WebSocket Authorizer kiểm tra token khi người dùng thiết lập kết nối. Sau đó, Lambda Handler quản lý Connection ID và phòng đấu giá mà người dùng tham gia.

Nhờ kết nối WebSocket, hệ thống có thể chủ động gửi mức giá mới đến trình duyệt mà không yêu cầu frontend liên tục gửi request để kiểm tra.

## Xử lý lượt đặt giá bằng Amazon SQS FIFO

Yêu cầu đặt giá không được ghi trực tiếp từ trình duyệt vào bản ghi giá hiện tại. Thay vào đó, yêu cầu được đưa vào Amazon SQS FIFO.

Luồng xử lý:

```text
Người dùng đặt giá
        ↓
API Gateway WebSocket
        ↓
Lambda WebSocket Handler
        ↓
Amazon SQS FIFO
        ↓
Lambda Bid Processor
        ↓
Amazon DynamoDB
        ↓
Broadcast kết quả qua WebSocket
```

Message Group được tổ chức theo từng vật phẩm nhằm duy trì thứ tự xử lý các yêu cầu trong cùng một cuộc đấu giá.

Lambda Bid Processor kiểm tra:

* Trạng thái phiên.
* Thời gian bắt đầu và kết thúc.
* Mức giá hiện tại.
* Bước giá tối thiểu.
* Quyền tham gia của người dùng.
* Yêu cầu đã được xử lý trước đó hay chưa.

Conditional Update của DynamoDB giúp hạn chế trường hợp hai yêu cầu đồng thời ghi đè lên cùng một mức giá.

Idempotency Record giúp request gửi lại không làm thay đổi kết quả đã được xử lý, trong khi Bid Event lưu lại kết quả được chấp nhận hoặc từ chối.

## Lưu trữ dữ liệu bằng Amazon DynamoDB

Amazon DynamoDB được sử dụng để lưu các nhóm dữ liệu như:

* Tài khoản và hồ sơ.
* Danh mục.
* Phiên đấu giá.
* Vật phẩm.
* Trạng thái đấu giá.
* Lịch sử đặt giá.
* Idempotency Record.
* Connection ID của WebSocket.
* Thành viên trong phòng đấu giá.
* Audit Event.

DynamoDB phù hợp với kiến trúc Lambda vì không yêu cầu duy trì Connection Pool như cơ sở dữ liệu quan hệ truyền thống và có thể mở rộng theo lượng request.

## Gửi kết quả đến người tham gia

Sau khi một lượt đặt giá được xử lý, Lambda Broadcast đọc danh sách kết nối đang hoạt động trong phòng và gửi kết quả thông qua API Gateway Management API.

Những kết nối đã hết hiệu lực được loại bỏ để không ảnh hưởng đến các người dùng vẫn còn tham gia.

Dữ liệu cập nhật có thể bao gồm:

* Mức giá mới nhất.
* Trạng thái chấp nhận hoặc từ chối lượt đặt giá.
* Người đang dẫn đầu.
* Thời gian còn lại.
* Trạng thái của vật phẩm và phiên đấu giá.

Luồng gửi kết quả:

```text
Lambda Bid Processor
        ↓
Cập nhật Amazon DynamoDB
        ↓
Lambda Broadcast
        ↓
API Gateway Management API
        ↓
Các trình duyệt đang kết nối
```

## Tự động hóa vòng đời phiên

Amazon EventBridge được sử dụng để hỗ trợ các tác vụ theo thời gian như:

* Bắt đầu phiên đấu giá.
* Đóng vật phẩm.
* Chuyển sang vật phẩm tiếp theo.
* Kết thúc phiên.

Luồng xử lý:

```text
Amazon EventBridge
        ↓
Lambda Admin Command
        ↓
Cập nhật trạng thái phiên
        ↓
Amazon DynamoDB
        ↓
Thông báo trạng thái mới
```

Khi một tác vụ không được xử lý thành công, Dead-letter Queue có thể lưu lại sự kiện để nhóm kiểm tra thay vì làm mất sự kiện một cách im lặng.

## Lưu trữ hình ảnh vật phẩm

Hình ảnh vật phẩm được lưu trong Amazon S3 riêng biệt với các tệp frontend.

Khi người dùng thêm hình ảnh cho vật phẩm, hệ thống có thể thực hiện theo luồng:

```text
Frontend yêu cầu tải ảnh
        ↓
Amazon API Gateway
        ↓
AWS Lambda
        ↓
Tạo Presigned URL
        ↓
Frontend tải ảnh trực tiếp lên Amazon S3
```

Cách thực hiện này giúp dữ liệu hình ảnh không phải truyền qua Lambda, giảm thời gian xử lý và hạn chế tải không cần thiết cho API.

## Triển khai hạ tầng bằng Terraform

Các tài nguyên AWS được mô tả và quản lý bằng Terraform theo mô hình Infrastructure as Code.

Những nhóm tài nguyên chính gồm:

* Identity.
* Data.
* Messaging.
* Compute.
* REST API.
* WebSocket API.
* Edge.
* Security.
* Monitoring.
* Backup.
* CI/CD.

Quy trình triển khai cơ bản:

```text
Terraform Configuration
          ↓
terraform init
          ↓
terraform plan
          ↓
Kiểm tra kế hoạch
          ↓
terraform apply
          ↓
Tạo tài nguyên trên AWS
```

Việc sử dụng Terraform giúp cấu hình hạ tầng có thể được kiểm tra, lưu phiên bản và triển khai lại một cách nhất quán.

Terraform cũng hỗ trợ nhóm quản lý quan hệ phụ thuộc giữa các thành phần, chẳng hạn Lambda cần IAM Role, API Gateway cần Lambda Integration và CloudFront cần S3 Origin.

## CI/CD

AWS CodeBuild được sử dụng để hỗ trợ build và đóng gói Lambda Artifact cũng như frontend asset.

Quy trình tổng quát:

```text
Thành viên cập nhật mã nguồn
          ↓
Repository nhận thay đổi
          ↓
AWS CodeBuild
          ↓
Build và đóng gói
          ↓
Tạo Artifact
          ↓
Triển khai phiên bản mới
```

CodePipeline và CodeDeploy có thể được sử dụng để hỗ trợ quá trình cập nhật phiên bản một cách có kiểm soát.

Việc áp dụng CI/CD giúp giảm các thao tác triển khai thủ công, đồng thời tạo lịch sử cho từng lần build và cập nhật hệ thống.

## Giám sát và ghi nhận hoạt động

Amazon CloudWatch thu thập:

* Log của Lambda.
* Chỉ số hoạt động của API.
* Số lượng message trong SQS.
* Lỗi xử lý.
* Độ trễ.
* Các chỉ số liên quan đến lượt đặt giá.
* Trạng thái của Dead-letter Queue.

CloudWatch Alarm có thể gửi cảnh báo khi:

* Lambda xuất hiện nhiều lỗi.
* Độ trễ tăng cao.
* SQS có message tồn đọng.
* Dead-letter Queue nhận được message.
* Tỷ lệ yêu cầu đặt giá bị từ chối tăng bất thường.

Ngoài Amazon CloudWatch, kiến trúc còn sử dụng:

* AWS CloudTrail để ghi nhận hoạt động API trên tài khoản AWS.
* AWS Config để theo dõi cấu hình tài nguyên.
* IAM Access Analyzer để hỗ trợ phát hiện quyền truy cập ngoài dự kiến.
* AWS Backup để hỗ trợ sao lưu và khôi phục dữ liệu.
* Amazon SNS để gửi thông báo cảnh báo.

## Bảo mật hệ thống

AWS IAM kiểm soát quyền truy cập giữa các dịch vụ. Mỗi Lambda Function chỉ được cấp những quyền cần thiết cho nhiệm vụ của nó.

Một số ví dụ:

* Lambda quản lý dữ liệu được cấp quyền truy cập các bảng DynamoDB cần thiết.
* Lambda WebSocket được cấp quyền quản lý kết nối.
* Lambda xử lý đặt giá được cấp quyền nhận message từ SQS FIFO.
* Lambda tiếp nhận yêu cầu được cấp quyền gửi message vào SQS.
* CloudFront được cấp quyền lấy nội dung từ S3 Origin.
* CodeBuild được cấp quyền truy cập các tài nguyên phục vụ quá trình build và triển khai.

Nhóm hướng đến nguyên tắc cấp quyền tối thiểu, tránh sử dụng một Role có toàn bộ quyền cho nhiều dịch vụ khác nhau.

## Bài học đạt được

Qua quá trình xây dựng hệ thống, nhóm nhận thấy serverless không có nghĩa là có thể bỏ qua các quyết định thiết kế.

Một kiến trúc serverless vẫn cần:

* Phân chia trách nhiệm rõ ràng.
* Xác định Access Pattern phù hợp.
* Kiểm soát retry.
* Bảo đảm tính idempotent.
* Cấp quyền IAM đúng phạm vi.
* Xử lý connection hết hiệu lực.
* Quản lý message thất bại.
* Theo dõi log và metrics.
* Có phương án sao lưu và khôi phục.

AWS cung cấp các thành phần để xây dựng hệ thống, nhưng độ ổn định phụ thuộc vào cách các dịch vụ được kết nối và phối hợp với nhau.

## Kết quả đạt được

Qua bài viết và quá trình thực hiện đồ án, nhóm đã:

* Xây dựng được kiến trúc serverless cho nền tảng đấu giá trực tuyến.
* Tách User Frontend và Admin Frontend.
* Phân phối frontend thông qua Amazon S3 và CloudFront.
* Xác thực tài khoản bằng Amazon Cognito.
* Xử lý nghiệp vụ bằng AWS Lambda.
* Cung cấp REST API và WebSocket API bằng Amazon API Gateway.
* Xử lý yêu cầu đặt giá theo thứ tự bằng Amazon SQS FIFO.
* Lưu dữ liệu và trạng thái bằng Amazon DynamoDB.
* Tự động hóa hạ tầng bằng Terraform.
* Theo dõi hệ thống bằng Amazon CloudWatch.
* Hiểu rõ hơn về kiến trúc hướng sự kiện và xử lý dữ liệu theo thời gian thực.

## Hướng phát triển

Hệ thống vẫn có thể tiếp tục được cải thiện thông qua:

* Thực hiện load test ở quy mô lớn hơn.
* Hoàn thiện cơ chế thông báo.
* Bổ sung phân tích dữ liệu đấu giá.
* Kiểm tra phương án khôi phục đa vùng.
* Tối ưu chi phí vận hành.
* Hoàn thiện cảnh báo và dashboard giám sát.
* Tiếp tục đánh giá độ tin cậy của luồng đặt giá khi số lượng người dùng tăng.

## Liên kết bài viết

Bài viết đã được đăng và duyệt trên cộng đồng AWS Study Group:

[**Xem bài viết “Live Auction: Xây dựng nền tảng đấu giá thời gian thực trên AWS Serverless”**](https://www.facebook.com/groups/awsstudygroupfcj/posts/2239889186776041/)