---
title: "API Gateway WebSocket"
date: 2026-07-27
weight: 6
chapter: false
pre: " <b> 5.4.6. </b> "
---

## Tổng quan

Hệ thống **Live Auction** sử dụng **Amazon API Gateway WebSocket API** để duy trì kết nối hai chiều giữa frontend và backend.

Khác với REST API, WebSocket cho phép máy chủ chủ động gửi dữ liệu đến trình duyệt mà không cần người dùng liên tục gửi yêu cầu mới. Cơ chế này phù hợp với chức năng đấu giá trực tuyến vì giá hiện tại và trạng thái của phiên cần được cập nhật nhanh đến những người đang theo dõi.

WebSocket API được tạo và cấu hình bằng Terraform trong module:

```text
infra/07-api
```

Tên WebSocket API của hệ thống:

```text
la-websocket
```

WebSocket API được tích hợp với các Lambda Function:

```text
la-ws-authorizer
la-ws-handler
la-bid-processor
la-broadcast
```


## Vai trò của WebSocket API

Trong hệ thống Live Auction, WebSocket API được sử dụng để:

* Thiết lập kết nối thời gian thực giữa người dùng và hệ thống.
* Xác thực người dùng khi tạo kết nối.
* Theo dõi vòng đời của kết nối.
* Cho phép người dùng tham gia phòng của phiên đấu giá.
* Tiếp nhận yêu cầu đặt giá.
* Chuyển yêu cầu đặt giá đến Lambda xử lý.
* Cập nhật giá mới đến các người dùng đang theo dõi.
* Gửi trạng thái phiên và vật phẩm theo thời gian thực.
* Phát hiện và loại bỏ các kết nối không còn hoạt động.
* Hạn chế việc frontend phải liên tục gửi REST API request để lấy giá mới.

## WebSocket URL

WebSocket API được triển khai tại Stage:

```text
prod
```

WebSocket URL có cấu trúc:

```text
wss://<websocket-api-id>.execute-api.ap-southeast-1.amazonaws.com/prod
```

Tiền tố `wss://` cho biết kết nối WebSocket được mã hóa bằng TLS.

Hệ thống cũng sử dụng WebSocket Management Endpoint có cấu trúc:

```text
https://<websocket-api-id>.execute-api.ap-southeast-1.amazonaws.com/prod
```

Management Endpoint được Lambda sử dụng để gửi dữ liệu chủ động đến một WebSocket Connection.

## Luồng hoạt động WebSocket

Luồng hoạt động tổng quát được thực hiện như sau:

1. Người dùng đăng nhập thông qua Amazon Cognito.
2. Frontend nhận token sau khi đăng nhập thành công.
3. Frontend tạo kết nối đến WebSocket URL.
4. Token được gửi kèm theo yêu cầu `$connect`.
5. API Gateway gọi `la-ws-authorizer` để kiểm tra token.
6. Nếu token hợp lệ, API Gateway cho phép thiết lập kết nối.
7. API Gateway gọi `la-ws-handler` để xử lý sự kiện `$connect`.
8. Lambda lưu Connection ID và thông tin liên quan vào DynamoDB.
9. Người dùng gửi action `joinRoom` để tham gia phòng đấu giá.
10. Người dùng gửi action `placeBid` để đặt giá.
11. WebSocket Handler kiểm tra yêu cầu và gửi lệnh đặt giá vào SQS FIFO.
12. Bid Processor Lambda xử lý thông điệp và cập nhật DynamoDB.
13. Broadcast Lambda lấy các Connection ID liên quan.
14. Broadcast Lambda gửi giá mới qua WebSocket Management API.
15. Frontend nhận dữ liệu và cập nhật giao diện.
16. Khi người dùng ngắt kết nối, route `$disconnect` được gọi để xóa Connection ID.

Luồng tổng quát:

```text
User Frontend
      ↓
API Gateway WebSocket
      ↓
WebSocket Authorizer
      ↓
WebSocket Handler
      ↓
Amazon SQS FIFO
      ↓
Bid Processor Lambda
      ↓
Amazon DynamoDB
      ↓
Broadcast Lambda
      ↓
API Gateway Management API
      ↓
WebSocket Clients
```

## Route Selection Expression

WebSocket API sử dụng Route Selection Expression:

```text
$request.body.action
```

API Gateway đọc trường `action` trong nội dung JSON mà frontend gửi để chọn route phù hợp.

Ví dụ yêu cầu tham gia phòng:

```json
{
  "action": "joinRoom",
  "sessionId": "<session-id>"
}
```

Ví dụ yêu cầu đặt giá:

```json
{
  "action": "placeBid",
  "itemId": "<item-id>",
  "amount": 1000000
}
```

Nếu `action` là `joinRoom`, API Gateway gọi route `joinRoom`. Nếu `action` là `placeBid`, API Gateway gọi route `placeBid`.

## Các WebSocket Route

WebSocket API sử dụng bốn route chính:

| Route           | Authorization     | Vai trò                                         |
| --------------- | ----------------- | ----------------------------------------------- |
| **$connect**    | Custom Authorizer | Xác thực và thiết lập kết nối WebSocket.        |
| **$disconnect** | None              | Xử lý khi WebSocket Client ngắt kết nối.        |
| **joinRoom**    | None              | Đăng ký Connection vào phòng của phiên đấu giá. |
| **placeBid**    | None              | Tiếp nhận yêu cầu đặt giá từ người dùng.        |

Cấu trúc route:

```text
la-websocket
├── $connect
├── $disconnect
├── joinRoom
└── placeBid
```

Route `$connect` sử dụng Custom Authorizer để chỉ cho phép tài khoản có token hợp lệ thiết lập kết nối.

Các route còn lại hoạt động trên kết nối đã được thiết lập. Lambda Handler tiếp tục kiểm tra dữ liệu và thông tin nhận dạng được lưu trong hệ thống trước khi thực hiện nghiệp vụ.

## Route $connect

Route `$connect` được gọi khi frontend yêu cầu mở kết nối WebSocket.

Luồng xử lý:

1. Frontend mở WebSocket connection.
2. API Gateway lấy token từ query string.
3. API Gateway gọi WebSocket Authorizer.
4. Authorizer kiểm tra Cognito JWT.
5. Nếu token hợp lệ, kết nối được chấp nhận.
6. WebSocket Handler nhận sự kiện `$connect`.
7. Connection ID được lưu vào DynamoDB.

Route được cấu hình:

```text
Route key: $connect
Authorization type: CUSTOM
Authorizer: la-ws-authorizer
Integration: la-ws-handler
```

## WebSocket Authorizer

WebSocket API sử dụng REQUEST Authorizer:

```text
Authorizer type: REQUEST
Identity source: route.request.querystring.token
Lambda function: la-ws-authorizer
```

Frontend gửi token trong query string khi thiết lập kết nối:

```text
wss://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod?token=<cognito-token>
```

Authorizer kiểm tra:

* Token có tồn tại hay không.
* Chữ ký của JWT.
* Token có hết hạn hay không.
* Cognito issuer.
* Thông tin định danh tài khoản.
* Token có được cấp bởi đúng Cognito User Pool hay không.


## Route $disconnect

Route `$disconnect` được gọi khi kết nối WebSocket kết thúc.

Route được cấu hình:

```text
Route key: $disconnect
Authorization type: NONE
Integration: la-ws-handler
```

WebSocket Handler sử dụng Connection ID trong sự kiện để:

* Xóa thông tin kết nối khỏi DynamoDB.
* Xóa Connection khỏi các phòng đấu giá liên quan.
* Hạn chế gửi dữ liệu đến Connection không còn hoạt động.
* Giải phóng dữ liệu kết nối không còn cần thiết.

Sự kiện `$disconnect` có thể xảy ra khi:

* Người dùng đóng trình duyệt.
* Người dùng rời khỏi trang.
* Mạng bị ngắt.
* Kết nối hết thời gian hoạt động.
* Client chủ động đóng WebSocket.

## Route joinRoom

Route `joinRoom` cho phép một WebSocket Client tham gia phòng của phiên đấu giá.

Route được cấu hình:

```text
Route key: joinRoom
Authorization type: NONE
Integration: la-ws-handler
```

Frontend gửi yêu cầu có cấu trúc:

```json
{
  "action": "joinRoom",
  "sessionId": "<session-id>"
}
```

WebSocket Handler liên kết Connection ID với phiên đấu giá tương ứng. Thông tin này được sử dụng khi Broadcast Lambda cần gửi dữ liệu đến những người đang theo dõi phiên.

## Route placeBid

Route `placeBid` tiếp nhận yêu cầu đặt giá từ người dùng.

Route được cấu hình:

```text
Route key: placeBid
Authorization type: NONE
Integration: la-ws-handler
```

Ví dụ nội dung yêu cầu:

```json
{
  "action": "placeBid",
  "itemId": "<item-id>",
  "amount": 1000000
}
```

WebSocket Handler thực hiện:

1. Kiểm tra cấu trúc yêu cầu.
2. Kiểm tra Connection và thông tin tài khoản.
3. Kiểm tra Item ID.
4. Kiểm tra giá trị đặt giá.
5. Tạo thông điệp đặt giá.
6. Gửi thông điệp vào Amazon SQS FIFO.
7. Phản hồi trạng thái tiếp nhận yêu cầu cho Client.

Việc xử lý và cập nhật giá cuối cùng không được thực hiện trực tiếp trong WebSocket Handler. Yêu cầu được chuyển qua SQS FIFO để bảo đảm thứ tự xử lý.

## Lambda Integration

Bốn WebSocket Route được tích hợp với:

```text
la-ws-handler
```

Integration được cấu hình:

```text
Integration type: AWS_PROXY
Integration method: POST
```

AWS_PROXY Integration cho phép API Gateway chuyển toàn bộ thông tin sự kiện WebSocket đến Lambda, bao gồm:

* Route Key.
* Connection ID.
* Domain Name.
* Stage.
* Request Context.
* Message Body.
* Thông tin Authorizer nếu có.

WebSocket Authorizer được tích hợp riêng với:

```text
la-ws-authorizer
```

## Quản lý Connection ID

Mỗi kết nối WebSocket được API Gateway cấp một Connection ID.

Connection ID được lưu trong DynamoDB để hệ thống có thể:

* Xác định Client đang kết nối.
* Xác định người dùng tương ứng.
* Xác định phòng đấu giá mà Client đang theo dõi.
* Gửi dữ liệu đến đúng Client.
* Xóa kết nối không còn hoạt động.

Connection ID chỉ có hiệu lực trong thời gian kết nối còn tồn tại.

Khi Broadcast Lambda gửi dữ liệu đến một Connection đã hết hạn, API Gateway có thể trả lỗi:

```text
410 Gone
```

Khi đó, Lambda xóa Connection ID không còn hợp lệ khỏi DynamoDB.

## Gửi dữ liệu đến WebSocket Client

Broadcast Lambda sử dụng API Gateway Management API để gửi dữ liệu đến Client.

Thao tác chính:

```text
POST @connections/{connection_id}
```

Dữ liệu có thể bao gồm:

* Giá hiện tại.
* Thông tin người đặt giá mới nhất.
* Trạng thái đặt giá thành công hoặc thất bại.
* Trạng thái vật phẩm.
* Thời gian còn lại.
* Trạng thái phiên đấu giá.

Ví dụ dữ liệu cập nhật:

```json
{
  "type": "bidUpdated",
  "itemId": "<item-id>",
  "currentPrice": 1100000
}
```

Lambda cần quyền IAM:

```text
execute-api:ManageConnections
```

Quyền này chỉ được cấp cho phạm vi WebSocket API cần thiết.

## Stage triển khai

WebSocket API được triển khai vào Stage:

```text
prod
```

Stage được cấu hình:

```text
Auto deploy: Enabled
```

Khi cấu hình route hoặc integration thay đổi, API Gateway tự động cập nhật Stage mà không cần tạo deployment thủ công.

Các endpoint của Stage gồm:

```text
WebSocket URL:
wss://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod

Connection URL:
https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod/@connections
```

## Kiểm tra WebSocket API trên AWS Management Console

### Bước 1: Truy cập Amazon API Gateway

Đăng nhập vào **AWS Management Console**.

Tại thanh tìm kiếm, nhập:

```text
API Gateway
```

Chọn **API Gateway**.

Bảo đảm Region đang được chọn là:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

## Bước 2: Kiểm tra WebSocket API

Trong danh sách API Gateway, chọn API:

```text
la-websocket
```

Tại màn hình **Routes**, kiểm tra:

- Tên API là `la-websocket`.
- API sử dụng giao thức WebSocket.
- Route Selection Expression.
- Danh sách các route đã được cấu hình.
- Cấu hình Authorizer của route `$connect`.

Route Selection Expression của hệ thống là:

```text
$request.body.action
```

Các route được triển khai gồm:

```text
$connect
$disconnect
joinRoom
placeBid
```

Trong đó, route `$connect` sử dụng Lambda Authorizer `la-ws-authorizer` để xác thực kết nối trước khi cho phép client tham gia WebSocket API.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-api-overview.png"
    title="Hình 5.4.6.1: Route Selection Expression, các Route và Authorizer của WebSocket API"
    width="100%"
>}}

### Bước 3: Kiểm tra Routes

Trong WebSocket API, mở:

```text
Routes
```

Kiểm tra bốn route:

```text
$connect
$disconnect
joinRoom
placeBid
```

Các nội dung cần xác nhận:

* Route Key.
* Authorization của route.
* Integration được gán.
* Route `$connect` sử dụng Custom Authorizer.
* Các route được tích hợp với WebSocket Handler Lambda.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-routes.png"
    title="Hình 5.4.6.2: Các Route của API Gateway WebSocket"
    width="50%"
>}}

### Bước 4: Kiểm tra route $connect

Chọn route:

```text
$connect
```

Kiểm tra:

* Authorization là Custom.
* Authorizer đã được gán.
* Integration trỏ đến Lambda.
* Route đang được gắn với Stage `prod`.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-connect-route.png"
    title="Hình 5.4.6.3: Cấu hình route $connect của WebSocket API"
    width="100%"
>}}

### Bước 5: Kiểm tra WebSocket Authorizer

Trong WebSocket API, mở:

```text
Authorizers
```

Chọn Authorizer của hệ thống và kiểm tra:

* Authorizer Name.
* Authorizer Type là REQUEST.
* Lambda Function là `la-ws-authorizer`.
* Identity Source là `route.request.querystring.token`.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-authorizer.png"
    title="Hình 5.4.6.4: Lambda Authorizer của WebSocket API"
    width="100%"
>}}

### Bước 6: Kiểm tra Lambda Integration

Trong WebSocket API, chọn từng Route và mở tab:

```text
Integration request
```

Kiểm tra Lambda Integration được liên kết với Route.

Các nội dung cần xác nhận:

- Integration Type là `Lambda`.
- Lambda Proxy Integration được bật.
- Lambda Function là `la-ws-handler`.
- Region là `ap-southeast-1`.
- Các Route được liên kết với đúng Lambda Integration.

Trong hệ thống Live Auction, các Route sau sử dụng Lambda Function `la-ws-handler` để xử lý kết nối và thông điệp WebSocket:

```text
$connect
$disconnect
joinRoom
placeBid
```

Ảnh dưới đây minh họa Integration Request của Route `$connect`. Có thể chọn lần lượt các Route còn lại để xác nhận chúng cũng được liên kết với Lambda Function tương ứng.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-lambda-integration.png"
    title="Hình 5.4.6.5: Lambda Proxy Integration của Route $connect trong WebSocket API"
    width="100%"
>}}

### Bước 7: Kiểm tra Stage prod

Trong WebSocket API, mở:

```text
Stages → prod
```

Kiểm tra:

* WebSocket URL.
* Connection URL.
* Auto Deploy.
* Deployment status.
* Last updated date.
* Stage Name.

Trạng thái cần xác nhận:

```text
Auto deploy: Enabled
```


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-prod-stage.png"
    title="Hình 5.4.6.6: Stage prod của API Gateway WebSocket"
    width="100%"
>}}

### Bước 8: Kiểm tra Lambda Trigger

Mở Lambda Function:

```text
la-ws-handler
```

Kiểm tra phần:

```text
Function overview
```

hoặc:

```text
Configuration → Triggers
```

Xác nhận API Gateway WebSocket đã được liên kết với Lambda.

Các nội dung cần kiểm tra:

* API Gateway Trigger.
* API Type là WebSocket.
* Stage là `prod`.
* Lambda Permission cho phép API Gateway gọi Function.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-lambda-trigger.png"
    title="Hình 5.4.6.7: API Gateway WebSocket Trigger của Lambda Handler"
    width="100%"
>}}

### Bước 9: Kiểm tra CloudWatch Logs

Sau khi WebSocket API được sử dụng, mở Lambda Function:

```text
la-ws-handler
```

Sau đó truy cập:

```text
Monitor → View CloudWatch logs
```

AWS chuyển đến CloudWatch Log Group:

```text
/aws/lambda/la-ws-handler
```

Trong Log Group, chọn Log Stream có thời gian cập nhật gần nhất và kiểm tra các Log Event của Lambda.

Các nội dung cần xác nhận:

- Lambda Function `la-ws-handler` đã được gọi.
- Log có các bản ghi `START`, `END` và `REPORT`.
- Quá trình thực thi Lambda đã hoàn tất.
- Không xuất hiện bản ghi `ERROR`.
- Không có lỗi quyền truy cập IAM hoặc lỗi thao tác với DynamoDB.
- Bản ghi `REPORT` thể hiện thời gian xử lý và lượng bộ nhớ được sử dụng.
- Không công khai token, Connection ID, email hoặc dữ liệu đặt giá của người dùng.

Nếu ứng dụng có ghi log nghiệp vụ, có thể kiểm tra thêm các sự kiện WebSocket:

```text
$connect
$disconnect
joinRoom
placeBid
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-handler-cloudwatch-log.png"
    title="Hình 5.4.6.8: CloudWatch Logs của Lambda WebSocket Handler"
    width="100%"
>}}

Kết quả kiểm tra cho thấy Lambda Function `la-ws-handler` đã được gọi và hoàn tất quá trình thực thi. Log Stream ghi nhận đầy đủ các bản ghi `START`, `END` và `REPORT`, đồng thời không xuất hiện bản ghi `ERROR`.

Bản ghi `REPORT` cung cấp thông tin về thời gian xử lý, thời gian tính phí và lượng bộ nhớ được sử dụng. Điều này xác nhận Lambda WebSocket Handler đang hoạt động và có thể tiếp nhận các sự kiện được API Gateway WebSocket chuyển tiếp.

Do ứng dụng hiện chưa ghi riêng `routeKey` và `connectionId` vào CloudWatch Logs, kết quả này chỉ được sử dụng để xác nhận quá trình thực thi Lambda thành công. Việc kiểm tra từng Route và Lambda Integration đã được thực hiện thông qua cấu hình trong API Gateway.

## Kiểm tra kết nối WebSocket

Sau khi kiểm tra tài nguyên trên AWS Console, có thể kiểm tra kết nối bằng frontend hoặc công cụ hỗ trợ WebSocket.

WebSocket URL minh họa:

```text
wss://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod?token=<cognito-token>
```

Kết quả mong đợi:

* Token hợp lệ: Kết nối được thiết lập.
* Thiếu token: Kết nối bị từ chối.
* Token không hợp lệ: Kết nối bị từ chối.
* `joinRoom`: Connection được liên kết với phiên đấu giá.
* `placeBid`: Yêu cầu được tiếp nhận và đưa vào hàng đợi.
* Sau khi đặt giá thành công: Frontend nhận giá mới.
* Khi đóng kết nối: Connection ID được xóa khỏi DynamoDB.

Việc kiểm thử chức năng WebSocket đầy đủ được trình bày tại mục **5.5 — Kiểm thử hệ thống**.

## Kết quả

Sau khi kiểm tra trực tiếp trên AWS Management Console, nhóm ghi nhận:

* WebSocket API `la-websocket` đã được Terraform tạo thành công.
* Route Selection Expression được cấu hình là `$request.body.action`.
* Bốn route `$connect`, `$disconnect`, `joinRoom` và `placeBid` đã được tạo.
* Route `$connect` sử dụng Custom Lambda Authorizer.
* Authorizer kiểm tra Cognito token từ query string.
* Các route được tích hợp với `la-ws-handler`.
* Stage `prod` đã được triển khai và bật Auto Deploy.
* WebSocket Handler có thể quản lý Connection ID và phòng đấu giá.
* Yêu cầu đặt giá được chuyển từ WebSocket đến SQS FIFO.
* Bid Processor xử lý thông điệp theo thứ tự.
* Broadcast Lambda có thể gửi giá mới đến các WebSocket Client.
* Lambda ghi log thực thi vào Amazon CloudWatch.
* WebSocket API đã sẵn sàng hỗ trợ chức năng đấu giá theo thời gian thực.

Việc triển khai và kiểm tra các bảng dữ liệu của hệ thống được trình bày tại mục **5.4.7 — Amazon DynamoDB**.