---
title: "Kiểm thử luồng đặt giá đầu cuối"
date: 2026-08-03
weight: 6
chapter: false
pre: " <b> 5.5.6. </b> "
---

#### Mục tiêu

Phần kiểm thử này xác minh toàn bộ luồng đặt giá thời gian thực của hệ thống Live Auction, từ lúc người dùng gửi yêu cầu trên frontend đến khi giá mới được xử lý, lưu trữ và broadcast đến các người dùng đang theo dõi.

Luồng cần kiểm tra:

```
Frontend
-> WebSocket API
-> la-ws-handler
-> SQS FIFO
-> la-bid-processor
-> DynamoDB
-> la-broadcast
-> WebSocket API
-> Frontend
```

Kiểm thử không chỉ xác nhận giao diện hiển thị giá mới mà còn phải chứng minh:

- Yêu cầu đặt giá được xác thực.
- Message được đưa vào đúng SQS FIFO queue.
- Message trùng không tạo nhiều lần đặt giá.
- Các yêu cầu được xử lý đúng thứ tự trong cùng phiên đấu giá.
- Việc cập nhật giá có tính nguyên tử.
- Người đặt giá cao nhất được xác định chính xác.
- Lịch sử đặt giá được lưu đầy đủ.
- Kết quả được broadcast đúng phòng đấu giá.
- Frontend cập nhật mà không cần tải lại trang.
- Không làm lộ token, thông tin xác thực hoặc dữ liệu hạ tầng nội bộ.

---



#### Thành phần liên quan

Các thành phần cần được kiểm tra bao gồm:

- Trang chi tiết vật phẩm đấu giá trên frontend.
- API Gateway WebSocket API.
- Route WebSocket dùng để đặt giá, ví dụ `place_bid`.
- Lambda `la-ws-handler`.
- Amazon SQS FIFO.
- Dead-letter queue nếu được cấu hình.
- Lambda `la-bid-processor`.
- DynamoDB Auction hoặc Current Bid Table.
- DynamoDB Bid History Table.
- Lambda `la-broadcast`.
- API Gateway Management API.
- CloudWatch Logs.
- Cơ chế xác thực JWT hoặc Amazon Cognito.
- Cơ chế kiểm soát quyền tham gia phiên đấu giá.

---



#### Điều kiện kiểm thử chung

Trước khi kiểm thử, hệ thống cần đáp ứng các điều kiện sau:

- WebSocket API đã được triển khai.
- Frontend đã được cấu hình đúng WebSocket URL của môi trường kiểm thử.
- Route đặt giá đã được liên kết với `la-ws-handler`.
- `la-ws-handler` có quyền gửi message vào SQS FIFO.
- SQS FIFO đã được cấu hình đúng.
- `la-bid-processor` được cấu hình nhận message từ SQS.
- `la-bid-processor` có quyền đọc và cập nhật dữ liệu DynamoDB.
- `la-broadcast` có quyền gọi `execute-api:ManageConnections`.
- Bảng lưu giá hiện tại và lịch sử đặt giá đã tồn tại.
- DynamoDB Connections hoặc Subscriptions Table đã có dữ liệu phòng.
- Có ít nhất hai tài khoản hợp lệ.
- Có một tài khoản không thuộc phiên hoặc không được phép đặt giá.
- Có ít nhất một phiên chưa bắt đầu, một phiên đang hoạt động và một phiên đã kết thúc.
- CloudWatch Logs đã được bật.
- Môi trường kiểm thử được tách khỏi production.
- Có quyền kiểm tra SQS, DynamoDB và CloudWatch Logs.
- Thời gian trên thiết bị kiểm thử được đồng bộ.

Nếu một thành phần trong luồng chưa được triển khai, test case liên quan phải được đánh dấu `BLOCKED`.

---



#### Dữ liệu kiểm thử


| Dữ liệu               | Mô tả                                                             |
| --------------------- | ----------------------------------------------------------------- |
| User A                | User hợp lệ và được phép đặt giá                                  |
| User B                | User hợp lệ thứ hai trong cùng phiên                              |
| User C                | User hợp lệ nhưng không thuộc hoặc không được phép tham gia phiên |
| Anonymous User        | Người dùng chưa đăng nhập                                         |
| Auction Active        | Phiên đang ở trạng thái `ACTIVE`                                  |
| Auction Scheduled     | Phiên chưa bắt đầu, trạng thái `SCHEDULED`                        |
| Auction Ended         | Phiên đã kết thúc, trạng thái `ENDED`                             |
| Current Price         | Giá hiện tại của vật phẩm, ví dụ `1.000.000 VND`                  |
| Minimum Increment     | Bước giá tối thiểu, ví dụ `100.000 VND`                           |
| Valid Bid             | Giá hợp lệ, ví dụ `1.100.000 VND`                                 |
| Low Bid               | Giá bằng hoặc thấp hơn giá hiện tại                               |
| Invalid Increment Bid | Giá cao hơn hiện tại nhưng không đủ bước giá                      |
| Request ID            | Mã duy nhất của một yêu cầu đặt giá                               |
| Client Message ID     | Mã do client tạo để hỗ trợ idempotency                            |
| Room ID               | Mã phòng WebSocket của vật phẩm                                   |
| Auction ID            | Mã phiên đấu giá                                                  |
| Item ID               | Mã vật phẩm đấu giá                                               |
| Expired Token         | Token đã hết hạn                                                  |
| Invalid Token         | Token sai chữ ký hoặc sai định dạng                               |


Không sử dụng dữ liệu production trong quá trình kiểm thử.

---



#### Cấu trúc yêu cầu đặt giá

Ví dụ message do frontend gửi:

```
{
  "action": "place_bid",
  "requestId": "bid-request-001",
  "auctionId": "auction-active-001",
  "itemId": "item-001",
  "amount": 1100000
}
```

Frontend không được phép gửi hoặc tự quyết định:

```
{
  "userId": "trusted-user-id",
  "role": "ADMIN",
  "currentPrice": 1000000,
  "isWinner": true
}
```

Danh tính người đặt giá phải được lấy từ token hoặc authentication context đã xác minh.

Giá hiện tại, trạng thái phiên, bước giá tối thiểu và người đặt giá cao nhất phải được đọc từ nguồn dữ liệu tin cậy phía server.

---



#### Cấu trúc kết quả thành công

Ví dụ sự kiện broadcast khi đặt giá thành công:

```
{
  "type": "BID_ACCEPTED",
  "requestId": "bid-request-001",
  "auctionId": "auction-active-001",
  "itemId": "item-001",
  "data": {
    "amount": 1100000,
    "highestBidderId": "masked-user-id",
    "bidSequence": 15
  },
  "timestamp": "2026-08-09T12:00:00Z"
}
```



#### Cấu trúc kết quả thất bại

```
{
  "type": "BID_REJECTED",
  "requestId": "bid-request-001",
  "error": {
    "code": "MINIMUM_INCREMENT_NOT_MET",
    "message": "The bid does not meet the minimum increment"
  },
  "timestamp": "2026-08-09T12:00:00Z"
}
```

Response không được chứa:

- Access Token.
- ID Token.
- Refresh Token.
- AWS credentials.
- Nội dung message SQS nội bộ không cần thiết.
- Tên bảng DynamoDB.
- Stack trace.
- Connection ID của người dùng khác.
- Email hoặc dữ liệu cá nhân không cần thiết.

---



### BID-01 — Đặt giá hợp lệ


| Trường                   | Nội dung                                                                                                                                                                                                                                                                                                  |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-01`                                                                                                                                                                                                                                                                                                  |
| **Tên kiểm thử**         | User đặt giá hợp lệ trong phiên đang hoạt động                                                                                                                                                                                                                                                            |
| **Mục tiêu**             | Xác minh một yêu cầu đặt giá hợp lệ đi qua toàn bộ luồng và được lưu thành công.                                                                                                                                                                                                                          |
| **Điều kiện tiên quyết** | Auction Active đang hoạt động; User A đã đăng nhập và tham gia đúng phòng; giá hiện tại là `1.000.000 VND`; bước giá là `100.000 VND`.                                                                                                                                                                    |
| **Các bước thực hiện**   | 1. User A mở trang đấu giá. 2. Nhập giá `1.100.000 VND`. 3. Gửi yêu cầu đặt giá. 4. Kiểm tra WebSocket frame. 5. Kiểm tra log `la-ws-handler`. 6. Kiểm tra message được gửi vào SQS FIFO. 7. Kiểm tra `la-bid-processor` xử lý message. 8. Kiểm tra DynamoDB. 9. Kiểm tra message broadcast và giao diện. |
| **Kết quả mong đợi**     | Yêu cầu được chấp nhận; message được đưa vào SQS đúng một lần; giá hiện tại được cập nhật thành `1.100.000 VND`; User A trở thành người đặt giá cao nhất; một bản ghi lịch sử được tạo; các user trong phòng nhận `BID_ACCEPTED`; giao diện cập nhật mà không tải lại trang.                              |
| **Kết quả thực tế**      | Điền Request ID, Message ID, giá trước và sau, người đặt giá cao nhất và thời gian xử lý.                                                                                                                                                                                                                 |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                            |
| **Bằng chứng**           | WebSocket frames, CloudWatch Logs, SQS metrics, DynamoDB trước và sau, giao diện frontend.                                                                                                                                                                                                                |


---



### BID-02 — Giá đặt bằng hoặc thấp hơn giá hiện tại


| Trường                   | Nội dung                                                                                                                                                                                                                    |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-02`                                                                                                                                                                                                                    |
| **Tên kiểm thử**         | Từ chối giá không cao hơn giá hiện tại                                                                                                                                                                                      |
| **Mục tiêu**             | Xác minh hệ thống không chấp nhận giá bằng hoặc thấp hơn current price.                                                                                                                                                     |
| **Điều kiện tiên quyết** | Giá hiện tại là `1.100.000 VND`; phiên đang hoạt động.                                                                                                                                                                      |
| **Các bước thực hiện**   | 1. Gửi giá `1.100.000 VND`. 2. Gửi tiếp giá `1.000.000 VND` bằng Request ID khác. 3. Kiểm tra kết quả từng yêu cầu. 4. Kiểm tra DynamoDB và lịch sử đặt giá. 5. Kiểm tra broadcast.                                         |
| **Kết quả mong đợi**     | Cả hai yêu cầu đều bị từ chối với error code như `BID_NOT_HIGHER_THAN_CURRENT_PRICE`; giá hiện tại không thay đổi; người đặt giá cao nhất không thay đổi; không tạo bid history thành công; không broadcast `BID_ACCEPTED`. |
| **Kết quả thực tế**      | Điền từng giá gửi, error code và giá sau kiểm thử.                                                                                                                                                                          |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                              |
| **Bằng chứng**           | Frame lỗi, CloudWatch Logs và dữ liệu DynamoDB không thay đổi.                                                                                                                                                              |


---



### BID-03 — Không đáp ứng bước giá tối thiểu


| Trường                   | Nội dung                                                                                                                                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `BID-03`                                                                                                                                                                                                                             |
| **Tên kiểm thử**         | Từ chối giá không đủ minimum increment                                                                                                                                                                                               |
| **Mục tiêu**             | Xác minh giá mới phải đáp ứng bước giá tối thiểu.                                                                                                                                                                                    |
| **Điều kiện tiên quyết** | Giá hiện tại là `1.100.000 VND`; bước giá là `100.000 VND`.                                                                                                                                                                          |
| **Các bước thực hiện**   | 1. User A gửi giá `1.150.000 VND`. 2. Kiểm tra quá trình xử lý. 3. Kiểm tra response. 4. Kiểm tra DynamoDB. 5. Kiểm tra các user khác có nhận cập nhật hay không.                                                                    |
| **Kết quả mong đợi**     | Yêu cầu bị từ chối với `MINIMUM_INCREMENT_NOT_MET`; server có thể trả minimum acceptable bid là `1.200.000 VND`; current price và highest bidder không thay đổi; không tạo lịch sử bid thành công; không broadcast cập nhật giá mới. |
| **Kết quả thực tế**      | Điền giá gửi, minimum acceptable bid và error code.                                                                                                                                                                                  |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                       |
| **Bằng chứng**           | WebSocket response, log processor và DynamoDB.                                                                                                                                                                                       |


---



### BID-04 — Đặt giá khi phiên chưa bắt đầu


| Trường                   | Nội dung                                                                                                                                                                                    |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-04`                                                                                                                                                                                    |
| **Tên kiểm thử**         | Từ chối đặt giá trước thời gian bắt đầu                                                                                                                                                     |
| **Mục tiêu**             | Xác minh phiên `SCHEDULED` không nhận bid.                                                                                                                                                  |
| **Điều kiện tiên quyết** | Auction Scheduled tồn tại và thời gian hiện tại nhỏ hơn `startTime`.                                                                                                                        |
| **Các bước thực hiện**   | 1. Mở phiên chưa bắt đầu. 2. Gửi một mức giá hợp lệ về mặt số tiền. 3. Kiểm tra response. 4. Kiểm tra trạng thái và thời gian phiên trong DynamoDB. 5. Kiểm tra lịch sử bid.                |
| **Kết quả mong đợi**     | Yêu cầu bị từ chối với `AUCTION_NOT_STARTED`; không cập nhật current price; không cập nhật highest bidder; không tạo bid history thành công; frontend vẫn hiển thị trạng thái chưa bắt đầu. |
| **Kết quả thực tế**      | Điền trạng thái phiên, start time, server time và error code.                                                                                                                               |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                              |
| **Bằng chứng**           | WebSocket response, DynamoDB và CloudWatch Logs.                                                                                                                                            |


---



### BID-05 — Đặt giá khi phiên đã kết thúc


| Trường                   | Nội dung                                                                                                                                                          |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-05`                                                                                                                                                          |
| **Tên kiểm thử**         | Từ chối đặt giá sau khi phiên kết thúc                                                                                                                            |
| **Mục tiêu**             | Xác minh phiên đã kết thúc không nhận thêm bid.                                                                                                                   |
| **Điều kiện tiên quyết** | Auction Ended có trạng thái `ENDED` hoặc thời gian hiện tại đã vượt `endTime`.                                                                                    |
| **Các bước thực hiện**   | 1. Mở phiên đã kết thúc. 2. Gửi một mức giá cao hơn current price. 3. Kiểm tra response. 4. Kiểm tra DynamoDB. 5. Kiểm tra winner và bid history.                 |
| **Kết quả mong đợi**     | Yêu cầu bị từ chối với `AUCTION_ENDED`; current price, winner và highest bidder không thay đổi; không tạo bid history thành công; không broadcast `BID_ACCEPTED`. |
| **Kết quả thực tế**      | Điền end time, server time, trạng thái phiên và error code.                                                                                                       |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                    |
| **Bằng chứng**           | Frame lỗi, DynamoDB và CloudWatch Logs.                                                                                                                           |


---



### BID-06 — User không hợp lệ hoặc chưa đăng nhập


| Trường                   | Nội dung                                                                                                                                                                                                    |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-06`                                                                                                                                                                                                    |
| **Tên kiểm thử**         | Từ chối yêu cầu đặt giá không được xác thực                                                                                                                                                                 |
| **Mục tiêu**             | Xác minh chỉ người dùng đã được xác thực mới có thể đặt giá.                                                                                                                                                |
| **Điều kiện tiên quyết** | Hệ thống có cơ chế xác thực WebSocket.                                                                                                                                                                      |
| **Các bước thực hiện**   | 1. Thử đặt giá khi chưa đăng nhập. 2. Thử bằng token hết hạn. 3. Thử bằng token sai chữ ký. 4. Thử gửi `userId` giả trong message. 5. Kiểm tra SQS, DynamoDB và logs.                                       |
| **Kết quả mong đợi**     | Kết nối hoặc yêu cầu bị từ chối với `UNAUTHENTICATED` hoặc mã tương đương; `userId` từ client không được tin tưởng; không tạo bid hợp lệ; không cập nhật giá; không tạo message nghiệp vụ hợp lệ trong SQS. |
| **Kết quả thực tế**      | Điền loại token, kết quả kết nối hoặc error code tương ứng.                                                                                                                                                 |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                              |
| **Bằng chứng**           | Handshake hoặc frame lỗi, SQS không có bid hợp lệ, DynamoDB không thay đổi và logs đã che token.                                                                                                            |


---



### BID-07 — User không thuộc phiên gửi yêu cầu


| Trường                   | Nội dung                                                                                                                                                                                |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-07`                                                                                                                                                                                |
| **Tên kiểm thử**         | Từ chối User không có quyền tham gia phiên                                                                                                                                              |
| **Mục tiêu**             | Xác minh User C không thể đặt giá trong phiên mà họ không được phép tham gia.                                                                                                           |
| **Điều kiện tiên quyết** | User C đã xác thực nhưng không có membership, registration hoặc quyền tham gia Auction Active.                                                                                          |
| **Các bước thực hiện**   | 1. Đăng nhập bằng User C. 2. Gửi yêu cầu đặt giá đến Auction Active. 3. Kiểm tra authorization result. 4. Kiểm tra SQS và processor log. 5. Kiểm tra DynamoDB và broadcast.             |
| **Kết quả mong đợi**     | Yêu cầu bị từ chối với `NOT_AUCTION_PARTICIPANT` hoặc `FORBIDDEN`; không cập nhật giá; không tạo lịch sử bid thành công; không thay đổi highest bidder; không broadcast bid thành công. |
| **Kết quả thực tế**      | Điền User ID đã che, Auction ID và error code.                                                                                                                                          |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                          |
| **Bằng chứng**           | Frame lỗi, authorization log và DynamoDB không thay đổi.                                                                                                                                |


---



### BID-08 — Gửi lại cùng một yêu cầu đặt giá


| Trường                   | Nội dung                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-08`                                                                                                                                                                                                                     |
| **Tên kiểm thử**         | Xử lý idempotency và message trùng                                                                                                                                                                                           |
| **Mục tiêu**             | Xác minh cùng một yêu cầu không tạo nhiều bid khi được gửi lại hoặc SQS giao lại message.                                                                                                                                    |
| **Điều kiện tiên quyết** | Có `requestId` hoặc `clientMessageId`; cơ chế idempotency đã được triển khai.                                                                                                                                                |
| **Các bước thực hiện**   | 1. Gửi bid hợp lệ với `requestId=bid-request-008`. 2. Chờ yêu cầu được xử lý thành công. 3. Gửi lại chính xác request đó. 4. Nếu có thể, mô phỏng SQS redelivery. 5. Kiểm tra bid history, current price và broadcast.       |
| **Kết quả mong đợi**     | Chỉ một bid được áp dụng; chỉ một bản ghi lịch sử nghiệp vụ được tạo; current price chỉ cập nhật một lần; request trùng nhận lại kết quả trước hoặc `DUPLICATE_REQUEST`; không tạo broadcast nghiệp vụ trùng ngoài thiết kế. |
| **Kết quả thực tế**      | Điền số lần gửi, số lần processor chạy, số bid history và số event broadcast.                                                                                                                                                |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                               |
| **Bằng chứng**           | Hai frame gửi, log idempotency, SQS message attributes, DynamoDB và broadcast logs.                                                                                                                                          |


> Không được chỉ phụ thuộc vào SQS FIFO deduplication. `la-bid-processor` vẫn cần cơ chế idempotency vì message có thể được giao lại sau khi visibility timeout hết hạn hoặc Lambda xử lý thành công nhưng chưa xác nhận hoàn tất batch.

---



### BID-09 — Nhiều người đặt giá lần lượt


| Trường                   | Nội dung                                                                                                                                                                                                         |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-09`                                                                                                                                                                                                         |
| **Tên kiểm thử**         | Xử lý chuỗi đặt giá của nhiều User                                                                                                                                                                               |
| **Mục tiêu**             | Xác minh các bid hợp lệ liên tiếp được xử lý đúng thứ tự.                                                                                                                                                        |
| **Điều kiện tiên quyết** | User A và User B đang theo dõi cùng phiên; current price là `1.000.000 VND`; minimum increment là `100.000 VND`.                                                                                                 |
| **Các bước thực hiện**   | 1. User A đặt `1.100.000 VND`. 2. Chờ kết quả thành công. 3. User B đặt `1.200.000 VND`. 4. User A đặt `1.300.000 VND`. 5. Kiểm tra thứ tự message trong SQS FIFO. 6. Kiểm tra DynamoDB và các event broadcast.  |
| **Kết quả mong đợi**     | Các bid được xử lý theo đúng thứ tự trong cùng Auction/Item message group; giá lần lượt là `1.100.000`, `1.200.000`, `1.300.000 VND`; highest bidder thay đổi A → B → A; lịch sử có đúng ba bản ghi theo thứ tự. |
| **Kết quả thực tế**      | Điền sequence, User, amount, timestamp và kết quả từng bid.                                                                                                                                                      |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                   |
| **Bằng chứng**           | WebSocket frames của hai User, logs, SQS attributes và DynamoDB bid history.                                                                                                                                     |


---



### BID-10 — Người đặt giá cao nhất được cập nhật đúng


| Trường                   | Nội dung                                                                                                                                                                                                                  |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-10`                                                                                                                                                                                                                  |
| **Tên kiểm thử**         | Cập nhật highest bidder chính xác                                                                                                                                                                                         |
| **Mục tiêu**             | Xác minh highest bidder luôn tương ứng với bid hợp lệ cao nhất đã được chấp nhận.                                                                                                                                         |
| **Điều kiện tiên quyết** | Có nhiều bid đã được gửi bởi User A và User B.                                                                                                                                                                            |
| **Các bước thực hiện**   | 1. Ghi nhận highest bidder ban đầu. 2. User A gửi bid hợp lệ. 3. Kiểm tra highest bidder. 4. User B gửi bid cao hơn. 5. Kiểm tra lại highest bidder. 6. Gửi một bid không hợp lệ từ User A. 7. Kiểm tra dữ liệu lần cuối. |
| **Kết quả mong đợi**     | Highest bidder chỉ thay đổi khi bid mới được chấp nhận; bid bị từ chối không thay đổi highest bidder; `highestBidAmount`, `highestBidderId` và bid history tham chiếu cùng một kết quả.                                   |
| **Kết quả thực tế**      | Điền highest bidder và amount sau từng bước.                                                                                                                                                                              |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                            |
| **Bằng chứng**           | DynamoDB trước và sau, processor logs và message broadcast.                                                                                                                                                               |


---



### BID-11 — Lịch sử đặt giá được lưu đúng


| Trường                   | Nội dung                                                                                                                                                                                                                                                 |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-11`                                                                                                                                                                                                                                                 |
| **Tên kiểm thử**         | Lưu đầy đủ và chính xác bid history                                                                                                                                                                                                                      |
| **Mục tiêu**             | Xác minh mỗi bid thành công tạo một bản ghi lịch sử có thể truy vết.                                                                                                                                                                                     |
| **Điều kiện tiên quyết** | Có ít nhất ba bid thành công trong cùng phiên.                                                                                                                                                                                                           |
| **Các bước thực hiện**   | 1. Thực hiện nhiều bid hợp lệ. 2. Truy vấn lịch sử theo Auction ID hoặc Item ID. 3. Kiểm tra User ID, amount, timestamp, request ID và sequence. 4. Kiểm tra thứ tự hiển thị. 5. Đối chiếu với processor logs và current price.                          |
| **Kết quả mong đợi**     | Mỗi bid được chấp nhận có đúng một bản ghi; amount và User ID chính xác; thứ tự lịch sử đúng; request ID không trùng ngoài yêu cầu idempotent; bid mới nhất khớp với current price và highest bidder; bid bị từ chối không xuất hiện như bid thành công. |
| **Kết quả thực tế**      | Điền số bid gửi, số bid được chấp nhận và số bản ghi lịch sử.                                                                                                                                                                                            |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                           |
| **Bằng chứng**           | Kết quả truy vấn DynamoDB, CloudWatch Logs và dữ liệu hiển thị trên frontend.                                                                                                                                                                            |


---



### BID-12 — Broadcast đến tất cả User đang theo dõi


| Trường                   | Nội dung                                                                                                                                                                                                                 |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `BID-12`                                                                                                                                                                                                                 |
| **Tên kiểm thử**         | Broadcast kết quả đặt giá đến đúng phòng                                                                                                                                                                                 |
| **Mục tiêu**             | Xác minh mọi kết nối đang hoạt động trong đúng phòng nhận kết quả cập nhật giá.                                                                                                                                          |
| **Điều kiện tiên quyết** | User A và User B đang theo dõi Room A; User C đang theo dõi Room B.                                                                                                                                                      |
| **Các bước thực hiện**   | 1. Mở Room A bằng User A và User B. 2. Mở Room B bằng User C. 3. User A đặt giá hợp lệ trong Room A. 4. Quan sát message tại ba cửa sổ. 5. Kiểm tra `la-broadcast` logs. 6. Kiểm tra danh sách connection được truy vấn. |
| **Kết quả mong đợi**     | User A và User B nhận `BID_ACCEPTED` của Room A; User C không nhận sự kiện đó; message chứa đúng Auction ID, Item ID, amount và sequence; một connection lỗi không làm thất bại toàn bộ broadcast.                       |
| **Kết quả thực tế**      | Điền message nhận được hoặc không nhận được tại từng User.                                                                                                                                                               |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                           |
| **Bằng chứng**           | Ba cửa sổ trình duyệt, WebSocket frames, DynamoDB subscription và broadcast logs.                                                                                                                                        |


---



### BID-13 — Frontend cập nhật giá không cần tải lại trang


| Trường                   | Nội dung                                                                                                                                                                                                                                  |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `BID-13`                                                                                                                                                                                                                                  |
| **Tên kiểm thử**         | Cập nhật giao diện theo thời gian thực                                                                                                                                                                                                    |
| **Mục tiêu**             | Xác minh frontend xử lý sự kiện broadcast và cập nhật dữ liệu ngay trên giao diện.                                                                                                                                                        |
| **Điều kiện tiên quyết** | User A và User B đang mở cùng trang; WebSocket đang ở trạng thái Connected hoặc Live.                                                                                                                                                     |
| **Các bước thực hiện**   | 1. Ghi nhận giá hiển thị ở hai cửa sổ. 2. User A đặt giá hợp lệ. 3. Không tải lại trang. 4. Quan sát giá mới ở cả hai cửa sổ. 5. Kiểm tra highest bidder, bid history và thông báo giao diện. 6. Kiểm tra browser console và Network tab. |
| **Kết quả mong đợi**     | Cả hai cửa sổ hiển thị giá mới mà không reload; highest bidder và bid history được cập nhật theo thiết kế; không tạo duplicate UI entry; không có lỗi JavaScript; giá hiển thị khớp với DynamoDB và message broadcast.                    |
| **Kết quả thực tế**      | Điền giá trước, giá sau, thời gian cập nhật và trạng thái UI.                                                                                                                                                                             |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                            |
| **Bằng chứng**           | Video hoặc ảnh trước và sau, WebSocket frame, browser console và DynamoDB.                                                                                                                                                                |


---



### Ma trận kiểm tra dữ liệu sau mỗi loại yêu cầu


| Trường hợp                 | Current price        | Highest bidder       | Bid history               | Broadcast thành công                 |
| -------------------------- | -------------------- | -------------------- | ------------------------- | ------------------------------------ |
| Bid hợp lệ                 | Phải cập nhật        | Phải cập nhật        | Thêm đúng một bản ghi     | Có                                   |
| Bid bằng/thấp hơn hiện tại | Không thay đổi       | Không thay đổi       | Không thêm bid thành công | Không                                |
| Không đủ bước giá          | Không thay đổi       | Không thay đổi       | Không thêm bid thành công | Không                                |
| Phiên chưa bắt đầu         | Không thay đổi       | Không thay đổi       | Không thêm bid thành công | Không                                |
| Phiên đã kết thúc          | Không thay đổi       | Không thay đổi       | Không thêm bid thành công | Không                                |
| User chưa xác thực         | Không thay đổi       | Không thay đổi       | Không thêm                | Không                                |
| User không có quyền        | Không thay đổi       | Không thay đổi       | Không thêm bid thành công | Không                                |
| Request trùng              | Chỉ cập nhật một lần | Chỉ cập nhật một lần | Chỉ có một bản ghi        | Không broadcast trùng ngoài thiết kế |


---



### Quy định kiểm tra SQS FIFO

SQS FIFO cần được kiểm tra theo các tiêu chí:

- Message được gửi vào đúng queue.
- `MessageGroupId` được xác định theo Auction ID hoặc Item ID.
- Các bid của cùng một vật phẩm được xử lý theo thứ tự.
- Không sử dụng một `MessageGroupId` chung cho toàn bộ hệ thống nếu điều đó làm các phiên khác chặn lẫn nhau.
- `MessageDeduplicationId` hoặc content-based deduplication được cấu hình đúng.
- Message chứa Request ID để hỗ trợ truy vết và idempotency.
- Message không chứa access token.
- Message không tin tưởng User ID do client gửi.
- User ID trong message nội bộ phải đến từ authentication context đã xác minh.
- Message lỗi được retry theo cấu hình.
- Message vượt số lần xử lý cho phép được chuyển vào DLQ nếu có.
- Không xóa message trước khi xử lý nghiệp vụ hoàn tất.
- Có cơ chế xử lý partial batch failure nếu Lambda đọc message theo batch.

---



### Quy định kiểm tra cập nhật đồng thời trong DynamoDB

`la-bid-processor` phải sử dụng conditional write, transaction hoặc cơ chế kiểm soát đồng thời tương đương.

Điều kiện cập nhật tối thiểu cần xác minh:

- Phiên vẫn ở trạng thái `ACTIVE`.
- Thời gian server vẫn nằm trong khoảng đấu giá.
- Giá mới cao hơn current price.
- Giá mới đáp ứng minimum increment.
- Version hoặc current price chưa bị thay đổi bởi một request khác.
- Request ID chưa được xử lý trước đó.

Không nên triển khai theo cách:

```
Đọc current price
→ kiểm tra trong Lambda
→ cập nhật không có điều kiện
```

Cách này có thể làm một bid thấp hơn ghi đè bid cao hơn khi hai Lambda chạy đồng thời.

Kết quả cập nhật current price và ghi bid history phải nhất quán. Nếu một thao tác thành công nhưng thao tác còn lại thất bại, hệ thống cần có transaction hoặc cơ chế phục hồi rõ ràng.

---



### Quy định kiểm tra CloudWatch Logs

Logs cần có đủ thông tin truy vết:

- Request ID.
- Lambda Request ID.
- WebSocket route key.
- Auction ID.
- Item ID.
- User ID đã xác minh.
- Bid amount.
- Message ID của SQS.
- Message Group ID.
- Receive count.
- Kết quả idempotency.
- Giá trước và sau cập nhật.
- Kết quả conditional write.
- Loại sự kiện broadcast.
- Số kết nối mục tiêu.
- Số lần broadcast thành công và thất bại.
- Thời gian xử lý từng thành phần.
- Error code khi bid bị từ chối.

Logs không được chứa:

- Access Token.
- ID Token.
- Refresh Token.
- Authorization header.
- Mật khẩu.
- AWS credentials.
- Toàn bộ dữ liệu cá nhân.
- Stack trace trong response trả về frontend.

---

### Bảng tổng hợp kết quả


| Mã       | Nội dung kiểm thử               | Kết quả chính mong đợi                       | Trạng thái    |
| -------- | ------------------------------- | -------------------------------------------- | ------------- |
| `BID-01` | Đặt giá hợp lệ                  | Giá, highest bidder và history được cập nhật | Chưa kiểm thử |
| `BID-02` | Giá bằng hoặc thấp hơn hiện tại | Bị từ chối, dữ liệu không thay đổi           | Chưa kiểm thử |
| `BID-03` | Không đủ bước giá               | Bị từ chối với lỗi phù hợp                   | Chưa kiểm thử |
| `BID-04` | Phiên chưa bắt đầu              | Không chấp nhận bid                          | Chưa kiểm thử |
| `BID-05` | Phiên đã kết thúc               | Không chấp nhận bid                          | Chưa kiểm thử |
| `BID-06` | User không hợp lệ               | Bị từ chối xác thực                          | Chưa kiểm thử |
| `BID-07` | User không thuộc phiên          | Bị từ chối phân quyền                        | Chưa kiểm thử |
| `BID-08` | Gửi lại cùng request            | Chỉ xử lý một lần                            | Chưa kiểm thử |
| `BID-09` | Nhiều User đặt giá lần lượt     | Xử lý đúng thứ tự                            | Chưa kiểm thử |
| `BID-10` | Cập nhật highest bidder         | Highest bidder luôn chính xác                | Chưa kiểm thử |
| `BID-11` | Lưu bid history                 | Lưu đủ, đúng và không trùng                  | Chưa kiểm thử |
| `BID-12` | Broadcast kết quả               | Đúng User, đúng Room                         | Chưa kiểm thử |
| `BID-13` | Cập nhật frontend               | Hiển thị giá mới không cần reload            | Chưa kiểm thử |


---

