---
title: "Amazon SQS FIFO"
date: 2026-07-27
weight: 8
chapter: false
pre: " <b> 5.4.8. </b> "
---

## Tổng quan

**Amazon Simple Queue Service (Amazon SQS)** là dịch vụ hàng đợi thông điệp được quản lý hoàn toàn trên AWS. Dịch vụ giúp tách rời các thành phần trong hệ thống và cho phép xử lý yêu cầu theo mô hình bất đồng bộ.

Trong hệ thống Live Auction, **Amazon SQS FIFO** được sử dụng để tiếp nhận các yêu cầu đặt giá trước khi chuyển chúng đến Lambda xử lý. Việc sử dụng FIFO giúp các yêu cầu đặt giá của cùng một phiên đấu giá được xử lý theo đúng thứ tự gửi đến.

Luồng xử lý yêu cầu đặt giá được triển khai như sau:

```text
Người dùng đặt giá
        ↓
API Gateway WebSocket
        ↓
Lambda la-ws-handler
        ↓
Amazon SQS FIFO
        ↓
Lambda la-bid-processor
        ↓
Amazon DynamoDB
        ↓
Lambda la-broadcast
        ↓
API Gateway WebSocket
        ↓
Cập nhật giá cho người dùng
```

Amazon SQS FIFO đóng vai trò trung gian giữa Lambda WebSocket Handler và Lambda Bid Processor, giúp hạn chế việc nhiều yêu cầu đặt giá được xử lý đồng thời không đúng thứ tự.

## Vai trò của Amazon SQS FIFO

Amazon SQS FIFO được sử dụng trong hệ thống nhằm:

- Tiếp nhận các yêu cầu đặt giá từ WebSocket Handler.
- Xử lý yêu cầu đặt giá theo đúng thứ tự.
- Nhóm các thông điệp theo từng phiên hoặc vật phẩm đấu giá.
- Hạn chế tình trạng hai yêu cầu đặt giá được xử lý sai thứ tự.
- Tách quá trình tiếp nhận yêu cầu khỏi quá trình xử lý nghiệp vụ.
- Cho phép Lambda Bid Processor xử lý yêu cầu theo mô hình bất đồng bộ.
- Tự động thử lại khi quá trình xử lý thông điệp gặp lỗi.
- Chuyển thông điệp không thể xử lý sang Dead-letter Queue nếu được cấu hình.

## Các thành phần liên quan

| Thành phần                    | Vai trò                                                            |
| ----------------------------- | ------------------------------------------------------------------ |
| **API Gateway WebSocket**     | Tiếp nhận kết nối và thông điệp từ người dùng.                     |
| **Lambda `la-ws-handler`**    | Kiểm tra thông điệp WebSocket và gửi yêu cầu đặt giá vào SQS FIFO. |
| **Amazon SQS FIFO**           | Lưu tạm thời và sắp xếp yêu cầu đặt giá theo đúng thứ tự.          |
| **Lambda `la-bid-processor`** | Nhận thông điệp từ SQS và xử lý nghiệp vụ đặt giá.                 |
| **Amazon DynamoDB**           | Lưu trạng thái đấu giá và lịch sử đặt giá.                         |
| **Lambda `la-broadcast`**     | Gửi kết quả đặt giá đến các kết nối WebSocket liên quan.           |
| **Dead-letter Queue**         | Lưu những thông điệp không thể xử lý sau số lần thử lại quy định.  |

## Bước 1: Mở Amazon SQS

Đăng nhập vào **AWS Management Console** và chọn đúng Region:

```text
Asia Pacific (Singapore) – ap-southeast-1
```

Tại thanh tìm kiếm dịch vụ, nhập:

```text
SQS
```

Sau đó chọn:

```text
Simple Queue Service
```

## Bước 2: Kiểm tra danh sách Queue

Trong giao diện Amazon SQS, mở:

```text
Queues
```

Tìm Queue được sử dụng để xử lý yêu cầu đặt giá của hệ thống Live Auction.

Tên của FIFO Queue phải kết thúc bằng:

```text
.fifo
```

Kiểm tra các thông tin:

- Tên Queue.
- Loại Queue là FIFO.
- Region là `ap-southeast-1`.
- Thời điểm Queue được tạo.
- Trạng thái hoạt động của Queue.
- Dead-letter Queue nếu đã được cấu hình.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-queue-list.png"
    title="Hình 5.4.8.1: Danh sách Amazon SQS Queue của hệ thống Live Auction"
    width="100%"
>}}

## Bước 3: Kiểm tra thông tin SQS FIFO Queue

Chọn FIFO Queue xử lý yêu cầu đặt giá và mở tab:

```text
Details
```

Kiểm tra:

- Type là `FIFO`.
- Queue ARN.
- Queue URL.
- Content-based deduplication.
- Visibility timeout.
- Message retention period.
- Delivery delay.
- Maximum message size.
- Receive message wait time.

Đặc điểm quan trọng nhất cần xác nhận là Queue thuộc loại:

```text
FIFO
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-fifo-details.png"
    title="Hình 5.4.8.2: Thông tin cấu hình của Amazon SQS FIFO Queue"
    width="100%"
>}}

## Bước 4: Kiểm tra cấu hình FIFO

Trong phần cấu hình Queue, kiểm tra các thuộc tính dành cho FIFO Queue.

Các nội dung cần xác nhận:

- Queue name kết thúc bằng `.fifo`.
- Queue type là FIFO.
- Cơ chế loại bỏ thông điệp trùng lặp.
- Thứ tự thông điệp được duy trì trong cùng một Message Group.
- Thông điệp có Message Deduplication ID hoặc sử dụng Content-based Deduplication.

Trong hệ thống Live Auction, các yêu cầu thuộc cùng một phiên hoặc vật phẩm đấu giá có thể sử dụng chung một **Message Group ID**. Nhờ đó, SQS duy trì đúng thứ tự các lượt đặt giá trong cùng nhóm.

Ví dụ:

```text
MessageGroupId = auction-session-or-item-id
```

Mỗi yêu cầu cũng cần có thông tin nhận diện để hạn chế xử lý trùng lặp:

```text
MessageDeduplicationId = unique-bid-request-id
```

{{% notice info %}}
SQS FIFO chỉ bảo đảm thứ tự trong cùng một Message Group. Các thông điệp thuộc những Message Group khác nhau vẫn có thể được xử lý song song.
{{% /notice %}}

## Bước 5: Kiểm tra Lambda Trigger

Mở Lambda Function:

```text
la-bid-processor
```

Tại tab **Configuration**, chọn:

```text
Triggers
```

Kiểm tra Trigger đến từ Amazon SQS.

Các nội dung cần xác nhận:

- Trigger Type là `SQS`.
- Queue được liên kết là FIFO Queue xử lý yêu cầu đặt giá.
- Trigger được bật.
- Batch size.
- Event source mapping ở trạng thái hoạt động.
- Lambda Function nhận thông điệp là `la-bid-processor`.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-lambda-trigger.png"
    title="Hình 5.4.8.3: SQS Trigger của Lambda Bid Processor"
    width="100%"
>}}

## Bước 6: Kiểm tra Event Source Mapping

Trong cấu hình Trigger của Lambda `la-bid-processor`, mở phần thông tin Event Source Mapping.

Kiểm tra:

- Event source là SQS FIFO Queue.
- Trạng thái là `Enabled`.
- Batch size phù hợp với quá trình xử lý đặt giá.
- Maximum batching window nếu có.
- Cấu hình báo cáo lỗi theo từng thông điệp nếu được sử dụng.
- Lambda có quyền nhận và xóa thông điệp khỏi Queue.

Event Source Mapping cho phép AWS Lambda tự động đọc thông điệp từ SQS. Sau khi Lambda xử lý thành công, thông điệp được xóa khỏi Queue. Nếu xử lý thất bại, thông điệp sẽ hiển thị lại sau khi hết Visibility Timeout và được thử lại.

## Bước 7: Kiểm tra Dead-letter Queue

Nếu hệ thống có cấu hình Dead-letter Queue, mở FIFO Queue chính và kiểm tra phần:

```text
Dead-letter queue
```

hoặc:

```text
Redrive policy
```

Kiểm tra:

- Tên Dead-letter Queue.
- ARN của Dead-letter Queue.
- Maximum receives.
- Dead-letter Queue tồn tại trong cùng Region.
- Tên Queue có đúng quy ước của dự án.

Dead-letter Queue tiếp nhận các thông điệp không thể xử lý sau số lần thử lại được cấu hình. Cơ chế này giúp nhóm có thể kiểm tra lỗi mà không làm mất thông điệp đặt giá.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-dead-letter-queue.png"
    title="Hình 5.4.8.4: Cấu hình Dead-letter Queue của SQS FIFO"
    width="100%"
>}}

{{% notice warning %}}
Chỉ giữ hình Dead-letter Queue nếu hệ thống thực tế đã cấu hình. Nếu giao diện AWS hiển thị chưa có Dead-letter Queue, không được ghi trong báo cáo rằng tính năng này đã được triển khai.
{{% /notice %}}

## Bước 8: Kiểm tra quyền IAM

Mở IAM Role của Lambda:

```text
la-bid-processor
```

Kiểm tra các quyền cần thiết để Lambda làm việc với Amazon SQS, chẳng hạn:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

Đối với Lambda gửi yêu cầu đặt giá vào Queue, IAM Role tương ứng cần có quyền:

```text
sqs:SendMessage
```

Các quyền phải được giới hạn vào đúng Queue ARN theo nguyên tắc đặc quyền tối thiểu, thay vì cấp quyền cho toàn bộ tài nguyên SQS.

Không cần chụp thêm nếu quyền IAM liên quan đã được trình bày trong mục **5.4.1 – AWS IAM và Amazon Cognito**.

## Bước 9: Kiểm tra Monitoring

Trong Amazon SQS, chọn FIFO Queue và mở tab:

```text
Monitoring
```

Chọn khoảng thời gian có dữ liệu và kiểm tra các chỉ số:

- Number of messages sent.
- Number of messages received.
- Number of messages deleted.
- Approximate number of messages available.
- Approximate number of messages not visible.
- Age of oldest message.

Các chỉ số này giúp xác nhận rằng:

- Yêu cầu đặt giá đã được gửi vào Queue.
- Lambda đã nhận thông điệp.
- Thông điệp được xóa sau khi xử lý thành công.
- Không có lượng lớn thông điệp tồn đọng.
- Không có thông điệp bị chờ quá lâu trong Queue.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-cloudwatch-metrics.png"
    title="Hình 5.4.8.5: Các chỉ số giám sát của Amazon SQS FIFO"
    width="100%"
>}}

## Bước 10: Kiểm tra hoạt động thực tế

Để kiểm tra luồng xử lý thực tế, thực hiện:

1. Đăng nhập vào User Frontend.
2. Mở một phiên đấu giá đang hoạt động.
3. Tham gia phiên đấu giá.
4. Gửi một mức giá hợp lệ.
5. Mở Amazon SQS và kiểm tra Monitoring.
6. Mở CloudWatch Logs của Lambda `la-bid-processor`.
7. Kiểm tra dữ liệu đấu giá trong Amazon DynamoDB.
8. Xác nhận mức giá mới được cập nhật trên giao diện người dùng.

Nếu Lambda xử lý nhanh, số lượng Message Available có thể trở về `0` trước khi kiểm tra. Trong trường hợp đó, sử dụng các biểu đồ **Messages sent**, **Messages received** và **Messages deleted** để chứng minh Queue đã hoạt động.


## Kết quả

Sau khi kiểm tra, Amazon SQS FIFO đã được tích hợp vào luồng xử lý đặt giá của hệ thống Live Auction.

Kết quả đạt được:

- Yêu cầu đặt giá được tiếp nhận thông qua WebSocket API.
- Lambda `la-ws-handler` gửi yêu cầu hợp lệ vào SQS FIFO.
- Các thông điệp trong cùng một nhóm được duy trì đúng thứ tự.
- Lambda `la-bid-processor` được kích hoạt để xử lý thông điệp.
- Kết quả đặt giá được lưu vào Amazon DynamoDB.
- Trạng thái mới được gửi đến người dùng thông qua WebSocket.
- IAM kiểm soát quyền gửi, nhận và xóa thông điệp.
- CloudWatch Metrics hỗ trợ theo dõi hoạt động và tình trạng tồn đọng của Queue.
- Thông điệp xử lý thất bại có thể được chuyển đến Dead-letter Queue nếu Redrive Policy đã được cấu hình.

Việc sử dụng Amazon SQS FIFO giúp quá trình xử lý đặt giá có thứ tự, giảm sự phụ thuộc trực tiếp giữa các Lambda Function và tăng khả năng ổn định của hệ thống khi nhiều người dùng đặt giá cùng lúc.