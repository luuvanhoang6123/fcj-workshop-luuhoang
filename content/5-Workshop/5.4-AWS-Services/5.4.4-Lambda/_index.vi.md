---
title: "AWS Lambda"
date: 2026-07-27
weight: 4
chapter: false
pre: " <b> 5.4.4. </b> "
---

## Tổng quan

Hệ thống **Live Auction** sử dụng **AWS Lambda** để thực thi các nghiệp vụ backend mà không cần quản lý máy chủ.

Các Lambda Function tiếp nhận yêu cầu từ Amazon API Gateway, xử lý dữ liệu đấu giá, giao tiếp với Amazon DynamoDB, xử lý thông điệp từ Amazon SQS FIFO và gửi dữ liệu cập nhật đến người dùng thông qua API Gateway WebSocket.

Các Lambda Function được tạo và cấu hình bằng Terraform trong những module:

```text
infra/03-identity
infra/06-compute
infra/06-compute/stage3-control-plane
```

Mã nguồn của từng Lambda Function được đóng gói thành tệp `.zip` trước khi Terraform triển khai lên AWS.


## Vai trò của AWS Lambda

AWS Lambda là dịch vụ điện toán serverless cho phép thực thi mã nguồn khi có sự kiện xảy ra.

Trong hệ thống Live Auction, AWS Lambda được sử dụng để:

* Xử lý tài khoản sau khi người dùng xác nhận đăng ký.
* Xử lý nghiệp vụ phiên đấu giá.
* Xử lý thông tin vật phẩm.
* Truy vấn dữ liệu danh mục, phiên và vật phẩm.
* Xử lý các lệnh quản trị.
* Kiểm tra token khi người dùng kết nối WebSocket.
* Quản lý vòng đời kết nối WebSocket.
* Tiếp nhận và xử lý yêu cầu đặt giá.
* Xử lý tuần tự thông điệp từ SQS FIFO.
* Cập nhật dữ liệu trong DynamoDB.
* Gửi kết quả đấu giá đến các kết nối WebSocket.
* Ghi log thực thi vào Amazon CloudWatch.

## Các Lambda Function của hệ thống

Các Lambda Function chính của hệ thống gồm:

| Lambda Function             | Vai trò                                                              |
| --------------------------- | -------------------------------------------------------------------- |
| **la-cognito-post-confirm** | Xử lý sự kiện sau khi người dùng xác nhận tài khoản Cognito.         |
| **la-session-service**      | Xử lý nghiệp vụ liên quan đến phiên đấu giá.                         |
| **la-item-service**         | Xử lý nghiệp vụ liên quan đến vật phẩm và dữ liệu media.             |
| **la-query-service**        | Truy vấn danh mục, phiên đấu giá, vật phẩm và dữ liệu cần hiển thị.  |
| **la-admin-command**        | Xử lý các chức năng quản trị và lệnh điều khiển vòng đời phiên.      |
| **la-ws-authorizer**        | Kiểm tra Cognito JWT khi người dùng tạo kết nối WebSocket.           |
| **la-ws-handler**           | Quản lý kết nối WebSocket, phòng đấu giá và tiếp nhận lệnh đặt giá.  |
| **la-bid-processor**        | Xử lý thông điệp đặt giá từ SQS FIFO và cập nhật trạng thái đấu giá. |
| **la-broadcast**            | Gửi kết quả và trạng thái đấu giá đến các WebSocket Client.          |

Tên thực tế có thể chứa thêm tiền tố hoặc tên môi trường tùy cấu hình Terraform, nhưng các Function vẫn được phân biệt theo nghiệp vụ tương ứng.

## Nhóm Lambda xác thực

### Cognito Post Confirmation

Lambda `la-cognito-post-confirm` được Amazon Cognito kích hoạt sau khi người dùng xác nhận đăng ký tài khoản thành công.

Function này được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 128 MB
Timeout: 10 seconds
```

Lambda giúp thực hiện các nghiệp vụ cần thiết sau khi tài khoản được xác nhận.

Cấu hình Lambda Trigger của Function này đã được trình bày trong mục **5.4.1 — AWS IAM và Amazon Cognito**.

## Nhóm Lambda xử lý REST API

### Session Service

Lambda `la-session-service` xử lý các nghiệp vụ liên quan đến phiên đấu giá.

Các nghiệp vụ có thể bao gồm:

* Tạo phiên đấu giá.
* Cập nhật thông tin phiên.
* Truy xuất phiên của người dùng.
* Quản lý trạng thái phiên.
* Kiểm tra quyền sở hữu phiên.
* Kết nối phiên với một hoặc nhiều vật phẩm.

Function được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### Item Service

Lambda `la-item-service` xử lý nghiệp vụ của vật phẩm đấu giá.

Các nhiệm vụ chính gồm:

* Thêm vật phẩm vào phiên đấu giá.
* Cập nhật thông tin vật phẩm.
* Quản lý thông tin hình ảnh.
* Tạo thông tin cần thiết để tải media lên Amazon S3.
* Liên kết vật phẩm với phiên đấu giá.
* Kiểm tra dữ liệu đầu vào của vật phẩm.

Function được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### Query Service

Lambda `la-query-service` cung cấp các nghiệp vụ truy vấn dữ liệu.

Function được sử dụng để:

* Lấy danh sách phiên đấu giá.
* Xem chi tiết một phiên.
* Lấy danh sách vật phẩm.
* Xem thông tin vật phẩm.
* Truy vấn danh mục sản phẩm.
* Lấy dữ liệu cần hiển thị trên User Frontend và Admin Frontend.

Function được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### Admin Command

Lambda `la-admin-command` xử lý các nghiệp vụ dành cho quản trị viên.

Các chức năng liên quan gồm:

* Quản lý tài khoản người dùng.
* Quản lý danh mục sản phẩm.
* Duyệt phiên đấu giá.
* Tạo thêm tài khoản Admin.
* Xử lý các lệnh điều khiển trạng thái phiên.
* Phối hợp với EventBridge Scheduler để xử lý vòng đời phiên đấu giá.

Function được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 60 seconds
```

## Nhóm Lambda WebSocket

### WebSocket Authorizer

Lambda `la-ws-authorizer` kiểm tra Cognito JWT khi người dùng yêu cầu kết nối đến WebSocket API.

Quá trình kiểm tra gồm:

1. Nhận token từ yêu cầu kết nối.
2. Kiểm tra chữ ký và tính hợp lệ của token.
3. Kiểm tra Cognito issuer.
4. Lấy thông tin tài khoản từ token.
5. Cho phép hoặc từ chối kết nối WebSocket.

Function được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### WebSocket Handler

Lambda `la-ws-handler` quản lý vòng đời và yêu cầu của WebSocket Client.

Function xử lý:

* Sự kiện `$connect`.
* Sự kiện `$disconnect`.
* Lưu Connection ID vào DynamoDB.
* Xóa Connection ID khi người dùng ngắt kết nối.
* Quản lý việc tham gia phòng đấu giá.
* Tiếp nhận yêu cầu đặt giá.
* Chuyển yêu cầu đặt giá hợp lệ vào SQS FIFO.

Function được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### Broadcast Lambda

Lambda `la-broadcast` gửi kết quả đấu giá đến các WebSocket Client đang theo dõi phiên.

Function thực hiện:

* Nhận kết quả sau khi yêu cầu đặt giá được xử lý.
* Lấy danh sách Connection ID liên quan.
* Gửi giá mới đến API Gateway Management API.
* Loại bỏ kết nối không còn hợp lệ khi cần thiết.
* Cập nhật dữ liệu thời gian thực cho người tham gia.

Function được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

## Lambda xử lý đặt giá

Lambda `la-bid-processor` xử lý các yêu cầu đặt giá được gửi đến SQS FIFO.

Luồng xử lý được thực hiện như sau:

1. WebSocket Handler tiếp nhận yêu cầu đặt giá.
2. Yêu cầu hợp lệ được gửi vào SQS FIFO.
3. SQS chuyển thông điệp đến Bid Processor Lambda.
4. Lambda kiểm tra dữ liệu của yêu cầu.
5. Lambda cập nhật trạng thái đấu giá trong DynamoDB.
6. Kết quả được chuyển đến Broadcast Lambda hoặc WebSocket API.
7. Người dùng nhận mức giá mới theo thời gian thực.

Function được cấu hình:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

Event Source Mapping giữa SQS FIFO và Bid Processor được cấu hình:

```text
State: Enabled
Batch size: 10
Report batch item failures: Enabled
```

`ReportBatchItemFailures` cho phép Lambda báo cáo riêng những thông điệp xử lý thất bại thay vì xử lý lại toàn bộ batch.

## Lambda Deployment Package

Mã nguồn của mỗi Lambda Function được đóng gói thành tệp `.zip`.

Các package có thể gồm:

```text
cognito_post_confirm.zip
session_service.zip
item_service.zip
query_service.zip
admin_command.zip
ws_authorizer.zip
ws_handler.zip
bid_processor.zip
broadcast.zip
```

Terraform sử dụng đường dẫn đến các package này để tải mã nguồn lên AWS Lambda.

Khi mã nguồn thay đổi, package cần được build lại trước khi chạy `terraform plan` hoặc `terraform apply`. Nếu package chưa tồn tại, Terraform có thể báo lỗi tại hàm:

```text
filebase64sha256(...)
```

## Biến môi trường

Các Lambda Function sử dụng biến môi trường để nhận thông tin cấu hình khi thực thi.

Các biến môi trường có thể chứa:

* Tên DynamoDB Table.
* Tên SQS FIFO Queue.
* Cognito User Pool ID.
* Cognito issuer.
* WebSocket API endpoint.
* Tên Media Bucket.
* Media CloudFront Domain.
* Tên môi trường triển khai.
* Các giá trị cấu hình nghiệp vụ.

Không nên ghi cứng những giá trị này trong mã nguồn. Terraform truyền các giá trị cần thiết vào Lambda trong quá trình triển khai.

{{% notice warning %}}
Không chụp hoặc công khai toàn bộ biến môi trường nếu bên trong có endpoint nội bộ, token, secret hoặc thông tin định danh tài nguyên. Chỉ giữ lại những tên biến cần thiết để mô tả cấu hình và che phần giá trị nếu cần.
{{% /notice %}}

## Quyền truy cập của Lambda

Mỗi Lambda Function được gán một IAM Execution Role phù hợp với nghiệp vụ.

Tùy chức năng, Lambda có thể được cấp quyền:

* Ghi log vào Amazon CloudWatch.
* Đọc và ghi các bảng DynamoDB được chỉ định.
* Gửi thông điệp vào SQS FIFO.
* Nhận và xóa thông điệp SQS thông qua Event Source Mapping.
* Truy cập thông tin Cognito.
* Gửi dữ liệu đến API Gateway WebSocket Management API.
* Truy cập đối tượng trong Item Media Bucket.
* Tạo hoặc quản lý EventBridge Schedule.
* Gọi Lambda Function khác nếu nghiệp vụ yêu cầu.

IAM Role và Policy của Lambda đã được trình bày tại mục **5.4.1 — AWS IAM và Amazon Cognito**.

## Ghi log bằng Amazon CloudWatch

Mỗi Lambda Function có một CloudWatch Log Group với cấu trúc:

```text
/aws/lambda/<function-name>
```

Ví dụ:

```text
/aws/lambda/la-bid-processor
/aws/lambda/la-ws-handler
/aws/lambda/la-session-service
```

Log của Lambda giúp nhóm:

* Kiểm tra Function đã được gọi hay chưa.
* Xem thời gian bắt đầu và kết thúc thực thi.
* Kiểm tra lỗi trong quá trình xử lý.
* Theo dõi request và event.
* Phát hiện lỗi quyền IAM.
* Theo dõi lỗi kết nối DynamoDB hoặc SQS.
* Kiểm tra thời gian thực thi và lượng bộ nhớ sử dụng.

Không nên công khai toàn bộ nội dung log nếu log có chứa token, email, dữ liệu tài khoản hoặc nội dung yêu cầu của người dùng.

## Kiểm tra Lambda Function trên AWS Management Console

### Bước 1: Truy cập AWS Lambda

Đăng nhập vào **AWS Management Console**.

Tại thanh tìm kiếm, nhập:

```text
Lambda
```

Chọn **Lambda**.

Bảo đảm Region đang được chọn là:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

### Bước 2: Kiểm tra danh sách Function

Tại menu bên trái, chọn:

```text
Functions
```

Trong ô tìm kiếm, nhập:

```text
la-
```

Kiểm tra các Lambda Function của hệ thống.

Các nội dung cần kiểm tra gồm:

* Function name.
* Description.
* Package type.
* Runtime.
* Last modified.
* Region.
* Các Function cần thiết đã được triển khai đầy đủ hay chưa.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-function-list.png"
    title="Hình 5.4.4.1: Danh sách Lambda Function của hệ thống Live Auction"
    width="100%"
>}}

### Bước 3: Kiểm tra cấu hình Function

Chọn một Lambda Function, chẳng hạn:

```text
la-session-service
```

Tại trang tổng quan, kiểm tra:

* Function ARN.
* Runtime.
* Handler.
* Architecture.
* Last modified.
* Code package.
* Trạng thái Function.

Tiếp tục mở:

```text
Configuration → General configuration
```

Kiểm tra:

* Memory.
* Ephemeral storage.
* Timeout.
* Execution Role.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-general-configuration.png"
    title="Hình 5.4.4.2: Cấu hình Runtime, Memory và Timeout của Lambda Function"
    width="100%"
>}}

### Bước 4: Kiểm tra Trigger của Bid Processor

Chọn Function:

```text
la-bid-processor
```

Trong phần **Function overview** hoặc tab **Configuration**, kiểm tra SQS Trigger.

Các nội dung cần xác nhận:

* Trigger là Amazon SQS.
* Event Source Mapping đang Enabled.
* Queue trỏ đến Bid Command FIFO Queue.
* Batch size là `10`.
* Report batch item failures được bật.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-bid-processor-sqs-trigger.png"
    title="Hình 5.4.4.3: SQS Trigger của Bid Processor Lambda"
    width="100%"
>}}

### Bước 5: Kiểm tra API Gateway Trigger

Chọn một Lambda Function được REST API gọi, chẳng hạn:

```text
la-session-service
```

Trong **Function overview** hoặc:

```text
Configuration → Triggers
```

Kiểm tra API Gateway Trigger.

Các thông tin cần kiểm tra gồm:

* API Gateway được liên kết.
* API type.
* Stage.
* Statement ID.
* Quyền cho phép API Gateway gọi Lambda.
* Lambda Function được liên kết đúng với API.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-api-gateway-trigger.png"
    title="Hình 5.4.4.4: API Gateway Trigger của Lambda Function"
    width="100%"
>}}

### Bước 6: Kiểm tra biến môi trường

Trong trang Lambda Function, mở:

```text
Configuration → Environment variables
```

Kiểm tra các tên biến được cấu hình cho Function.

Chỉ cần xác nhận:

* Các biến cần thiết đã tồn tại.
* Tên DynamoDB Table đúng.
* Tên Queue đúng.
* Cognito configuration đúng.
* WebSocket endpoint hoặc Media Bucket đúng với môi trường.

Không hiển thị toàn bộ giá trị biến nếu có thông tin không muốn công khai.



{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-environment-variables.png"
    title="Hình 5.4.4.5: Các biến môi trường của Lambda Function"
    width="100%"
>}}

Mỗi Lambda Function đều có các biến mỗi trường riêng của chúng.

### Bước 7: Kiểm tra Execution Role

Trong Lambda Function, mở:

```text
Configuration → Permissions
```

Kiểm tra phần:

```text
Execution role
```

Các nội dung cần xác nhận:

* Lambda được gán IAM Role.
* Role có đúng tên theo quy ước dự án.
* Resource summary phù hợp với nghiệp vụ.
* Không gán quyền `AdministratorAccess`.
* Function chỉ truy cập những tài nguyên cần thiết.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-execution-role.png"
    title="Hình 5.4.4.6: IAM Execution Role được gán cho Lambda Function"
    width="100%"
>}}

### Bước 8: Kiểm tra CloudWatch Log

Chọn một Lambda Function đã được gọi, chẳng hạn:

```text
la-bid-processor
```

Mở:

```text
Monitor → View CloudWatch logs
```

Chọn Log Stream gần nhất và kiểm tra:

* Thời gian Function được gọi.
* Dòng `START`.
* Dòng `END`.
* Dòng `REPORT`.
* Execution duration.
* Memory used.
* Lỗi nếu có.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-cloudwatch-log.png"
    title="Hình 5.4.4.7: Log thực thi của Lambda Function trên Amazon CloudWatch"
    width="100%"
>}}

### Bước 9: Kiểm tra Metrics

Trong Lambda Function, mở tab:

```text
Monitor
```

Kiểm tra các Metrics:

* Invocations.
* Duration.
* Error count and success rate.
* Throttles.
* Concurrent executions.
* Async event age nếu được sử dụng.

Metrics giúp xác nhận Function đã được gọi và hỗ trợ phát hiện lỗi trong quá trình vận hành.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-monitoring-metrics.png"
    title="Hình 5.4.4.8: Metrics giám sát Lambda Function"
    width="100%"
>}}

## Kết quả

Sau khi kiểm tra trực tiếp trên AWS Management Console, nhóm ghi nhận:

* Các Lambda Function cần thiết đã được Terraform triển khai.
* Các Function sử dụng Python 3.13 và kiến trúc `x86_64`.
* Runtime, Handler, Memory và Timeout được cấu hình phù hợp với từng nhóm nghiệp vụ.
* Cognito Post Confirmation Lambda đã được liên kết với Cognito User Pool.
* Các Lambda xử lý REST API đã được kết nối với Amazon API Gateway.
* WebSocket Authorizer và WebSocket Handler đã sẵn sàng xử lý kết nối thời gian thực.
* Bid Processor Lambda đã được kết nối với SQS FIFO thông qua Event Source Mapping.
* Event Source Mapping đang ở trạng thái Enabled.
* Các Function được gán IAM Execution Role phù hợp.
* Các biến môi trường cần thiết đã được cấu hình.
* Lambda có thể ghi log vào Amazon CloudWatch.
* Metrics và log có thể được sử dụng để theo dõi hoạt động và xử lý lỗi.
* Các Lambda Function đã sẵn sàng tích hợp với API Gateway, DynamoDB, SQS FIFO và WebSocket API.

Việc cấu hình và kiểm tra REST API được trình bày tại mục **5.4.5 — Amazon API Gateway**.