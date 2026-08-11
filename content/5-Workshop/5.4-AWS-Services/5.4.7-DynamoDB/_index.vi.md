---
title: "Amazon DynamoDB"
date: 2026-07-27
weight: 7
chapter: false
pre: " <b> 5.4.7. </b> "
---

## Tổng quan

Hệ thống **Live Auction** sử dụng **Amazon DynamoDB** để lưu trữ dữ liệu nghiệp vụ, trạng thái đấu giá, lịch sử đặt giá, kết nối WebSocket, danh mục sản phẩm và lịch sử thao tác quản trị.

DynamoDB là dịch vụ cơ sở dữ liệu NoSQL serverless của AWS. Dịch vụ có khả năng mở rộng tự động và phù hợp với hệ thống cần truy cập dữ liệu nhanh theo thời gian thực.

Các bảng DynamoDB được tạo và cấu hình bằng Terraform trong module:

```text
infra/04-data
```

Hệ thống sử dụng chế độ thanh toán:

```text
PAY_PER_REQUEST
```

Với chế độ này, nhóm không cần cấu hình trước Read Capacity Unit và Write Capacity Unit. DynamoDB tính chi phí dựa trên số lượng yêu cầu đọc và ghi thực tế.


## Vai trò của Amazon DynamoDB

Trong hệ thống Live Auction, DynamoDB được sử dụng để:

* Lưu trạng thái hiện tại của vật phẩm đấu giá.
* Lưu lịch sử các sự kiện đặt giá.
* Lưu Connection ID của WebSocket Client.
* Liên kết người dùng với bí danh hiển thị trong từng vật phẩm.
* Ngăn xử lý trùng lặp một yêu cầu.
* Lưu dữ liệu phiên đấu giá và vật phẩm.
* Lưu danh mục sản phẩm.
* Lưu lịch sử thao tác của quản trị viên.
* Hỗ trợ truy vấn dữ liệu theo nhiều điều kiện thông qua Global Secondary Index.
* Tự động xóa dữ liệu tạm thời thông qua Time to Live.
* Hỗ trợ khôi phục dữ liệu thông qua Point-in-Time Recovery.
* Cung cấp DynamoDB Stream cho các bảng cần xử lý sự kiện thay đổi.

## Các bảng DynamoDB của hệ thống

Hệ thống sử dụng tám bảng DynamoDB chính:

| DynamoDB Table               | Partition Key | Sort Key        | Vai trò                                                        |
| ---------------------------- | ------------- | --------------- | -------------------------------------------------------------- |
| **la_item_auction_state**    | `item_id`     | Không có        | Lưu trạng thái hiện tại của vật phẩm đấu giá.                  |
| **la_bid_events**            | `item_id`     | `sk`            | Lưu lịch sử và sự kiện đặt giá.                                |
| **la_websocket_connections** | `item_id`     | `connection_id` | Lưu Connection ID của người dùng theo dõi vật phẩm.            |
| **la_item_bidder_aliases**   | `item_id`     | `user_id`       | Lưu bí danh người đặt giá trong từng vật phẩm.                 |
| **la_idempotency**           | `id`          | Không có        | Ngăn cùng một yêu cầu được xử lý nhiều lần.                    |
| **la_auction_catalog**       | `pk`          | `sk`            | Lưu dữ liệu phiên đấu giá, vật phẩm và các thực thể liên quan. |
| **la_category_catalog**      | `category_id` | Không có        | Lưu danh mục sản phẩm.                                         |
| **la_admin_audit_events**    | `pk`          | `sk`            | Lưu lịch sử thao tác quản trị.                                 |

Tên bảng sử dụng dấu gạch dưới theo quy ước:

```text
<name-prefix>_<table-purpose>
```

## Item Auction State Table

Bảng:

```text
la_item_auction_state
```

được sử dụng để lưu trạng thái hiện tại của từng vật phẩm đấu giá.

Cấu hình khóa chính:

```text
Partition key: item_id
Attribute type: String
```

Dữ liệu có thể bao gồm:

* Item ID.
* Giá hiện tại.
* Người đặt giá gần nhất.
* Trạng thái vật phẩm.
* Thời gian cập nhật gần nhất.
* Version hoặc thông tin kiểm soát cập nhật.
* Thời gian bắt đầu và kết thúc.

Bảng được bật DynamoDB Stream:

```text
Stream: Enabled
View type: NEW_AND_OLD_IMAGES
```

`NEW_AND_OLD_IMAGES` cho phép sự kiện Stream chứa cả dữ liệu trước và sau khi thay đổi.

Bảng cũng được bật:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## Bid Events Table

Bảng:

```text
la_bid_events
```

lưu các sự kiện đặt giá của từng vật phẩm.

Cấu hình khóa chính:

```text
Partition key: item_id
Sort key: sk
```

Sort Key `sk` giúp sắp xếp các sự kiện của cùng một vật phẩm theo thứ tự được thiết kế trong hệ thống.

Dữ liệu có thể bao gồm:

* Item ID.
* Bidder Subject.
* Giá đặt.
* Thời điểm đặt giá.
* Trạng thái xử lý.
* Event ID.
* Idempotency information.

Bảng được bật DynamoDB Stream:

```text
Stream: Enabled
View type: NEW_IMAGE
```

`NEW_IMAGE` cho phép Stream chứa dữ liệu mới sau khi sự kiện được ghi.

Bảng có Global Secondary Index:

```text
Index name: bidder_sub-sk-index
Partition key: bidder_sub
Sort key: sk
Projection: ALL
```

Index này hỗ trợ truy vấn các lượt đặt giá theo người dùng.

Bảng cũng được bật:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## WebSocket Connections Table

Bảng:

```text
la_websocket_connections
```

lưu thông tin các WebSocket Client đang theo dõi vật phẩm đấu giá.

Cấu hình khóa chính:

```text
Partition key: item_id
Sort key: connection_id
```

Cấu trúc khóa cho phép hệ thống truy vấn các Connection ID đang theo dõi một vật phẩm.

Dữ liệu có thể bao gồm:

* Item ID.
* Connection ID.
* User ID hoặc Cognito Subject.
* Thời điểm kết nối.
* Thời điểm hết hạn.
* Thông tin phòng đấu giá.

Bảng được bật Time to Live:

```text
TTL attribute: ttl
TTL status: Enabled
```

TTL giúp DynamoDB tự động xóa dữ liệu kết nối đã hết hạn.

Khi API Gateway trả lỗi `410 Gone`, Lambda cũng có thể chủ động xóa Connection ID không còn hợp lệ.

## Item Bidder Aliases Table

Bảng:

```text
la_item_bidder_aliases
```

lưu bí danh của người đặt giá trong từng vật phẩm.

Cấu hình khóa chính:

```text
Partition key: item_id
Sort key: user_id
```

Bảng giúp:

* Liên kết người dùng với một vật phẩm.
* Hiển thị bí danh thay vì thông tin tài khoản thật.
* Hạn chế công khai danh tính người đặt giá.
* Giữ bí danh nhất quán trong cùng một vật phẩm.

Bảng được bật Server-side Encryption.

## Idempotency Table

Bảng:

```text
la_idempotency
```

được sử dụng để ngăn một yêu cầu bị xử lý nhiều lần.

Cấu hình khóa chính:

```text
Partition key: id
```

Ví dụ, khi một yêu cầu đặt giá được gửi lại do lỗi mạng, hệ thống có thể kiểm tra Idempotency ID trước khi xử lý.

Bảng được bật TTL:

```text
TTL attribute: expiration
TTL status: Enabled
```

Các bản ghi Idempotency cũ sẽ được DynamoDB tự động xóa sau khi hết hạn.

Cơ chế này giúp:

* Hạn chế ghi trùng dữ liệu.
* Hạn chế xử lý cùng một thông điệp nhiều lần.
* Tăng tính an toàn khi Lambda thực hiện retry.
* Hỗ trợ xử lý thông điệp từ SQS FIFO.

## Auction Catalog Table

Bảng:

```text
la_auction_catalog
```

lưu dữ liệu nghiệp vụ của phiên đấu giá, vật phẩm và các thực thể liên quan.

Cấu hình khóa chính:

```text
Partition key: pk
Sort key: sk
```

Bảng sử dụng thiết kế khóa tổng hợp để lưu nhiều loại thực thể trong cùng một bảng.

Bảng có hai Global Secondary Index:

```text
Index name: gsi1
Partition key: gsi1pk
Sort key: gsi1sk
Projection: ALL
```

```text
Index name: gsi2
Partition key: gsi2pk
Sort key: gsi2sk
Projection: ALL
```

Các GSI hỗ trợ những cách truy vấn khác ngoài khóa chính, chẳng hạn:

* Truy vấn phiên theo trạng thái.
* Truy vấn phiên của một người dùng.
* Truy vấn vật phẩm thuộc một phiên.
* Sắp xếp dữ liệu theo thời gian.
* Truy vấn các thực thể theo quan hệ nghiệp vụ.

Bảng được bật:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## Category Catalog Table

Bảng:

```text
la_category_catalog
```

lưu danh mục sản phẩm được sử dụng khi tạo vật phẩm đấu giá.

Cấu hình khóa chính:

```text
Partition key: category_id
```

Bảng có hai Global Secondary Index.

Index truy vấn theo slug:

```text
Index name: slug-index
Partition key: slug
Projection: ALL
```

Index truy vấn theo trạng thái:

```text
Index name: status-index
Partition key: status
Sort key: created_at
Projection: ALL
```

Các index hỗ trợ:

* Tìm danh mục bằng slug.
* Lấy danh mục theo trạng thái.
* Sắp xếp danh mục theo thời gian tạo.
* Phân trang danh sách danh mục.

Bảng được bật:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## Admin Audit Events Table

Bảng:

```text
la_admin_audit_events
```

lưu lịch sử thao tác của quản trị viên.

Cấu hình khóa chính:

```text
Partition key: pk
Sort key: sk
```

Dữ liệu Audit Event có thể bao gồm:

* Admin thực hiện thao tác.
* Loại hành động.
* Tài nguyên bị tác động.
* Kết quả thao tác.
* Thời gian thực hiện.
* Request ID.
* Dữ liệu mô tả liên quan.

Bảng có hai Global Secondary Index.

Truy vấn theo người thực hiện:

```text
Index name: actor-index
Partition key: actor_sub
Sort key: sk
Projection: ALL
```

Truy vấn theo tài nguyên:

```text
Index name: resource-index
Partition key: resource_key
Sort key: sk
Projection: ALL
```

Bảng được bật TTL:

```text
TTL attribute: expires_at
TTL status: Enabled
```

Bảng cũng được bật:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## Global Secondary Index

Global Secondary Index — GSI cho phép truy vấn dữ liệu bằng khóa khác với khóa chính của bảng.

Các GSI chính trong hệ thống gồm:

| Table                     | Index                           |
| ------------------------- | ------------------------------- |
| **la_bid_events**         | `bidder_sub-sk-index`           |
| **la_auction_catalog**    | `gsi1`, `gsi2`                  |
| **la_category_catalog**   | `slug-index`, `status-index`    |
| **la_admin_audit_events** | `actor-index`, `resource-index` |

Các GSI sử dụng:

```text
Projection type: ALL
```

Điều này cho phép toàn bộ thuộc tính của bản ghi được sao chép vào Index để phục vụ truy vấn.

## Time to Live

Time to Live — TTL cho phép DynamoDB tự động xóa Item sau khi thuộc tính thời gian đạt đến thời điểm hết hạn.

Các bảng sử dụng TTL:

| Table                        | TTL attribute | Mục đích                                         |
| ---------------------------- | ------------- | ------------------------------------------------ |
| **la_websocket_connections** | `ttl`         | Xóa Connection đã hết hạn.                       |
| **la_idempotency**           | `expiration`  | Xóa bản ghi chống trùng lặp không còn cần thiết. |
| **la_admin_audit_events**    | `expires_at`  | Xóa Audit Event khi hết thời gian lưu trữ.       |

TTL giúp hạn chế dữ liệu cũ tồn tại lâu dài và giảm công việc dọn dẹp thủ công.

Việc xóa Item bằng TTL không diễn ra ngay lập tức tại đúng thời điểm hết hạn. DynamoDB thực hiện xóa bất đồng bộ sau đó.

## Point-in-Time Recovery

Point-in-Time Recovery — PITR cho phép khôi phục bảng DynamoDB về một thời điểm trong khoảng thời gian được AWS hỗ trợ.

PITR được bật cho các bảng quan trọng:

```text
la_item_auction_state
la_bid_events
la_auction_catalog
la_category_catalog
la_admin_audit_events
```

Cấu hình này hỗ trợ:

* Khôi phục dữ liệu sau khi xóa nhầm.
* Khôi phục dữ liệu sau khi ghi sai.
* Hạn chế ảnh hưởng khi ứng dụng gây lỗi dữ liệu.
* Tăng khả năng phục hồi của hệ thống.

## Server-side Encryption

Các bảng được bật Server-side Encryption.

DynamoDB tự động mã hóa:

* Dữ liệu bảng.
* Index.
* DynamoDB Stream.
* Bản sao lưu.

Mã hóa giúp bảo vệ dữ liệu khi được lưu trữ trên AWS.

## DynamoDB Streams

DynamoDB Stream được bật cho hai bảng:

| Table                     | Stream View Type     |
| ------------------------- | -------------------- |
| **la_item_auction_state** | `NEW_AND_OLD_IMAGES` |
| **la_bid_events**         | `NEW_IMAGE`          |

Stream lưu lại thông tin thay đổi của Item trong bảng.

Dữ liệu Stream có thể được sử dụng để:

* Phát hiện giá đấu thay đổi.
* Kích hoạt xử lý bất đồng bộ.
* Gửi cập nhật đến WebSocket Client.
* Theo dõi lịch sử thay đổi.
* Tích hợp thêm các chức năng xử lý sự kiện.

## Kiểm tra DynamoDB trên AWS Management Console

### Bước 1: Truy cập Amazon DynamoDB

Đăng nhập vào **AWS Management Console**.

Tại thanh tìm kiếm, nhập:

```text
DynamoDB
```

Chọn **DynamoDB**.

Bảo đảm Region đang được chọn là:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

### Bước 2: Kiểm tra danh sách Table

Tại menu bên trái, chọn:

```text
Tables
```

Trong ô tìm kiếm, nhập:

```text
la_
```

Kiểm tra tám bảng của hệ thống.

Các nội dung cần xác nhận:

* Table Name.
* Status.
* Partition Key.
* Sort Key.
* Billing Mode.
* Region.
* Các bảng cần thiết đã được triển khai đầy đủ hay chưa.

Trạng thái của các bảng phải là:

```text
Active
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-table-list.png"
    title="Hình 5.4.7.1: Các bảng DynamoDB của hệ thống Live Auction"
    width="100%"
>}}

### Bước 3: Kiểm tra thông tin Table

Chọn bảng:

```text
la_auction_catalog
```

Mở tab:

```text
Setting -> General information 
```

Kiểm tra:

* Table Status.
* Partition Key.
* Sort Key.
* Billing Mode.
* Table ARN.
* Point-in-Time Recovery.
* Encryption.
* Item Count.
* Table Size.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-table-overview.png"
    title="Hình 5.4.7.2: Thông tin bảng la_auction_catalog"
    width="100%"
>}}

### Bước 4: Kiểm tra dữ liệu trong Table

Trong bảng `la_auction_catalog`, chọn:

```text
Explore items
```

Chọn **Run** để hiển thị một phần dữ liệu.

Kiểm tra:

* Partition Key `pk`.
* Sort Key `sk`.
* Các thuộc tính dữ liệu.
* Các Item thuộc phiên hoặc vật phẩm.
* Dữ liệu đã được Lambda ghi vào bảng.

Không chỉnh sửa hoặc xóa Item trực tiếp trên Console.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-auction-catalog-items.png"
    title="Hình 5.4.7.3: Dữ liệu phiên và vật phẩm trong bảng la_auction_catalog"
    width="100%"
>}}

### Bước 5: Kiểm tra Global Secondary Index

Trong bảng `la_auction_catalog`, mở:

```text
Indexes
```

Kiểm tra:

```text
gsi1
gsi2
```

Các nội dung cần xác nhận:

* Index Name.
* Partition Key.
* Sort Key.
* Index Status.
* Projection Type.
* Item Count.
* Index Size.

Trạng thái Index cần là:

```text
Active
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-global-secondary-indexes.png"
    title="Hình 5.4.7.4: Global Secondary Index của bảng la_auction_catalog"
    width="100%"
>}}

### Bước 6: Kiểm tra TTL

Chọn bảng:

```text
la_websocket_connections
```

Mở:

```text
Additional settings
```

hoặc tìm phần:

```text
Time to Live (TTL)
```

Kiểm tra:

```text
TTL status: On
TTL attribute: ttl
```

Có thể kiểm tra thêm bảng `la_idempotency` với thuộc tính `expiration`.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-websocket-ttl.png"
    title="Hình 5.4.7.5: Cấu hình TTL của bảng WebSocket Connection"
    width="100%"
>}}

### Bước 7: Kiểm tra Point-in-Time Recovery

Chọn một bảng quan trọng, chẳng hạn:

```text
la_auction_catalog
```

Mở:

```text
Backups
```

Kiểm tra phần:

```text
Point-in-time recovery
```

Trạng thái cần xác nhận:

```text
Point-in-time recovery: On
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-point-in-time-recovery.png"
    title="Hình 5.4.7.6: Point-in-Time Recovery của bảng DynamoDB"
    width="100%"
>}}

### Bước 8: Kiểm tra DynamoDB Stream

Chọn bảng:

```text
la_item_auction_state
```

Mở:

```text
Exports and streams
```

Kiểm tra DynamoDB Stream.

Trạng thái cần xác nhận:

```text
DynamoDB stream: On
View type: New and old images
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-stream.png"
    title="Hình 5.4.7.7: DynamoDB Stream của bảng trạng thái đấu giá"
    width="100%"
>}}

### Bước 9: Kiểm tra Monitoring

Trong một bảng đang có dữ liệu, mở:

```text
Monitor
```

Kiểm tra các Metrics:

* Read requests.
* Write requests.
* Throttled requests.
* System errors.
* User errors.
* Successful request latency.

Các Metrics giúp theo dõi hoạt động của bảng trong quá trình sử dụng hệ thống.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-monitoring-metrics.png"
    title="Hình 5.4.7.8: Metrics giám sát bảng Amazon DynamoDB"
    width="100%"
>}}

## Kiểm tra dữ liệu WebSocket Connection

Sau khi người dùng kết nối đến WebSocket API, có thể mở:

```text
DynamoDB
→ Tables
→ la_websocket_connections
→ Explore table items
```

Kết quả mong đợi:

* Có Item tương ứng với kết nối đang hoạt động.
* Item chứa `item_id`.
* Item chứa `connection_id`.
* Item có thuộc tính `ttl`.
* Khi người dùng ngắt kết nối, dữ liệu được Lambda xóa hoặc tự động hết hạn.

Không đưa Connection ID thật vào nội dung báo cáo nếu không cần thiết.

Việc kiểm thử đầy đủ WebSocket Connection được trình bày tại mục **5.5 — Kiểm thử hệ thống**.

## Kết quả

Sau khi kiểm tra trực tiếp trên AWS Management Console, nhóm ghi nhận:

* Tám bảng DynamoDB đã được Terraform triển khai.
* Các bảng đang ở trạng thái Active.
* Các bảng sử dụng Billing Mode `PAY_PER_REQUEST`.
* Partition Key và Sort Key được cấu hình phù hợp với từng nghiệp vụ.
* Auction Catalog Table lưu dữ liệu phiên, vật phẩm và các thực thể liên quan.
* Bid Events Table lưu lịch sử đặt giá và hỗ trợ truy vấn theo người dùng.
* WebSocket Connections Table lưu các Connection đang hoạt động.
* TTL được bật cho dữ liệu kết nối, Idempotency và Audit Event.
* Global Secondary Index hỗ trợ các nhu cầu truy vấn bổ sung.
* Point-in-Time Recovery được bật cho các bảng dữ liệu quan trọng.
* Server-side Encryption được bật để bảo vệ dữ liệu lưu trữ.
* DynamoDB Stream được bật cho bảng trạng thái và sự kiện đặt giá.
* Lambda có thể đọc và ghi dữ liệu theo quyền IAM đã cấu hình.
* DynamoDB đã sẵn sàng hỗ trợ nghiệp vụ REST API và đấu giá thời gian thực.

Việc triển khai và kiểm tra hàng đợi xử lý yêu cầu đặt giá được trình bày tại mục **5.4.8 — Amazon SQS FIFO**.