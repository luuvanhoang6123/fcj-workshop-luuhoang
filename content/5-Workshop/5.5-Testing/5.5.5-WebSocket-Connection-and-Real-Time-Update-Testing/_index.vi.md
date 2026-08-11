---
title: "Kiểm thử kết nối WebSocket và cập nhật thời gian thực"
date: 2026-08-03
weight: 5
chapter: false
pre: " <b> 5.5.5. </b> "
---

#### Mục tiêu kiểm thử

Phần này kiểm tra khả năng kết nối và cập nhật dữ liệu thời gian thực của hệ thống đấu giá thông qua:

- Amazon API Gateway WebSocket API.
- Lambda WebSocket Handler.
- Các route `$connect`, `$disconnect` và `$default`.
- Route tham gia phòng đấu giá và gửi thông điệp.
- Bảng quản lý kết nối trong Amazon DynamoDB.
- Lambda Broadcast.
- API Gateway Management API.
- Amazon CloudWatch Logs.
- Frontend của hệ thống đấu giá.

Các test case được đánh mã từ `WS-01` đến `WS-13`.

Một test case WebSocket chỉ được đánh dấu `PASS` khi kiểm tra được đồng thời:

1. Client nhận đúng trạng thái kết nối hoặc thông điệp.
2. WebSocket Handler thực hiện đúng nghiệp vụ.
3. Connection ID được lưu, cập nhật hoặc xóa đúng trong DynamoDB.
4. Lambda Broadcast gửi đúng người và đúng phòng đấu giá.
5. Kết nối lỗi hoặc hết hạn không làm thất bại toàn bộ quá trình broadcast.
6. Không có dữ liệu riêng của phòng đấu giá bị gửi nhầm sang người dùng khác.
7. Có bằng chứng trực tiếp từ trình duyệt, DynamoDB hoặc CloudWatch Logs.

---

#### Phạm vi kiểm thử

Các thành phần được kiểm tra gồm:

- User Frontend.
- Amazon API Gateway WebSocket API.
- Route `$connect`.
- Route `$disconnect`.
- Route `$default`.
- Route tham gia hoặc rời phòng đấu giá.
- Lambda WebSocket Handler.
- Lambda Broadcast.
- API Gateway Management API.
- DynamoDB Connections Table.
- DynamoDB Auction Room hoặc Subscription Table nếu được tách riêng.
- Amazon CloudWatch Logs.
- Amazon Cognito hoặc cơ chế xác thực WebSocket của hệ thống.

---

#### Điều kiện kiểm thử chung

Trước khi thực hiện kiểm thử, hệ thống cần đáp ứng các điều kiện sau:

- WebSocket API đã được triển khai trên API Gateway.
- WebSocket URL của môi trường kiểm thử đã được cấu hình trên frontend.
- Các route `$connect`, `$disconnect` và `$default` đã được liên kết với đúng Lambda.
- Các route nghiệp vụ như `join_room`, `leave_room` hoặc `send_message` đã được triển khai nếu kiến trúc sử dụng route riêng.
- Lambda có quyền đọc, ghi và xóa dữ liệu trong bảng kết nối DynamoDB.
- Lambda Broadcast có quyền gọi `execute-api:ManageConnections`.
- CloudWatch Logs đã được bật cho các Lambda liên quan.
- Có ít nhất hai tài khoản User hợp lệ.
- Có ít nhất hai vật phẩm hoặc phòng đấu giá khác nhau.
- Có thể mở hai cửa sổ trình duyệt hoặc hai phiên trình duyệt độc lập.
- Có thể kiểm tra trực tiếp bản ghi trong DynamoDB.
- Môi trường kiểm thử được tách khỏi dữ liệu production.
- Đồng hồ trên thiết bị kiểm thử được đồng bộ để đối chiếu thời gian log.

Nếu Lambda, route, bảng DynamoDB hoặc chức năng frontend liên quan chưa được triển khai, test case phải được đánh dấu `BLOCKED`.

---

#### Dữ liệu kiểm thử


| Dữ liệu                 | Mô tả                                                              |
| ----------------------- | ------------------------------------------------------------------ |
| User A                  | Tài khoản hợp lệ tham gia vật phẩm A                               |
| User B                  | Tài khoản hợp lệ tham gia cùng vật phẩm A                          |
| User C                  | Tài khoản hợp lệ nhưng chỉ tham gia vật phẩm B                     |
| Auction Item A          | Vật phẩm có phòng WebSocket hợp lệ                                 |
| Auction Item B          | Vật phẩm khác với Item A                                           |
| Room A                  | Phòng WebSocket của Auction Item A                                 |
| Room B                  | Phòng WebSocket của Auction Item B                                 |
| Connection ID hợp lệ    | Connection ID đang hoạt động do API Gateway cấp                    |
| Connection ID hết hạn   | Connection ID của client đã đóng hoặc mất kết nối                  |
| Thông điệp hợp lệ       | JSON có đúng `action`, `roomId` và các trường bắt buộc             |
| Thông điệp không hợp lệ | JSON lỗi cú pháp, thiếu trường hoặc chứa action không hỗ trợ       |
| Sự kiện hợp lệ          | Cập nhật trạng thái, giá đấu, số người xem hoặc thông báo hệ thống |
| Token hợp lệ            | Token chưa hết hạn và thuộc tài khoản hợp lệ                       |
| Token không hợp lệ      | Token sai chữ ký, hết hạn hoặc không đúng định dạng                |


Không được đưa Access Token, ID Token, Refresh Token hoặc header xác thực vào ảnh chụp và báo cáo.

---

#### Quy ước trạng thái và cấu trúc thông điệp

WebSocket không sử dụng HTTP status cho tất cả thông điệp sau khi kết nối được thiết lập. Vì vậy, kết quả cần được kiểm tra ở hai giai đoạn:

- Giai đoạn bắt tay kết nối: kiểm tra HTTP status của WebSocket handshake.
- Giai đoạn đã kết nối: kiểm tra WebSocket message và trạng thái kết nối.

Các kết quả handshake thông dụng:


| Kết quả                   | Ý nghĩa                                                |
| ------------------------- | ------------------------------------------------------ |
| `101 Switching Protocols` | Kết nối WebSocket được thiết lập thành công            |
| `401 Unauthorized`        | Không có thông tin xác thực hoặc xác thực không hợp lệ |
| `403 Forbidden`           | Người dùng đã xác thực nhưng không được phép kết nối   |
| Kết nối bị đóng           | API Gateway hoặc Lambda từ chối hoặc kết thúc kết nối  |


Thông điệp thành công nên có cấu trúc nhất quán, ví dụ:

```json
{
  "type": "AUCTION_STATUS_UPDATED",
  "roomId": "auction-item-a",
  "data": {
    "status": "ACTIVE"
  },
  "timestamp": "2026-08-09T12:00:00Z"
}
```

Thông điệp lỗi nên có cấu trúc mà frontend có thể xử lý:

```json
{
  "type": "ERROR",
  "error": {
    "code": "INVALID_MESSAGE_FORMAT",
    "message": "The WebSocket message is invalid"
  },
  "timestamp": "2026-08-09T12:00:00Z"
}
```

Thông điệp gửi đến client không được chứa:

- Access Token.
- ID Token.
- Refresh Token.
- AWS credentials.
- Connection ID của người dùng khác.
- Stack trace.
- Tên bảng DynamoDB.
- Chi tiết hạ tầng nội bộ không cần thiết.
- Dữ liệu cá nhân của người dùng khác.

---

#### WS-01 — User kết nối WebSocket thành công


| Trường                   | Nội dung                                                                                                                                                                                                                                   |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `WS-01`                                                                                                                                                                                                                                    |
| **Tên kiểm thử**         | User hợp lệ kết nối WebSocket thành công                                                                                                                                                                                                   |
| **Mục tiêu**             | Xác minh User đã xác thực có thể thiết lập kết nối với API Gateway WebSocket.                                                                                                                                                              |
| **Điều kiện tiên quyết** | WebSocket API và route `$connect` đã được triển khai; User A có thông tin xác thực hợp lệ.                                                                                                                                                 |
| **Các bước thực hiện**   | 1. Đăng nhập bằng User A. 2. Mở trang chi tiết Auction Item A. 3. Mở Network tab và chọn bộ lọc WebSocket. 4. Quan sát quá trình WebSocket handshake. 5. Kiểm tra trạng thái kết nối trên frontend. 6. Kiểm tra log của `$connect` Lambda. |
| **Dữ liệu đầu vào**      | WebSocket URL hợp lệ và thông tin xác thực hợp lệ của User A.                                                                                                                                                                              |
| **Kết quả mong đợi**     | Handshake trả `101 Switching Protocols`; frontend chuyển sang trạng thái Connected hoặc Live; `$connect` Lambda được gọi đúng một lần; không xuất hiện reconnect liên tục; log không chứa token.                                           |
| **Kết quả thực tế**      | Điền HTTP status, trạng thái kết nối, thời gian kết nối và Request ID thực tế.                                                                                                                                                             |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                             |
| **Bằng chứng**           | Network tab, trạng thái Live trên frontend và CloudWatch Logs của `$connect`.                                                                                                                                                              |


---

#### WS-02 — Kết nối không hợp lệ bị từ chối


| Trường                   | Nội dung                                                                                                                                                                                                                |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-02`                                                                                                                                                                                                                 |
| **Tên kiểm thử**         | Từ chối kết nối WebSocket không hợp lệ                                                                                                                                                                                  |
| **Mục tiêu**             | Xác minh người dùng không có thông tin xác thực hợp lệ không thể thiết lập kết nối.                                                                                                                                     |
| **Điều kiện tiên quyết** | `$connect` có kiểm tra xác thực hoặc Authorizer đã được cấu hình.                                                                                                                                                       |
| **Các bước thực hiện**   | 1. Thử kết nối không có thông tin xác thực. 2. Thử kết nối bằng token sai định dạng. 3. Thử kết nối bằng token hết hạn. 4. Ghi nhận kết quả handshake. 5. Kiểm tra DynamoDB. 6. Kiểm tra CloudWatch Logs.               |
| **Dữ liệu đầu vào**      | Không có token, token không hợp lệ hoặc token hết hạn.                                                                                                                                                                  |
| **Kết quả mong đợi**     | Kết nối bị từ chối bằng `401`, `403` hoặc bị đóng theo hợp đồng hệ thống; frontend không hiển thị trạng thái Live; không lưu Connection ID hoạt động trong DynamoDB; log ghi error code nhưng không ghi nội dung token. |
| **Kết quả thực tế**      | Điền từng loại dữ liệu thử nghiệm và kết quả tương ứng.                                                                                                                                                                 |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                          |
| **Bằng chứng**           | Kết quả handshake, DynamoDB không có bản ghi kết nối hợp lệ và CloudWatch Logs đã che dữ liệu nhạy cảm.                                                                                                                 |


Nếu token được truyền qua query string, nhóm phải xác minh token không bị ghi vào access log, browser history hoặc ảnh chụp bằng chứng. Không được công khai WebSocket URL chứa token.

---

#### WS-03 — Sự kiện `$connect` lưu Connection ID


| Trường                   | Nội dung                                                                                                                                                                                                                                                                               |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-03`                                                                                                                                                                                                                                                                                |
| **Tên kiểm thử**         | Lưu thông tin kết nối khi `$connect` thành công                                                                                                                                                                                                                                        |
| **Mục tiêu**             | Xác minh `$connect` Lambda lưu Connection ID và thông tin người dùng chính xác trong DynamoDB.                                                                                                                                                                                         |
| **Điều kiện tiên quyết** | User A có thể kết nối; Lambda có quyền ghi vào Connections Table.                                                                                                                                                                                                                      |
| **Các bước thực hiện**   | 1. Ghi nhận dữ liệu trong bảng trước khi kết nối. 2. User A mở trang Auction Item A. 3. Xác nhận kết nối thành công. 4. Tìm bản ghi mới trong DynamoDB. 5. Đối chiếu thời gian tạo, User ID và Connection ID với CloudWatch Logs. 6. Kiểm tra thuộc tính hết hạn nếu bảng sử dụng TTL. |
| **Dữ liệu đầu vào**      | Kết nối hợp lệ của User A.                                                                                                                                                                                                                                                             |
| **Kết quả mong đợi**     | Một bản ghi kết nối được tạo; Connection ID không rỗng; User ID được lấy từ danh tính đã xác minh; `connectedAt` được ghi đúng; TTL nằm trong tương lai nếu có; không lưu token trong DynamoDB.                                                                                        |
| **Kết quả thực tế**      | Điền Connection ID đã che một phần, User ID, thời gian tạo và TTL.                                                                                                                                                                                                                     |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                         |
| **Bằng chứng**           | Bản ghi DynamoDB và CloudWatch Logs của `$connect`.                                                                                                                                                                                                                                    |


---

#### WS-04 — User tham gia đúng phòng đấu giá


| Trường                   | Nội dung                                                                                                                                                                                                                                                        |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-04`                                                                                                                                                                                                                                                         |
| **Tên kiểm thử**         | Tham gia phòng WebSocket của đúng vật phẩm                                                                                                                                                                                                                      |
| **Mục tiêu**             | Xác minh User A được liên kết với Room A khi mở Auction Item A.                                                                                                                                                                                                 |
| **Điều kiện tiên quyết** | User A đã kết nối; Auction Item A tồn tại; route tham gia phòng đã được triển khai.                                                                                                                                                                             |
| **Các bước thực hiện**   | 1. User A mở trang Auction Item A. 2. Frontend gửi thông điệp `join_room` nếu kiến trúc yêu cầu. 3. Kiểm tra thông điệp phản hồi. 4. Kiểm tra bản ghi kết nối trong DynamoDB. 5. Kiểm tra log của WebSocket Handler. 6. Phát một sự kiện thử nghiệm vào Room A. |
| **Dữ liệu đầu vào**      | Room ID hoặc Auction Item ID của Item A.                                                                                                                                                                                                                        |
| **Kết quả mong đợi**     | Kết nối của User A được gắn với Room A; Lambda xác minh Room A tồn tại; client nhận xác nhận tham gia; sự kiện của Room A được gửi đến User A; không tạo nhiều subscription trùng cho cùng kết nối và phòng.                                                    |
| **Kết quả thực tế**      | Điền Room ID, phản hồi nhận được và dữ liệu DynamoDB.                                                                                                                                                                                                           |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                  |
| **Bằng chứng**           | WebSocket frame, bản ghi DynamoDB và CloudWatch Logs.                                                                                                                                                                                                           |


---

#### WS-05 — Hai người dùng cùng tham gia một vật phẩm


| Trường                   | Nội dung                                                                                                                                                                                                                                                                                            |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-05`                                                                                                                                                                                                                                                                                             |
| **Tên kiểm thử**         | Hai User cùng tham gia Room A                                                                                                                                                                                                                                                                       |
| **Mục tiêu**             | Xác minh hệ thống quản lý đồng thời nhiều kết nối trong cùng một phòng đấu giá.                                                                                                                                                                                                                     |
| **Điều kiện tiên quyết** | Có User A và User B; có thể mở hai phiên trình duyệt độc lập.                                                                                                                                                                                                                                       |
| **Các bước thực hiện**   | 1. Mở Auction Item A bằng User A trong cửa sổ thứ nhất. 2. Mở cùng Item A bằng User B trong cửa sổ thứ hai. 3. Xác nhận cả hai kết nối đều ở trạng thái Live. 4. Kiểm tra số người đang xem nếu frontend hỗ trợ. 5. Kiểm tra các bản ghi trong DynamoDB. 6. Phát một sự kiện thử nghiệm vào Room A. |
| **Dữ liệu đầu vào**      | Hai User khác nhau và cùng Room A.                                                                                                                                                                                                                                                                  |
| **Kết quả mong đợi**     | Hai Connection ID khác nhau được liên kết với Room A; số người xem là `2` nếu có chức năng viewer count; cả hai cửa sổ nhận được sự kiện của Room A; không ghi đè kết nối của nhau.                                                                                                                 |
| **Kết quả thực tế**      | Điền số kết nối, số người xem và thông điệp nhận được ở từng cửa sổ.                                                                                                                                                                                                                                |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                      |
| **Bằng chứng**           | Ảnh hai cửa sổ hoạt động đồng thời, WebSocket frames, DynamoDB và log broadcast.                                                                                                                                                                                                                    |


---

#### WS-06 — User gửi thông điệp hợp lệ


| Trường                   | Nội dung                                                                                                                                                                                                                                                                      |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-06`                                                                                                                                                                                                                                                                       |
| **Tên kiểm thử**         | Xử lý WebSocket message hợp lệ                                                                                                                                                                                                                                                |
| **Mục tiêu**             | Xác minh WebSocket Handler nhận, xác thực và xử lý đúng thông điệp hợp lệ.                                                                                                                                                                                                    |
| **Điều kiện tiên quyết** | User A đã kết nối và tham gia Room A.                                                                                                                                                                                                                                         |
| **Các bước thực hiện**   | 1. User A gửi một thông điệp có action được hỗ trợ. 2. Kiểm tra frame đã gửi. 3. Kiểm tra phản hồi của server. 4. Kiểm tra CloudWatch Logs. 5. Nếu thông điệp gây broadcast, kiểm tra kết quả ở User B. 6. Nếu thông điệp thay đổi dữ liệu, kiểm tra dữ liệu nguồn tương ứng. |
| **Dữ liệu đầu vào**      | JSON hợp lệ với đúng `action`, `roomId` và các trường bắt buộc.                                                                                                                                                                                                               |
| **Kết quả mong đợi**     | Handler đọc đúng action; xác minh User và Room; xử lý nghiệp vụ một lần; client nhận ACK hoặc kết quả phù hợp; các User liên quan nhận đúng sự kiện; không để client tự xác định danh tính hoặc quyền hạn bằng dữ liệu trong message.                                         |
| **Kết quả thực tế**      | Điền message type, phản hồi và kết quả broadcast thực tế.                                                                                                                                                                                                                     |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                |
| **Bằng chứng**           | WebSocket frames đã che dữ liệu nhạy cảm, CloudWatch Logs và dữ liệu liên quan.                                                                                                                                                                                               |


---

#### WS-07 — Thông điệp sai định dạng bị từ chối


| Trường                   | Nội dung                                                                                                                                                                                                                                           |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-07`                                                                                                                                                                                                                                            |
| **Tên kiểm thử**         | Từ chối WebSocket message không hợp lệ                                                                                                                                                                                                             |
| **Mục tiêu**             | Xác minh Lambda xử lý an toàn khi message không phải JSON hợp lệ, thiếu trường hoặc chứa action không hỗ trợ.                                                                                                                                      |
| **Điều kiện tiên quyết** | User A đã kết nối WebSocket.                                                                                                                                                                                                                       |
| **Các bước thực hiện**   | 1. Gửi chuỗi không phải JSON. 2. Gửi JSON thiếu `action`. 3. Gửi action không được hỗ trợ. 4. Gửi message thiếu `roomId` khi trường này bắt buộc. 5. Gửi trường sai kiểu dữ liệu. 6. Kiểm tra phản hồi, log và dữ liệu sau mỗi lần.                |
| **Dữ liệu đầu vào**      | Message sai cú pháp hoặc vi phạm schema.                                                                                                                                                                                                           |
| **Kết quả mong đợi**     | Server trả message lỗi có cấu trúc nhất quán; không thực hiện nghiệp vụ; không broadcast message không hợp lệ; không thay đổi dữ liệu ngoài ý muốn; Lambda không bị lỗi chưa xử lý; kết nối có thể được giữ hoặc đóng theo hợp đồng đã định nghĩa. |
| **Kết quả thực tế**      | Điền từng message kiểm thử, error code và trạng thái kết nối sau lỗi.                                                                                                                                                                              |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                     |
| **Bằng chứng**           | Frame gửi và nhận, DynamoDB hoặc dữ liệu nghiệp vụ không thay đổi, CloudWatch Logs.                                                                                                                                                                |


---

#### WS-08 — Trạng thái đấu giá được gửi đến tất cả người tham gia


| Trường                   | Nội dung                                                                                                                                                                                                                                                   |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-08`                                                                                                                                                                                                                                                    |
| **Tên kiểm thử**         | Broadcast cập nhật trạng thái đến toàn bộ Room A                                                                                                                                                                                                           |
| **Mục tiêu**             | Xác minh Lambda Broadcast gửi cập nhật trạng thái đấu giá đến mọi kết nối đang hoạt động trong đúng phòng.                                                                                                                                                 |
| **Điều kiện tiên quyết** | User A và User B đang tham gia Room A; Lambda Broadcast và Management API đã sẵn sàng.                                                                                                                                                                     |
| **Các bước thực hiện**   | 1. Mở hai cửa sổ tại Auction Item A. 2. Thực hiện nghiệp vụ thay đổi trạng thái hoặc tạo sự kiện hợp lệ. 3. Ghi nhận thời điểm phát sự kiện. 4. Quan sát message tại cả hai cửa sổ. 5. Kiểm tra giao diện được cập nhật. 6. Kiểm tra log Lambda Broadcast. |
| **Dữ liệu đầu vào**      | Sự kiện như `AUCTION_STATUS_UPDATED`, `BID_UPDATED` hoặc `VIEWER_COUNT_UPDATED`.                                                                                                                                                                           |
| **Kết quả mong đợi**     | Cả User A và User B nhận cùng loại sự kiện, Room ID và dữ liệu mới; frontend cập nhật mà không cần tải lại trang; không gửi trùng ngoài số lần được thiết kế; log cho biết tổng số kết nối mục tiêu, số lần thành công và thất bại.                        |
| **Kết quả thực tế**      | Điền message nhận được ở từng cửa sổ và độ trễ quan sát được.                                                                                                                                                                                              |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                             |
| **Bằng chứng**           | Ảnh hai cửa sổ, WebSocket messages và CloudWatch Logs của Lambda Broadcast.                                                                                                                                                                                |


---

#### WS-09 — Một User rời trang hoặc ngắt kết nối


| Trường                   | Nội dung                                                                                                                                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `WS-09`                                                                                                                                                                                                                                          |
| **Tên kiểm thử**         | User rời phòng đấu giá                                                                                                                                                                                                                           |
| **Mục tiêu**             | Xác minh hệ thống xử lý đúng khi một User đóng trang, chuyển trang hoặc mất kết nối.                                                                                                                                                             |
| **Điều kiện tiên quyết** | User A và User B đang cùng tham gia Room A.                                                                                                                                                                                                      |
| **Các bước thực hiện**   | 1. Xác nhận hai User đang kết nối. 2. Đóng tab của User B hoặc chuyển khỏi trang Item A. 3. Quan sát trạng thái trên cửa sổ User A. 4. Kiểm tra viewer count hoặc thông báo rời phòng nếu có. 5. Kiểm tra CloudWatch Logs. 6. Kiểm tra DynamoDB. |
| **Dữ liệu đầu vào**      | Hành động đóng tab, chuyển trang hoặc ngắt mạng của User B.                                                                                                                                                                                      |
| **Kết quả mong đợi**     | User B không tiếp tục được xem là thành viên hoạt động của Room A; viewer count giảm từ `2` xuống `1` nếu có; User A nhận sự kiện cập nhật tương ứng; hệ thống không ảnh hưởng đến kết nối của User A.                                           |
| **Kết quả thực tế**      | Điền trạng thái hai client, viewer count và thời điểm phát hiện ngắt kết nối.                                                                                                                                                                    |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                   |
| **Bằng chứng**           | Ảnh trước và sau khi rời trang, frame của User A, DynamoDB và CloudWatch Logs.                                                                                                                                                                   |


Khi thiết bị mất mạng đột ngột, `$disconnect` có thể không được xử lý ngay. Nếu hệ thống sử dụng heartbeat hoặc TTL, cần ghi rõ khoảng thời gian tối đa để phát hiện và dọn kết nối.

---

#### WS-10 — Sự kiện `$disconnect` xóa hoặc vô hiệu hóa kết nối


| Trường                   | Nội dung                                                                                                                                                                                                                                                             |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-10`                                                                                                                                                                                                                                                              |
| **Tên kiểm thử**         | Dọn bản ghi kết nối khi `$disconnect` xảy ra                                                                                                                                                                                                                         |
| **Mục tiêu**             | Xác minh `$disconnect` Lambda xóa hoặc đánh dấu không hoạt động đối với Connection ID đã ngắt.                                                                                                                                                                       |
| **Điều kiện tiên quyết** | User B có Connection ID đang được lưu trong DynamoDB.                                                                                                                                                                                                                |
| **Các bước thực hiện**   | 1. Ghi nhận bản ghi User B trước khi ngắt kết nối. 2. Đóng kết nối WebSocket của User B. 3. Kiểm tra log `$disconnect`. 4. Đọc lại bản ghi trong DynamoDB. 5. Thực hiện broadcast tiếp theo tới Room A. 6. Kiểm tra danh sách kết nối được Lambda Broadcast sử dụng. |
| **Dữ liệu đầu vào**      | Connection ID đang hoạt động của User B.                                                                                                                                                                                                                             |
| **Kết quả mong đợi**     | `$disconnect` nhận đúng Connection ID; bản ghi bị xóa hoặc chuyển sang trạng thái inactive theo thiết kế; User B không còn trong danh sách broadcast; thao tác dọn dẹp có tính idempotent và không lỗi khi chạy lại.                                                 |
| **Kết quả thực tế**      | Điền trạng thái bản ghi trước và sau `$disconnect`.                                                                                                                                                                                                                  |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                       |
| **Bằng chứng**           | CloudWatch Logs của `$disconnect`, bản ghi DynamoDB trước và sau và log lần broadcast kế tiếp.                                                                                                                                                                       |


---

#### WS-11 — Kết nối hết hạn không làm broadcast thất bại toàn bộ


| Trường                   | Nội dung                                                                                                                                                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `WS-11`                                                                                                                                                                                                                                                      |
| **Tên kiểm thử**         | Cô lập lỗi của một kết nối hết hạn                                                                                                                                                                                                                           |
| **Mục tiêu**             | Xác minh một Connection ID không còn hợp lệ không ngăn Lambda gửi dữ liệu đến các kết nối còn hoạt động.                                                                                                                                                     |
| **Điều kiện tiên quyết** | Room A có ít nhất một kết nối hợp lệ và một Connection ID đã hết hạn hoặc không còn tồn tại.                                                                                                                                                                 |
| **Các bước thực hiện**   | 1. Chuẩn bị User A đang kết nối. 2. Tạo điều kiện để bản ghi cũ của User B vẫn còn trong DynamoDB. 3. Phát một sự kiện đến Room A. 4. Quan sát message tại User A. 5. Kiểm tra kết quả Lambda Broadcast. 6. Kiểm tra log lỗi cho Connection ID cũ.           |
| **Dữ liệu đầu vào**      | Một Connection ID hợp lệ và một Connection ID hết hạn trong cùng Room A.                                                                                                                                                                                     |
| **Kết quả mong đợi**     | User A vẫn nhận được sự kiện; Lambda xử lý lỗi theo từng Connection ID; một lần gửi thất bại không kết thúc toàn bộ vòng broadcast; kết quả log thể hiện ít nhất một lần thành công và một lần thất bại; Lambda không trả lỗi toàn bộ chỉ vì một kết nối cũ. |
| **Kết quả thực tế**      | Điền số kết nối mục tiêu, số lần gửi thành công, thất bại và kết quả tại User A.                                                                                                                                                                             |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                               |
| **Bằng chứng**           | Message của User A, CloudWatch Logs và bản ghi Connection ID hết hạn.                                                                                                                                                                                        |


---

#### WS-12 — `GoneException` được xử lý và dọn khỏi DynamoDB


| Trường                   | Nội dung                                                                                                                                                                                                                                                              |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-12`                                                                                                                                                                                                                                                               |
| **Tên kiểm thử**         | Dọn kết nối cũ khi Management API trả `GoneException`                                                                                                                                                                                                                 |
| **Mục tiêu**             | Xác minh Lambda Broadcast nhận biết lỗi HTTP `410 Gone` và xóa Connection ID không còn hợp lệ.                                                                                                                                                                        |
| **Điều kiện tiên quyết** | DynamoDB còn một Connection ID của kết nối đã đóng; Lambda Broadcast có quyền xóa hoặc cập nhật bản ghi.                                                                                                                                                              |
| **Các bước thực hiện**   | 1. Xác định một Connection ID đã hết hạn. 2. Xác nhận bản ghi vẫn tồn tại trước broadcast. 3. Kích hoạt Lambda Broadcast. 4. Kiểm tra log của lệnh `postToConnection`. 5. Kiểm tra việc bắt `GoneException`. 6. Đọc lại DynamoDB. 7. Kích hoạt broadcast lần thứ hai. |
| **Dữ liệu đầu vào**      | Connection ID không còn tồn tại trong API Gateway nhưng vẫn còn trong DynamoDB.                                                                                                                                                                                       |
| **Kết quả mong đợi**     | Management API trả lỗi tương đương `410 Gone`; Lambda bắt lỗi mà không dừng toàn bộ broadcast; bản ghi kết nối cũ bị xóa hoặc vô hiệu hóa; lần broadcast tiếp theo không tiếp tục gửi đến Connection ID đó.                                                           |
| **Kết quả thực tế**      | Điền mã lỗi, hành động dọn dẹp và trạng thái bản ghi sau xử lý.                                                                                                                                                                                                       |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                        |
| **Bằng chứng**           | CloudWatch Logs có `GoneException` hoặc `410`, DynamoDB trước và sau và log lần broadcast tiếp theo.                                                                                                                                                                  |


Không được coi mọi lỗi từ Management API là kết nối hết hạn. Chỉ những lỗi xác định kết nối không còn tồn tại, chẳng hạn `GoneException`, mới được dùng làm căn cứ xóa bản ghi.

---

#### WS-13 — Người ngoài phòng không nhận dữ liệu riêng của phòng


| Trường                   | Nội dung                                                                                                                                                                                                                                                                |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `WS-13`                                                                                                                                                                                                                                                                 |
| **Tên kiểm thử**         | Cô lập dữ liệu giữa các phòng đấu giá                                                                                                                                                                                                                                   |
| **Mục tiêu**             | Xác minh sự kiện của Room A chỉ được gửi đến các kết nối thuộc Room A.                                                                                                                                                                                                  |
| **Điều kiện tiên quyết** | User A và User B tham gia Room A; User C tham gia Room B.                                                                                                                                                                                                               |
| **Các bước thực hiện**   | 1. Mở Room A bằng User A và User B. 2. Mở Room B bằng User C. 3. Xác nhận ba kết nối hoạt động. 4. Phát một sự kiện chỉ thuộc Room A. 5. Kiểm tra message tại cả ba cửa sổ. 6. Kiểm tra truy vấn DynamoDB của Lambda Broadcast. 7. Lặp lại bằng một sự kiện của Room B. |
| **Dữ liệu đầu vào**      | Sự kiện riêng của Auction Item A và sự kiện riêng của Auction Item B.                                                                                                                                                                                                   |
| **Kết quả mong đợi**     | User A và User B nhận sự kiện của Room A; User C không nhận sự kiện đó; User C chỉ nhận sự kiện của Room B; Lambda truy vấn kết nối theo đúng Room ID; không có dữ liệu giá đấu, trạng thái hoặc thông tin riêng của phiên bị gửi chéo.                                 |
| **Kết quả thực tế**      | Điền message nhận được hoặc không nhận được ở từng cửa sổ.                                                                                                                                                                                                              |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                          |
| **Bằng chứng**           | Ảnh ba cửa sổ hoặc các WebSocket frames, truy vấn/log broadcast và dữ liệu subscription trong DynamoDB.                                                                                                                                                                 |


---

#### Ma trận phân phối sự kiện cần kiểm tra


| Người dùng         | Phòng đã tham gia      | Sự kiện Room A           | Sự kiện Room B                  |
| ------------------ | ---------------------- | ------------------------ | ------------------------------- |
| User A             | Room A                 | Phải nhận                | Không được nhận                 |
| User B             | Room A                 | Phải nhận                | Không được nhận                 |
| User C             | Room B                 | Không được nhận          | Phải nhận                       |
| Kết nối đã hết hạn | Room A                 | Gửi thất bại và được dọn | Không áp dụng                   |
| User đã rời Room A | Không còn subscription | Không được nhận          | Chỉ nhận nếu đã tham gia Room B |


---

#### Quy định kiểm tra DynamoDB Connections Table

Đối với bảng quản lý kết nối, cần kiểm tra:

- Connection ID được lấy từ `requestContext.connectionId`.
- User ID được lấy từ danh tính đã xác minh.
- Không tin tưởng `userId`, `role` hoặc `connectionId` do client gửi trong message.
- Mỗi kết nối được liên kết với đúng User.
- Mỗi subscription được liên kết với đúng Room ID hoặc Auction Item ID.
- Hai tab của cùng một User có thể có hai Connection ID khác nhau.
- Kết nối bị đóng được xóa hoặc vô hiệu hóa.
- TTL được thiết lập đúng nếu hệ thống sử dụng cơ chế tự động dọn dữ liệu.
- Không lưu Access Token, ID Token hoặc Refresh Token.
- Không ghi đè kết nối của người dùng khác.
- Không tồn tại subscription trùng ngoài thiết kế.
- `GoneException` dẫn đến việc dọn Connection ID tương ứng.
- Broadcast chỉ truy vấn kết nối thuộc phòng mục tiêu.
- Lỗi của một kết nối không làm mất cập nhật của các kết nối còn lại.

Nếu bảng sử dụng thiết kế single-table, nhóm phải kiểm tra đúng partition key, sort key và các index dùng để truy vấn kết nối theo Room ID.

---

#### Quy định kiểm tra Lambda Broadcast

Lambda Broadcast phải được kiểm tra theo các tiêu chí sau:

- Nhận đúng Room ID và loại sự kiện.
- Truy vấn đúng danh sách kết nối của phòng.
- Không broadcast dựa hoàn toàn vào danh sách Connection ID do client cung cấp.
- Gửi dữ liệu bằng đúng WebSocket API endpoint và stage.
- Message có cấu trúc JSON hợp lệ.
- Không gửi dữ liệu nhạy cảm.
- Tiếp tục xử lý khi một kết nối gửi thất bại.
- Bắt và xử lý `GoneException`.
- Dọn Connection ID hết hạn.
- Ghi nhận số kết nối mục tiêu.
- Ghi nhận số lần gửi thành công và thất bại.
- Không ghi toàn bộ token hoặc dữ liệu cá nhân vào log.
- Có cơ chế giới hạn kích thước message.
- Không gửi trùng sự kiện ngoài thiết kế.
- Không để lỗi của một phòng ảnh hưởng đến phòng khác.

---

#### Quy định kiểm tra CloudWatch Logs

CloudWatch Logs nên chứa các thông tin cần thiết để truy vết:

- Request ID.
- Route key như `$connect`, `$disconnect`, `$default` hoặc `join_room`.
- Connection ID đã che một phần khi đưa vào báo cáo.
- User ID hoặc Cognito `sub` đã xác minh.
- Room ID hoặc Auction Item ID.
- Loại message hoặc loại sự kiện.
- Số kết nối mục tiêu.
- Số lần gửi thành công.
- Số lần gửi thất bại.
- Error code như `INVALID_MESSAGE_FORMAT` hoặc `GONE_CONNECTION`.
- Thời gian xử lý.
- Kết quả dọn bản ghi kết nối cũ.

Không được ghi:

- Access Token.
- ID Token.
- Refresh Token.
- Header hoặc query parameter chứa thông tin xác thực.
- Mật khẩu.
- AWS Access Key ID.
- AWS Secret Access Key.
- AWS Session Token.
- Toàn bộ WebSocket message nếu message chứa dữ liệu nhạy cảm.
- Stack trace trong response gửi về client.

---

#### Bảng tổng hợp kết quả


| Mã      | Nội dung kiểm thử            | Kết quả chính mong đợi                  | Kiểm tra DynamoDB   | Trạng thái  |
| ------- | ---------------------------- | --------------------------------------- | ------------------- | ----------- |
| `WS-01` | User kết nối thành công      | Handshake `101`, frontend hiển thị Live | Có                  | Đã kiểm thử |
| `WS-02` | Kết nối không hợp lệ         | Bị từ chối, không có kết nối hoạt động  | Có                  | Đã kiểm thử |
| `WS-03` | `$connect` lưu Connection ID | Bản ghi kết nối được tạo chính xác      | Bắt buộc            | Đã kiểm thử |
| `WS-04` | User tham gia đúng phòng     | Connection được gắn với đúng Room ID    | Bắt buộc            | Đã kiểm thử |
| `WS-05` | Hai User cùng một vật phẩm   | Hai kết nối cùng nhận sự kiện           | Bắt buộc            | Đã kiểm thử |
| `WS-06` | Gửi message hợp lệ           | Handler xử lý và phản hồi đúng          | Khi có thay đổi     | Đã kiểm thử |
| `WS-07` | Message sai định dạng        | Bị từ chối, không broadcast             | Phải không thay đổi | Đã kiểm thử |
| `WS-08` | Broadcast trạng thái         | Tất cả thành viên phòng nhận cập nhật   | Có                  | Đã kiểm thử |
| `WS-09` | User rời trang               | Không còn được xem là kết nối hoạt động | Có                  | Đã kiểm thử |
| `WS-10` | `$disconnect` dọn kết nối    | Xóa hoặc vô hiệu hóa Connection ID      | Bắt buộc            | Đã kiểm thử |
| `WS-11` | Có kết nối hết hạn           | Các kết nối hợp lệ vẫn nhận dữ liệu     | Có                  | Đã kiểm thử |
| `WS-12` | Xử lý `GoneException`        | Kết nối cũ được dọn khỏi bảng           | Bắt buộc            | Đã kiểm thử |
| `WS-13` | Cô lập giữa các phòng        | Người ngoài phòng không nhận dữ liệu    | Có                  | Đã kiểm thử |


---



