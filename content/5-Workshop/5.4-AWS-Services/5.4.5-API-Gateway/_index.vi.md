---
title: "Amazon API Gateway"
date: 2026-07-27
weight: 5
chapter: false
pre: " <b> 5.4.5. </b> "
---

## Tổng quan

Hệ thống **Live Auction** sử dụng **Amazon API Gateway REST API** để tiếp nhận các yêu cầu nghiệp vụ từ User Frontend và Admin Frontend.

API Gateway đóng vai trò là cổng giao tiếp giữa frontend và các Lambda Function backend. Khi frontend gửi yêu cầu, API Gateway kiểm tra thông tin xác thực, áp dụng các chính sách giới hạn truy cập và chuyển yêu cầu đến Lambda Function phù hợp.

REST API của hệ thống được tạo và cấu hình bằng Terraform trong module:

```text
infra/07-api
```

Tên REST API được tạo theo cấu trúc:

```text
<name-prefix>-control-plane
```

Với tiền tố hiện tại, API có tên:

```text
la-control-plane
```

## Vai trò của Amazon API Gateway

Trong hệ thống Live Auction, Amazon API Gateway được sử dụng để:

* Cung cấp REST API cho User Frontend.
* Cung cấp REST API cho Admin Frontend.
* Định tuyến yêu cầu đến Lambda Function phù hợp.
* Xác thực Cognito JWT của tài khoản gửi yêu cầu.
* Yêu cầu API Key đối với các API đã cấu hình.
* Áp dụng giới hạn tốc độ truy cập — throttling.
* Áp dụng quota cho API Key.
* Cấu hình CORS cho các frontend được phép truy cập.
* Ghi Access Log vào Amazon CloudWatch.
* Thu thập Metrics phục vụ giám sát.
* Lưu cache cho một số API truy vấn.
* Trả phản hồi lỗi thống nhất cho frontend.

## Luồng xử lý REST API

Luồng xử lý yêu cầu REST API được thực hiện như sau:

1. User Frontend hoặc Admin Frontend gửi yêu cầu HTTPS đến API Gateway.
2. Yêu cầu chứa Cognito token trong header `Authorization`.
3. Yêu cầu chứa API Key trong header `x-api-key`.
4. API Gateway kiểm tra Cognito token thông qua Cognito User Pool Authorizer.
5. API Gateway kiểm tra API Key và Usage Plan.
6. API Gateway kiểm tra Method, Resource và các tham số của yêu cầu.
7. Yêu cầu được chuyển đến Lambda Function thông qua Lambda Proxy Integration.
8. Lambda xử lý nghiệp vụ và truy cập DynamoDB hoặc các dịch vụ liên quan.
9. Lambda trả kết quả về API Gateway.
10. API Gateway trả phản hồi HTTP cho frontend.

Luồng tổng quát:

```text
User/Admin Frontend
        ↓
Amazon API Gateway REST API
        ↓
Cognito Authorizer + API Key
        ↓
Lambda Proxy Integration
        ↓
AWS Lambda
        ↓
Amazon DynamoDB và các dịch vụ liên quan
```

## Cấu trúc REST API

REST API sử dụng đường dẫn cơ sở:

```text
/api/v1
```

Các API được chia thành những nhóm nghiệp vụ chính:

| Nhóm API                | Mục đích                                                        |
| ----------------------- | --------------------------------------------------------------- |
| **User API**            | Lấy thông tin tài khoản hiện tại.                               |
| **Auction Session API** | Tạo, xem và quản lý phiên đấu giá.                              |
| **Auction Item API**    | Xem, thêm và quản lý vật phẩm.                                  |
| **Bid API**             | Xem lịch sử hoặc dữ liệu đặt giá của người dùng.                |
| **Category API**        | Xem danh mục sản phẩm.                                          |
| **Admin API**           | Quản lý người dùng, tài khoản Admin, danh mục và phiên đấu giá. |

## Các API dành cho người dùng

### Thông tin tài khoản

```text
GET /api/v1/users/me
```

API trả thông tin tài khoản hiện tại dựa trên Cognito token.

### Phiên đấu giá

```text
GET  /api/v1/auction-sessions
POST /api/v1/auction-sessions
GET  /api/v1/auction-sessions/mine
GET  /api/v1/auction-sessions/{session_id}
PUT  /api/v1/auction-sessions/{session_id}/rules
POST /api/v1/auction-sessions/{session_id}/items
POST /api/v1/auction-sessions/{session_id}/schedule
```

Nhóm API này hỗ trợ:

* Xem danh sách phiên đấu giá.
* Tạo phiên đấu giá.
* Xem các phiên do người dùng tạo.
* Xem chi tiết phiên.
* Cập nhật quy tắc phiên.
* Thêm một hoặc nhiều vật phẩm vào phiên.
* Thiết lập lịch hoạt động của phiên.

### Vật phẩm đấu giá

```text
GET  /api/v1/auction-items
GET  /api/v1/auction-items/{item_id}
POST /api/v1/auction-items/{item_id}/images/presign
```

Nhóm API này hỗ trợ:

* Xem danh sách vật phẩm.
* Xem thông tin chi tiết vật phẩm.
* Tạo thông tin cần thiết để tải hình ảnh vật phẩm lên Amazon S3.

### Lịch sử đặt giá

```text
GET /api/v1/bids/my
```

API được sử dụng để truy vấn dữ liệu đặt giá liên quan đến người dùng hiện tại.

### Danh mục sản phẩm

```text
GET /api/v1/categories
GET /api/v1/categories/{category_id}
```

Nhóm API này cho phép frontend lấy danh sách và thông tin chi tiết của danh mục sản phẩm.

## Các API dành cho quản trị viên

### Quản lý phiên đấu giá

```text
GET  /api/v1/admin/auction-sessions
GET  /api/v1/admin/auction-sessions/{session_id}
POST /api/v1/admin/auction-sessions/{session_id}/approve
POST /api/v1/admin/auction-sessions/{session_id}/reject
POST /api/v1/admin/auction-sessions/{session_id}/cancel
POST /api/v1/admin/auction-sessions/{session_id}/close
```

Quản trị viên có thể:

* Xem danh sách phiên cần quản lý.
* Xem chi tiết phiên.
* Duyệt phiên đấu giá.
* Từ chối phiên.
* Hủy phiên.
* Đóng phiên.

### Quản lý vật phẩm

```text
POST /api/v1/admin/items/{item_id}/pause
POST /api/v1/admin/items/{item_id}/resume
POST /api/v1/admin/items/{item_id}/approve
POST /api/v1/admin/items/{item_id}/close
POST /api/v1/admin/items/{item_id}/cancel
```

Nhóm API này được sử dụng để quản lý trạng thái vật phẩm trong quá trình đấu giá.

### Quản lý tài khoản người dùng

```text
GET   /api/v1/admin/users
GET   /api/v1/admin/users/{user_id}
PATCH /api/v1/admin/users/{user_id}/status
```

Quản trị viên có thể:

* Xem danh sách tài khoản người dùng.
* Xem chi tiết tài khoản.
* Thay đổi trạng thái tài khoản.

### Quản lý tài khoản Admin

```text
GET   /api/v1/admin/admin-accounts
POST  /api/v1/admin/admin-accounts
PATCH /api/v1/admin/admin-accounts/{user_id}/status
POST  /api/v1/admin/admin-accounts/{user_id}/reset-invitation
```

Nhóm API này hỗ trợ:

* Xem danh sách Admin.
* Tạo thêm tài khoản Admin.
* Thay đổi trạng thái tài khoản Admin.
* Gửi lại thông tin mời kích hoạt tài khoản.

### Quản lý danh mục

```text
GET   /api/v1/admin/categories
POST  /api/v1/admin/categories
PATCH /api/v1/admin/categories/{category_id}
POST  /api/v1/admin/categories/{category_id}/archive
```

Quản trị viên có thể:

* Xem danh sách danh mục.
* Tạo danh mục mới.
* Cập nhật danh mục.
* Lưu trữ danh mục không còn sử dụng.

### Dashboard và Audit Event

```text
GET /api/v1/admin/dashboard
GET /api/v1/admin/audit-events
```

Các API này cung cấp dữ liệu tổng quan và lịch sử thao tác quản trị.

## Lambda Integration

REST API sử dụng Lambda Proxy Integration:

```text
Integration type: AWS_PROXY
Integration HTTP method: POST
```

Mỗi nhóm API được tích hợp với Lambda Function phù hợp:

| Lambda Function        | Nhóm API                                                          |
| ---------------------- | ----------------------------------------------------------------- |
| **la-session-service** | Tạo phiên, cập nhật quy tắc và quản lý thông tin phiên.           |
| **la-item-service**    | Thêm vật phẩm và xử lý yêu cầu tải hình ảnh.                      |
| **la-query-service**   | Các API truy vấn User, phiên, vật phẩm, danh mục và lượt đặt giá. |
| **la-admin-command**   | Các API quản trị, duyệt phiên, quản lý tài khoản và danh mục.     |

API Gateway sử dụng Lambda Permission để được phép gọi các Function tương ứng.

## Cognito User Pool Authorizer

REST API sử dụng Cognito User Pool Authorizer để kiểm tra token trong header:

```text
Authorization
```

Cơ chế xác thực được thực hiện như sau:

1. Người dùng đăng nhập thông qua Amazon Cognito.
2. Cognito trả token về frontend.
3. Frontend gửi token trong header `Authorization`.
4. API Gateway kiểm tra token với Cognito User Pool.
5. Nếu token hợp lệ, yêu cầu được chuyển đến Lambda.
6. Nếu token không hợp lệ hoặc hết hạn, API Gateway từ chối yêu cầu.

Cognito Authorizer kiểm tra danh tính của tài khoản. Đối với API quản trị, Lambda tiếp tục kiểm tra thông tin nhóm `admin` trước khi thực hiện nghiệp vụ.

## API Key và Usage Plan

Ngoài Cognito token, REST API yêu cầu API Key trong header:

```text
x-api-key
```

API Key được liên kết với một Usage Plan.

Usage Plan của hệ thống được cấu hình mặc định:

```text
Rate limit: 50 requests/second
Burst limit: 100 requests
Daily quota: 10,000 requests
```

Usage Plan giúp:

* Giới hạn số lượng yêu cầu gửi đến API.
* Hạn chế việc gọi API quá mức.
* Giảm nguy cơ lạm dụng endpoint.
* Theo dõi lượng sử dụng theo API Key.
* Bảo vệ Lambda và các dịch vụ backend.


## Cấu hình CORS

REST API cấu hình CORS để cho phép User Frontend và Admin Frontend gửi yêu cầu đến API Gateway.

Các header được cho phép gồm:

```text
Content-Type
Authorization
X-Api-Key
```

Các phương thức được cho phép gồm:

```text
GET
POST
PUT
PATCH
OPTIONS
```

Phương thức `OPTIONS` được sử dụng để xử lý CORS Preflight Request trước khi trình duyệt gửi yêu cầu chính.

CORS chỉ cho phép những origin frontend được cấu hình trong hệ thống.

## Stage triển khai

REST API được triển khai vào Stage:

```text
prod
```

Invoke URL có cấu trúc:

```text
https://<rest-api-id>.execute-api.ap-southeast-1.amazonaws.com/prod
```

Stage `prod` được cấu hình:

* Access Logging.
* CloudWatch Metrics.
* Logging Level là `INFO`.
* Throttling.
* API Cache.
* Cache Data Encryption.
* Usage Plan và API Key.

## API Cache

Stage `prod` bật API Gateway Cache với kích thước mặc định:

```text
0.5 GB
```

Cache được áp dụng cho một số API truy vấn:

```text
GET /api/v1/auction-sessions
GET /api/v1/auction-items
```

Thời gian lưu cache mặc định:

```text
60 seconds
```

Các Cache Key có thể bao gồm:

* `status`
* `pageSize`
* `cursor`
* `sessionId`
* `categoryId`

Việc sử dụng Cache giúp giảm số lần gọi Lambda đối với những truy vấn được thực hiện thường xuyên.

## Access Logging và Metrics

API Gateway ghi Access Log vào Amazon CloudWatch.

Log có thể chứa:

* Request ID.
* Source IP.
* HTTP Method.
* Resource Path.
* Status Code.
* Response Length.
* Response Latency.
* Integration Latency.
* Cognito Subject.

CloudWatch Metrics hỗ trợ theo dõi:

* Count.
* Latency.
* Integration Latency.
* Lỗi `4XX`.
* Lỗi `5XX`.
* Cache Hit.
* Cache Miss.

Không nên bật Data Trace trong môi trường có dữ liệu thật nếu log có khả năng chứa token hoặc nội dung yêu cầu nhạy cảm.

## Gateway Response

REST API cấu hình Gateway Response cho:

```text
DEFAULT_4XX
DEFAULT_5XX
```

Các phản hồi lỗi cũng được bổ sung CORS Header để frontend có thể đọc nội dung lỗi.

Cấu hình này giúp phản hồi lỗi giữa các API nhất quán hơn.

## Kiểm tra REST API trên AWS Management Console

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

### Bước 2: Kiểm tra danh sách API

Trong giao diện API Gateway, mở:

```text
APIs
```

Tìm REST API:

```text
la-control-plane
```

Kiểm tra:

* API Name.
* API ID.
* API Type là REST.
* Endpoint Type.
* Created Date.
* Region.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-api-list.png"
    title="Hình 5.4.5.1: Các API của hệ thống Live Auction trên Amazon API Gateway"
    width="100%"
>}}

### Bước 3: Kiểm tra Resources và Methods

Chọn REST API:

```text
la-control-plane
```

Mở:

```text
Resources
```

Kiểm tra cây Resource của API.

Các nội dung cần xác nhận:

* Resource Path bắt đầu bằng `/api/v1`.
* Các nhóm `auction-sessions`, `auction-items`, `users`, `categories` và `admin`.
* Các HTTP Method như `GET`, `POST`, `PUT`, `PATCH` và `OPTIONS`.
* Resource có Path Parameter như `{session_id}`, `{item_id}`, `{user_id}` và `{category_id}`.
* Các Resource đều có phương thức `OPTIONS` để xử lý CORS Preflight Request.
* Các Resource quản trị được đặt dưới đường dẫn `/api/v1/admin`.

Cấu trúc đầy đủ của REST API được triển khai như sau:

```text
/
└── api
    └── v1
        ├── users
        │   └── me
        │       ├── GET
        │       └── OPTIONS
        │
        ├── auction-sessions
        │   ├── GET
        │   ├── POST
        │   ├── OPTIONS
        │   ├── mine
        │   │   ├── GET
        │   │   └── OPTIONS
        │   └── {session_id}
        │       ├── GET
        │       ├── OPTIONS
        │       ├── items
        │       │   ├── POST
        │       │   └── OPTIONS
        │       ├── rules
        │       │   ├── PUT
        │       │   └── OPTIONS
        │       └── schedule
        │           ├── POST
        │           └── OPTIONS
        │
        ├── auction-items
        │   ├── GET
        │   ├── OPTIONS
        │   └── {item_id}
        │       ├── GET
        │       ├── OPTIONS
        │       └── images
        │           └── presign
        │               ├── POST
        │               └── OPTIONS
        │
        ├── bids
        │   └── my
        │       ├── GET
        │       └── OPTIONS
        │
        ├── categories
        │   ├── GET
        │   ├── OPTIONS
        │   └── {category_id}
        │       ├── GET
        │       └── OPTIONS
        │
        └── admin
            ├── dashboard
            │   ├── GET
            │   └── OPTIONS
            │
            ├── audit-events
            │   ├── GET
            │   └── OPTIONS
            │
            ├── users
            │   ├── GET
            │   ├── OPTIONS
            │   └── {user_id}
            │       ├── GET
            │       ├── OPTIONS
            │       └── status
            │           ├── PATCH
            │           └── OPTIONS
            │
            ├── admin-accounts
            │   ├── GET
            │   ├── POST
            │   ├── OPTIONS
            │   └── {user_id}
            │       ├── status
            │       │   ├── PATCH
            │       │   └── OPTIONS
            │       └── reset-invitation
            │           ├── POST
            │           └── OPTIONS
            │
            ├── categories
            │   ├── GET
            │   ├── POST
            │   ├── OPTIONS
            │   └── {category_id}
            │       ├── PATCH
            │       ├── OPTIONS
            │       └── archive
            │           ├── POST
            │           └── OPTIONS
            │
            ├── auction-sessions
            │   ├── GET
            │   ├── OPTIONS
            │   └── {session_id}
            │       ├── GET
            │       ├── OPTIONS
            │       ├── approve
            │       │   ├── POST
            │       │   └── OPTIONS
            │       ├── reject
            │       │   ├── POST
            │       │   └── OPTIONS
            │       ├── cancel
            │       │   ├── POST
            │       │   └── OPTIONS
            │       └── close
            │           ├── POST
            │           └── OPTIONS
            │
            └── items
                └── {item_id}
                    ├── approve
                    │   ├── POST
                    │   └── OPTIONS
                    ├── cancel
                    │   ├── POST
                    │   └── OPTIONS
                    ├── close
                    │   ├── POST
                    │   └── OPTIONS
                    ├── pause
                    │   ├── POST
                    │   └── OPTIONS
                    └── resume
                        ├── POST
                        └── OPTIONS
```

Cây Resource cho thấy REST API đã được chia thành hai nhóm chính:

* **User API:** Cung cấp chức năng xem thông tin cá nhân, xem và tạo phiên đấu giá, quản lý vật phẩm, xem lượt đặt giá và danh mục.
* **Admin API:** Cung cấp chức năng quản lý tài khoản người dùng, tài khoản Admin, danh mục, phiên đấu giá, vật phẩm và Audit Event.

Do cây Resource trên AWS Management Console có nhiều đường dẫn và không thể hiển thị đầy đủ trong một màn hình, hình dưới đây chỉ minh họa một phần cấu trúc đã được triển khai. Danh sách đầy đủ được thể hiện trong sơ đồ cây phía trên.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-resources-methods.png"
    title="Hình 5.4.5.2: Một phần Resource và HTTP Method của REST API trên AWS Management Console"
    width="50%"
>}}

### Bước 4: Kiểm tra Method Execution

Chọn một Method, chẳng hạn:

```text
GET /api/v1/auction-sessions
```

Kiểm tra:

* Method Request.
* Integration Request.
* Method Response.
* Integration Response.
* Authorization.
* API Key Required.
* Lambda Function được tích hợp.

Các giá trị cần xác nhận:

```text
Authorization: Cognito User Pool Authorizer
API Key Required: True
Integration type: Lambda Function
Lambda Proxy Integration: Enabled
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-method-execution.png"
    title="Hình 5.4.5.3: Cấu hình Method và Lambda Integration của REST API"
    width="100%"
>}}

### Bước 5: Kiểm tra Cognito Authorizer

Tại menu của REST API, chọn:

```text
Authorizers
```

Kiểm tra Cognito Authorizer.

Các nội dung cần xác nhận:

* Authorizer Type là Cognito.
* Cognito User Pool đã được liên kết.
* Token Source là `Authorization`.
* Region của User Pool.
* Authorizer đang được sử dụng bởi các Method.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-cognito-authorizer.png"
    title="Hình 5.4.5.4: Cognito User Pool Authorizer của REST API"
    width="100%"
>}}

### Bước 6: Kiểm tra Stage prod

Tại menu của REST API, chọn:

```text
Stages → prod
```

Kiểm tra:

* Invoke URL.
* Deployment ID.
* Last Updated Date.
* Cache Cluster.
* Logging.
* CloudWatch Metrics.
* Throttling.
* Method Settings.

Không sử dụng chức năng chỉnh sửa trực tiếp trên Console vì Stage đang được quản lý bằng Terraform.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-prod-stage.png"
    title="Hình 5.4.5.5: Stage prod của Amazon API Gateway REST API"
    width="100%"
>}}

### Bước 7: Kiểm tra API Key

Quay lại giao diện API Gateway và mở:

```text
API keys
```

Tìm API Key của dự án.

Kiểm tra:

* API Key Name.
* Enabled state.
* Created Date.
* Usage Plan được liên kết.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-api-key.png"
    title="Hình 5.4.5.6: API Key được cấu hình cho REST API"
    width="100%"
>}}

### Bước 8: Kiểm tra Usage Plan

Trong API Gateway, mở:

```text
Usage plans
```

Chọn Usage Plan của hệ thống và kiểm tra:

* Rate Limit.
* Burst Limit.
* Quota.
* API Stage được liên kết.
* API Key được liên kết.

Các giá trị mặc định của dự án:

```text
Rate limit: 50 requests/second
Burst limit: 100 requests
Quota: 10,000 requests/day
```



{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-usage-plan.png"
    title="Hình 5.4.5.7: Throttling và Quota của API Gateway Usage Plan"
    width="100%"
>}}

### Bước 9: Kiểm tra CloudWatch Logs và Metrics

Trong REST API, chọn:

```text
Dashboard
```

Chọn Stage:

```text
prod
```

Sau đó chọn khoảng thời gian có dữ liệu và kiểm tra các Metrics:

* Số lượng API Call.
* Latency.
* Integration Latency.
* Lỗi `4XX`.
* Lỗi `5XX` nếu có.
* Cache Hit và Cache Miss nếu có dữ liệu.

Dashboard cho thấy REST API đã tiếp nhận yêu cầu thực tế trong quá trình vận hành. Biểu đồ Latency và Integration Latency hỗ trợ theo dõi thời gian phản hồi của API Gateway và Lambda Integration.

Biểu đồ `4XX error` ghi nhận các yêu cầu bị từ chối do lỗi phía client, chẳng hạn như thiếu Cognito token, thiếu API Key, đường dẫn không hợp lệ hoặc tài khoản không có quyền phù hợp.

Access Log của Stage `prod` được lưu trong CloudWatch Log Group đã cấu hình tại phần **Logs and tracing** của Stage.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-cloudwatch-metrics.png"
    title="Hình 5.4.5.8: API Call, Latency và Error Metrics của REST API"
    width="100%"
>}}

## Kiểm tra phản hồi API

Sau khi kiểm tra cấu hình trên AWS Console, có thể gửi một yêu cầu thử đến Invoke URL.

Yêu cầu hợp lệ cần chứa:

```text
Authorization: <cognito-token>
x-api-key: <api-key>
Content-Type: application/json
```

Kết quả mong đợi:

* Thiếu Cognito token: API trả lỗi xác thực.
* Token không hợp lệ: API từ chối yêu cầu.
* Thiếu API Key: API từ chối yêu cầu.
* Đầy đủ token và API Key: API chuyển yêu cầu đến Lambda.
* Tài khoản User gọi API Admin: Lambda từ chối vì không có quyền.
* Tài khoản Admin hợp lệ: API xử lý yêu cầu quản trị.

Không đưa token và API Key thật vào mã nguồn Hugo hoặc ảnh chụp báo cáo.

Việc kiểm thử đầy đủ từng API được trình bày tại mục **5.5 — Kiểm thử hệ thống**.

## Kết quả

Sau khi kiểm tra trực tiếp trên AWS Management Console, nhóm ghi nhận:

* REST API `la-control-plane` đã được Terraform tạo thành công.
* Các Resource và HTTP Method đã được cấu hình theo đường dẫn `/api/v1`.
* Các API User và Admin được tổ chức thành những nhóm riêng.
* Các Method được tích hợp với Lambda Function phù hợp.
* Cognito User Pool Authorizer đã được cấu hình.
* Các Method yêu cầu Cognito token và API Key.
* Stage `prod` đã được triển khai.
* CORS cho phép hai frontend gửi yêu cầu đến API.
* API Cache được bật cho các API truy vấn cần thiết.
* Access Logging và CloudWatch Metrics đã được cấu hình.
* Throttling và Daily Quota được áp dụng thông qua Usage Plan.
* Gateway Response đã được cấu hình cho lỗi `4XX` và `5XX`.
* REST API đã sẵn sàng tiếp nhận yêu cầu nghiệp vụ từ User Frontend và Admin Frontend.

Việc cấu hình và kiểm tra API Gateway WebSocket được trình bày tại mục **5.4.6 — API Gateway WebSocket**.