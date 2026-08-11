---
title: "Kiểm thử REST API và nghiệp vụ quản lý đấu giá"
date: 2026-08-03
weight: 4
chapter: false
pre: " <b> 5.5.4. </b> "
---

### Kiểm thử REST API và nghiệp vụ quản lý đấu giá

#### Mục tiêu kiểm thử

phần này kiểm tra các REST API và nghiệp vụ quản lý phiên đấu giá được triển khai thông qua Amazon API Gateway và Business Logic Lambda.

Mục tiêu kiểm thử bao gồm:

- Lấy danh sách phiên đấu giá.
- Xem chi tiết phiên đấu giá và vật phẩm.
- Tạo và cập nhật phiên đấu giá.
- Bắt đầu và kết thúc phiên đấu giá.
- Kiểm tra phân quyền giữa User và Admin.
- Kiểm tra dữ liệu đầu vào.
- Kiểm tra quy tắc chuyển trạng thái phiên đấu giá.
- Kiểm tra dữ liệu thực tế được ghi vào DynamoDB.
- Kiểm tra mã HTTP và cấu trúc JSON của API.
- Kiểm tra frontend nhận và hiển thị đúng kết quả.

Các test case được đánh mã từ `API-01` đến `API-12`.

Nhóm phải kiểm tra đồng thời:

1. API Gateway trả đúng HTTP status.
2. Response body có đúng cấu trúc JSON.
3. Business Logic Lambda thực hiện đúng nghiệp vụ.
4. Dữ liệu trong DynamoDB được tạo hoặc cập nhật chính xác.
5. Frontend nhận và hiển thị đúng kết quả.
6. Không có thay đổi dữ liệu ngoài phạm vi yêu cầu.

---

#### Phạm vi kiểm thử

Các thành phần được kiểm tra gồm:

- User Frontend.
- Admin Frontend.
- Amazon API Gateway REST API.
- API Gateway Authorizer.
- Business Logic Lambda.
- Amazon DynamoDB.
- Amazon CloudWatch Logs.
- Các Cognito claim như `sub` và `cognito:groups`.

---

#### Điều kiện kiểm thử chung

Trước khi thực hiện các test case, hệ thống cần đáp ứng các điều kiện sau:

- REST API đã được triển khai trên API Gateway.
- Các route đã được liên kết với đúng Lambda Function.
- API yêu cầu xác thực đã được bảo vệ bằng Authorizer.
- Business Logic Lambda có quyền đọc hoặc ghi đúng bảng DynamoDB.
- DynamoDB đã có bảng lưu phiên đấu giá và vật phẩm.
- User Frontend và Admin Frontend có thể gọi API.
- Có tài khoản User đã xác nhận.
- Có tài khoản Admin thuộc Cognito Group `Admin`.
- Có Access Token hợp lệ cho User và Admin.
- CloudWatch Logs đã được bật.
- Có dữ liệu phiên đấu giá ở các trạng thái cần kiểm thử.
- Môi trường kiểm thử được tách biệt với dữ liệu thật.

---

#### Dữ liệu kiểm thử


| Dữ liệu                  | Mô tả                                                                       |
| ------------------------ | --------------------------------------------------------------------------- |
| Admin hợp lệ             | Tài khoản thuộc Cognito Group `Admin`                                       |
| User hợp lệ              | Tài khoản không thuộc nhóm Admin                                            |
| Phiên `SCHEDULED`        | Phiên đã được tạo nhưng chưa bắt đầu                                        |
| Phiên `ACTIVE`           | Phiên đang diễn ra                                                          |
| Phiên `ENDED`            | Phiên đã kết thúc                                                           |
| Phiên `CANCELLED`        | Phiên đã bị hủy nếu hệ thống hỗ trợ                                         |
| Session ID hợp lệ        | ID của phiên đang tồn tại                                                   |
| Session ID không tồn tại | ID đúng định dạng nhưng không có trong DynamoDB                             |
| Item ID hợp lệ           | ID của vật phẩm đang tồn tại                                                |
| Item ID không tồn tại    | ID đúng định dạng nhưng không có trong DynamoDB                             |
| Dữ liệu tạo phiên hợp lệ | Có đầy đủ tên, thời gian bắt đầu, thời gian kết thúc và các trường bắt buộc |
| Dữ liệu không hợp lệ     | Thiếu trường, sai kiểu dữ liệu hoặc vi phạm quy tắc nghiệp vụ               |
| Thời gian hợp lệ         | `startTime` nhỏ hơn `endTime`                                               |
| Thời gian không hợp lệ   | `startTime` lớn hơn hoặc bằng `endTime`                                     |


Không đưa token, mật khẩu hoặc thông tin nhạy cảm vào tài liệu và bằng chứng kiểm thử.

---

#### Quy ước mã HTTP


| HTTP status                 | Trường hợp sử dụng                                                         |
| --------------------------- | -------------------------------------------------------------------------- |
| `200 OK`                    | Lấy dữ liệu hoặc cập nhật nghiệp vụ thành công                             |
| `201 Created`               | Tạo mới phiên đấu giá thành công                                           |
| `400 Bad Request`           | Request thiếu trường, sai định dạng hoặc vi phạm quy tắc đầu vào           |
| `401 Unauthorized`          | Không có token hoặc token không hợp lệ                                     |
| `403 Forbidden`             | Người dùng đã xác thực nhưng không có quyền thực hiện                      |
| `404 Not Found`             | Phiên đấu giá hoặc vật phẩm không tồn tại                                  |
| `409 Conflict`              | Trạng thái hiện tại không cho phép thao tác hoặc xảy ra xung đột nghiệp vụ |
| `500 Internal Server Error` | Lỗi không được xử lý bên trong hệ thống                                    |


> Dự án có thể sử dụng `400` thay cho `409` đối với chuyển trạng thái không hợp lệ. Tuy nhiên, toàn bộ API phải tuân theo một hợp đồng thống nhất và phải được ghi rõ trong tài liệu API.

---

#### Cấu trúc JSON chung cần kiểm tra

Đối với response thành công, API nên có cấu trúc nhất quán, ví dụ:

```json
{
  "data": {},
  "message": "Operation completed successfully",
  "requestId": "example-request-id"
}
```

Đối với response thất bại, API nên trả về cấu trúc có thể xử lý được:

```json
{
  "error": {
    "code": "AUCTION_SESSION_NOT_FOUND",
    "message": "Auction session was not found"
  },
  "requestId": "example-request-id"
}
```

Không được trả về cho client:

- Stack trace.
- Đường dẫn mã nguồn nội bộ.
- AWS credentials.
- Tên bảng hoặc thông tin hạ tầng không cần thiết.
- Nội dung token.
- Chi tiết lỗi có thể làm lộ cấu trúc hệ thống.

Cấu trúc thực tế có thể khác ví dụ trên nhưng phải nhất quán giữa các endpoint.

---

#### API-01 — Lấy danh sách phiên đấu giá


| Trường                   | Nội dung                                                                                                                                                                                                                                   |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `API-01`                                                                                                                                                                                                                                   |
| **Tên kiểm thử**         | Lấy danh sách phiên đấu giá                                                                                                                                                                                                                |
| **Mục tiêu**             | Xác minh API trả về danh sách phiên đấu giá từ DynamoDB và frontend hiển thị đúng dữ liệu.                                                                                                                                                 |
| **Điều kiện tiên quyết** | DynamoDB có nhiều phiên đấu giá ở các trạng thái khác nhau; endpoint lấy danh sách đã được triển khai.                                                                                                                                     |
| **Các bước thực hiện**   | 1. Ghi nhận các phiên hiện có trong DynamoDB. 2. Mở trang danh sách phiên đấu giá. 3. Gửi request lấy danh sách phiên. 4. Ghi nhận HTTP status và response JSON. 5. So sánh dữ liệu API với DynamoDB. 6. Kiểm tra danh sách trên frontend. |
| **Dữ liệu đầu vào**      | Tham số phân trang, bộ lọc trạng thái hoặc sắp xếp nếu API hỗ trợ.                                                                                                                                                                         |
| **Kết quả mong đợi**     | API trả HTTP `200`; response chứa danh sách phiên theo đúng cấu trúc; dữ liệu khớp với DynamoDB; bộ lọc, phân trang và sắp xếp hoạt động đúng; frontend hiển thị đúng tên, trạng thái và thời gian của từng phiên.                         |
| **Kết quả thực tế**      | Điền số bản ghi trả về, HTTP status, cấu trúc response và kết quả hiển thị thực tế.                                                                                                                                                        |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                             |
| **Bằng chứng**           | HTTP request/response, dữ liệu DynamoDB đối chiếu và ảnh danh sách trên frontend.                                                                                                                                                          |


---

#### API-02 — Xem chi tiết phiên đấu giá hoặc vật phẩm


| Trường                   | Nội dung                                                                                                                                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `API-02`                                                                                                                                                                                                                             |
| **Tên kiểm thử**         | Xem chi tiết phiên đấu giá và vật phẩm                                                                                                                                                                                               |
| **Mục tiêu**             | Xác minh API trả về đúng thông tin chi tiết của tài nguyên được yêu cầu.                                                                                                                                                             |
| **Điều kiện tiên quyết** | Có phiên đấu giá và vật phẩm hợp lệ trong DynamoDB.                                                                                                                                                                                  |
| **Các bước thực hiện**   | 1. Chọn một phiên hoặc vật phẩm đang tồn tại. 2. Gửi request lấy chi tiết bằng ID. 3. Ghi nhận HTTP status và response JSON. 4. Đối chiếu các trường với DynamoDB. 5. Mở trang chi tiết trên frontend. 6. Kiểm tra dữ liệu hiển thị. |
| **Dữ liệu đầu vào**      | Session ID hoặc Item ID hợp lệ.                                                                                                                                                                                                      |
| **Kết quả mong đợi**     | API trả HTTP `200`; ID trong response đúng với ID yêu cầu; các trường tên, mô tả, trạng thái, thời gian, giá và vật phẩm liên quan khớp với DynamoDB; frontend hiển thị đúng dữ liệu.                                                |
| **Kết quả thực tế**      | Điền HTTP status, ID, các trường đã đối chiếu và kết quả trên frontend.                                                                                                                                                              |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                       |
| **Bằng chứng**           | Response JSON, bản ghi DynamoDB và ảnh trang chi tiết.                                                                                                                                                                               |


---

#### API-03 — Admin tạo phiên đấu giá thành công


| Trường                   | Nội dung                                                                                                                                                                                                                                                        |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `API-03`                                                                                                                                                                                                                                                        |
| **Tên kiểm thử**         | Tạo phiên đấu giá bằng tài khoản Admin                                                                                                                                                                                                                          |
| **Mục tiêu**             | Xác minh Admin có thể tạo phiên đấu giá mới và dữ liệu được lưu chính xác.                                                                                                                                                                                      |
| **Điều kiện tiên quyết** | Admin đã đăng nhập; token có claim `cognito:groups` chứa `Admin`; endpoint tạo phiên đã được triển khai.                                                                                                                                                        |
| **Các bước thực hiện**   | 1. Ghi nhận dữ liệu trước kiểm thử. 2. Đăng nhập bằng Admin. 3. Nhập đầy đủ dữ liệu phiên trên Admin Frontend. 4. Gửi request tạo phiên. 5. Kiểm tra HTTP status và response. 6. Tìm bản ghi mới trong DynamoDB. 7. Mở lại danh sách hoặc trang chi tiết phiên. |
| **Dữ liệu đầu vào**      | Tên phiên, mô tả, `startTime`, `endTime` và các trường bắt buộc hợp lệ.                                                                                                                                                                                         |
| **Kết quả mong đợi**     | API trả HTTP `201`; response chứa ID mới; phiên được lưu đúng một lần trong DynamoDB; trạng thái ban đầu là `SCHEDULED` hoặc trạng thái theo hợp đồng; `createdBy` được lấy từ claim `sub` của Admin; frontend hiển thị phiên mới.                              |
| **Kết quả thực tế**      | Điền HTTP status, ID được tạo, trạng thái và dữ liệu thực tế trong DynamoDB.                                                                                                                                                                                    |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                  |
| **Bằng chứng**           | Request đã che token, response `201`, bản ghi DynamoDB, CloudWatch Logs và ảnh Admin Frontend.                                                                                                                                                                  |


---

#### API-04 — Admin cập nhật phiên đấu giá


| Trường                   | Nội dung                                                                                                                                                                                                                                   |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `API-04`                                                                                                                                                                                                                                   |
| **Tên kiểm thử**         | Cập nhật thông tin phiên đấu giá                                                                                                                                                                                                           |
| **Mục tiêu**             | Xác minh Admin có thể cập nhật phiên trong trạng thái được phép.                                                                                                                                                                           |
| **Điều kiện tiên quyết** | Có phiên ở trạng thái cho phép chỉnh sửa; Admin đã đăng nhập.                                                                                                                                                                              |
| **Các bước thực hiện**   | 1. Ghi nhận dữ liệu phiên trước khi cập nhật. 2. Thay đổi một số trường cho phép, chẳng hạn tên hoặc thời gian. 3. Gửi request cập nhật. 4. Kiểm tra response. 5. Đọc lại bản ghi trong DynamoDB. 6. Tải lại trang chi tiết trên frontend. |
| **Dữ liệu đầu vào**      | Session ID hợp lệ và các trường cập nhật hợp lệ.                                                                                                                                                                                           |
| **Kết quả mong đợi**     | API trả HTTP `200`; chỉ các trường được yêu cầu được cập nhật; các trường không liên quan không bị thay đổi; `updatedAt` được cập nhật nếu hệ thống có trường này; dữ liệu DynamoDB và frontend hiển thị giá trị mới.                      |
| **Kết quả thực tế**      | Điền dữ liệu trước và sau khi cập nhật, HTTP status và kết quả hiển thị.                                                                                                                                                                   |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                             |
| **Bằng chứng**           | Response, bản ghi DynamoDB trước và sau cập nhật, CloudWatch Logs và ảnh frontend.                                                                                                                                                         |


---

#### API-05 — Bắt đầu phiên đấu giá


| Trường                   | Nội dung                                                                                                                                                                                                                                                                 |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `API-05`                                                                                                                                                                                                                                                                 |
| **Tên kiểm thử**         | Chuyển phiên từ `SCHEDULED` sang `ACTIVE`                                                                                                                                                                                                                                |
| **Mục tiêu**             | Xác minh nghiệp vụ bắt đầu phiên chỉ thực hiện khi phiên đáp ứng đầy đủ điều kiện.                                                                                                                                                                                       |
| **Điều kiện tiên quyết** | Phiên đang ở trạng thái `SCHEDULED`; đã có dữ liệu và vật phẩm cần thiết; Admin đã đăng nhập.                                                                                                                                                                            |
| **Các bước thực hiện**   | 1. Kiểm tra trạng thái ban đầu trong DynamoDB. 2. Gửi request bắt đầu phiên. 3. Ghi nhận HTTP status và response. 4. Đọc lại phiên trong DynamoDB. 5. Kiểm tra trạng thái trên User Frontend và Admin Frontend. 6. Thử truy cập chức năng dành cho phiên đang hoạt động. |
| **Dữ liệu đầu vào**      | Session ID của phiên `SCHEDULED`.                                                                                                                                                                                                                                        |
| **Kết quả mong đợi**     | API trả HTTP `200`; trạng thái chuyển thành `ACTIVE`; thời điểm bắt đầu thực tế được ghi nhận nếu có; phiên chỉ được cập nhật một lần; frontend hiển thị phiên đang hoạt động và cho phép các thao tác phù hợp.                                                          |
| **Kết quả thực tế**      | Điền trạng thái trước và sau, thời gian cập nhật và kết quả trên frontend.                                                                                                                                                                                               |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                           |
| **Bằng chứng**           | Response API, bản ghi DynamoDB trước và sau, CloudWatch Logs và ảnh trạng thái `ACTIVE`.                                                                                                                                                                                 |


---

#### API-06 — Kết thúc phiên đấu giá


| Trường                   | Nội dung                                                                                                                                                                                                                                     |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `API-06`                                                                                                                                                                                                                                     |
| **Tên kiểm thử**         | Chuyển phiên từ `ACTIVE` sang `ENDED`                                                                                                                                                                                                        |
| **Mục tiêu**             | Xác minh phiên đang hoạt động có thể được kết thúc và hệ thống ngăn các thao tác không còn hợp lệ.                                                                                                                                           |
| **Điều kiện tiên quyết** | Phiên đang ở trạng thái `ACTIVE`; Admin đã đăng nhập hoặc cơ chế kết thúc tự động đã sẵn sàng.                                                                                                                                               |
| **Các bước thực hiện**   | 1. Kiểm tra trạng thái ban đầu. 2. Gửi request kết thúc phiên. 3. Ghi nhận response. 4. Đọc lại dữ liệu trong DynamoDB. 5. Tải lại trang phiên trên frontend. 6. Thử thực hiện thao tác chỉ được phép khi phiên `ACTIVE`, chẳng hạn đặt giá. |
| **Dữ liệu đầu vào**      | Session ID của phiên `ACTIVE`.                                                                                                                                                                                                               |
| **Kết quả mong đợi**     | API trả HTTP `200`; trạng thái chuyển thành `ENDED`; thời điểm kết thúc được ghi nhận; frontend hiển thị phiên đã kết thúc; hệ thống không tiếp tục chấp nhận thao tác chỉ dành cho phiên `ACTIVE`.                                          |
| **Kết quả thực tế**      | Điền trạng thái, thời gian kết thúc, response và kết quả thử thao tác sau khi kết thúc.                                                                                                                                                      |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                               |
| **Bằng chứng**           | Response, DynamoDB trước và sau, CloudWatch Logs và ảnh frontend.                                                                                                                                                                            |


---

#### API-07 — User không được tạo hoặc sửa phiên đấu giá


| Trường                   | Nội dung                                                                                                                                                                                                             |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `API-07`                                                                                                                                                                                                             |
| **Tên kiểm thử**         | Từ chối User thực hiện nghiệp vụ quản trị phiên                                                                                                                                                                      |
| **Mục tiêu**             | Xác minh User thông thường không thể tạo, cập nhật, bắt đầu hoặc kết thúc phiên đấu giá.                                                                                                                             |
| **Điều kiện tiên quyết** | User đã đăng nhập nhưng không thuộc Cognito Group `Admin`; có endpoint quản trị để kiểm thử.                                                                                                                         |
| **Các bước thực hiện**   | 1. Đăng nhập bằng User. 2. Gửi request tạo phiên. 3. Gửi request cập nhật một phiên hiện có. 4. Nếu phù hợp, thử bắt đầu hoặc kết thúc phiên. 5. Kiểm tra HTTP status. 6. Kiểm tra dữ liệu DynamoDB sau mỗi request. |
| **Dữ liệu đầu vào**      | Token hợp lệ của User và request hợp lệ về mặt định dạng.                                                                                                                                                            |
| **Kết quả mong đợi**     | Các API quản trị trả HTTP `403`; Business Logic Lambda không thực hiện thay đổi trái phép; không có phiên mới; phiên hiện có không bị cập nhật; frontend không hiển thị hoặc không cho phép sử dụng chức năng Admin. |
| **Kết quả thực tế**      | Điền status của từng request và trạng thái dữ liệu sau kiểm thử.                                                                                                                                                     |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                       |
| **Bằng chứng**           | Response `403`, dữ liệu DynamoDB không thay đổi và ảnh giao diện User.                                                                                                                                               |


> Việc frontend ẩn nút quản trị không thay thế cho kiểm tra phân quyền ở backend. Test case phải gửi request trực tiếp đến API bằng token của User.

---

#### API-08 — Dữ liệu đầu vào thiếu hoặc không hợp lệ


| Trường                   | Nội dung                                                                                                                                                                                                                                                                         |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `API-08`                                                                                                                                                                                                                                                                         |
| **Tên kiểm thử**         | Kiểm tra validation của request                                                                                                                                                                                                                                                  |
| **Mục tiêu**             | Xác minh API từ chối request thiếu trường bắt buộc, sai kiểu hoặc vi phạm quy tắc dữ liệu.                                                                                                                                                                                       |
| **Điều kiện tiên quyết** | Admin đã đăng nhập và có quyền gọi API tạo hoặc cập nhật phiên.                                                                                                                                                                                                                  |
| **Các bước thực hiện**   | 1. Gửi request thiếu tên phiên. 2. Gửi request thiếu `startTime` hoặc `endTime`. 3. Gửi request có thời gian sai định dạng. 4. Gửi request có `startTime` lớn hơn hoặc bằng `endTime`. 5. Gửi request có trường sai kiểu dữ liệu. 6. Kiểm tra response và DynamoDB sau từng lần. |
| **Dữ liệu đầu vào**      | Các request thiếu trường hoặc chứa dữ liệu không hợp lệ.                                                                                                                                                                                                                         |
| **Kết quả mong đợi**     | API trả HTTP `400`; response chỉ rõ trường hoặc quy tắc không hợp lệ; không tạo hoặc cập nhật dữ liệu một phần trong DynamoDB; frontend hiển thị thông báo có thể hiểu được.                                                                                                     |
| **Kết quả thực tế**      | Điền từng bộ dữ liệu, HTTP status, error code và trạng thái DynamoDB.                                                                                                                                                                                                            |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                   |
| **Bằng chứng**           | Các request/response lỗi, DynamoDB không thay đổi và ảnh validation trên frontend.                                                                                                                                                                                               |


---

#### API-09 — ID tài nguyên không tồn tại


| Trường                   | Nội dung                                                                                                                                                                                               |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `API-09`                                                                                                                                                                                               |
| **Tên kiểm thử**         | Truy cập hoặc cập nhật tài nguyên không tồn tại                                                                                                                                                        |
| **Mục tiêu**             | Xác minh API xử lý đúng khi Session ID hoặc Item ID không tồn tại.                                                                                                                                     |
| **Điều kiện tiên quyết** | Có ID đúng định dạng nhưng không tồn tại trong DynamoDB.                                                                                                                                               |
| **Các bước thực hiện**   | 1. Gọi API xem chi tiết với ID không tồn tại. 2. Gọi API cập nhật với ID đó. 3. Nếu phù hợp, thử bắt đầu hoặc kết thúc phiên với ID đó. 4. Ghi nhận response. 5. Kiểm tra DynamoDB và CloudWatch Logs. |
| **Dữ liệu đầu vào**      | Session ID hoặc Item ID không tồn tại.                                                                                                                                                                 |
| **Kết quả mong đợi**     | API trả HTTP `404`; response có error code và message nhất quán; không tạo tài nguyên mới ngoài ý muốn; không có dữ liệu nào bị thay đổi; frontend hiển thị thông báo không tìm thấy tài nguyên.       |
| **Kết quả thực tế**      | Điền HTTP status, error code và trạng thái dữ liệu.                                                                                                                                                    |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                         |
| **Bằng chứng**           | Response `404`, kết quả tìm kiếm trong DynamoDB và ảnh frontend.                                                                                                                                       |


---

#### API-10 — Trạng thái phiên không cho phép thao tác


| Trường                   | Nội dung                                                                                                                                                                                                                                                 |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `API-10`                                                                                                                                                                                                                                                 |
| **Tên kiểm thử**         | Từ chối chuyển trạng thái hoặc thao tác không hợp lệ                                                                                                                                                                                                     |
| **Mục tiêu**             | Xác minh Business Logic Lambda thực thi đúng state machine của phiên đấu giá.                                                                                                                                                                            |
| **Điều kiện tiên quyết** | Có phiên ở trạng thái `SCHEDULED`, `ACTIVE` và `ENDED`.                                                                                                                                                                                                  |
| **Các bước thực hiện**   | 1. Thử kết thúc trực tiếp một phiên `SCHEDULED` nếu luồng này không được cho phép. 2. Thử bắt đầu lại phiên `ACTIVE`. 3. Thử bắt đầu hoặc chỉnh sửa phiên `ENDED`. 4. Thử kết thúc lại phiên `ENDED`. 5. Kiểm tra response và dữ liệu sau từng thao tác. |
| **Dữ liệu đầu vào**      | Session ID và thao tác không hợp lệ với trạng thái hiện tại.                                                                                                                                                                                             |
| **Kết quả mong đợi**     | API trả HTTP `409` hoặc `400` theo hợp đồng; response cho biết trạng thái hiện tại không cho phép thao tác; trạng thái và dữ liệu phiên không thay đổi; frontend hiển thị thông báo phù hợp.                                                             |
| **Kết quả thực tế**      | Điền trạng thái ban đầu, thao tác, HTTP status và trạng thái sau request.                                                                                                                                                                                |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                           |
| **Bằng chứng**           | Response lỗi nghiệp vụ, DynamoDB trước và sau, CloudWatch Logs và ảnh frontend.                                                                                                                                                                          |


Các chuyển trạng thái hợp lệ cần tuân theo state machine của dự án, ví dụ:

```text
SCHEDULED → ACTIVE → ENDED
```

Không cho phép:

```text
ENDED → ACTIVE
ACTIVE → SCHEDULED
ENDED → SCHEDULED
```

Nếu dự án hỗ trợ trạng thái `CANCELLED`, các chuyển trạng thái liên quan phải được định nghĩa và kiểm thử riêng theo hợp đồng nghiệp vụ.

---

#### API-11 — Kiểm tra tính nhất quán của dữ liệu trong DynamoDB


| Trường                   | Nội dung                                                                                                                                                                                                                                                                                                |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `API-11`                                                                                                                                                                                                                                                                                                |
| **Tên kiểm thử**         | Xác minh dữ liệu sau nghiệp vụ quản lý phiên                                                                                                                                                                                                                                                            |
| **Mục tiêu**             | Xác minh dữ liệu thực tế trong DynamoDB nhất quán với request, response và quy tắc nghiệp vụ.                                                                                                                                                                                                           |
| **Điều kiện tiên quyết** | Có thể thực hiện ít nhất một nghiệp vụ tạo, cập nhật hoặc chuyển trạng thái phiên.                                                                                                                                                                                                                      |
| **Các bước thực hiện**   | 1. Ghi nhận dữ liệu trước nghiệp vụ. 2. Thực hiện nghiệp vụ qua REST API. 3. Ghi nhận response. 4. Đọc trực tiếp bản ghi tương ứng trong DynamoDB. 5. Đối chiếu ID, trạng thái, thời gian, người tạo và các thuộc tính liên quan. 6. Kiểm tra số lượng bản ghi. 7. Gọi lại API đọc dữ liệu để xác nhận. |
| **Dữ liệu đầu vào**      | Một request tạo, cập nhật, bắt đầu hoặc kết thúc phiên hợp lệ.                                                                                                                                                                                                                                          |
| **Kết quả mong đợi**     | DynamoDB chứa đúng dữ liệu đã được nghiệp vụ chấp nhận; không có bản ghi trùng; khóa chính và khóa sắp xếp đúng; trạng thái đúng; `createdBy` hoặc `updatedBy` lấy từ claim đã xác minh; các thuộc tính không liên quan không bị mất; API đọc lại trả về dữ liệu mới nhất.                              |
| **Kết quả thực tế**      | Điền dữ liệu trước, dữ liệu sau và những trường đã đối chiếu.                                                                                                                                                                                                                                           |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                          |
| **Bằng chứng**           | Request/response và bản ghi DynamoDB trước và sau nghiệp vụ.                                                                                                                                                                                                                                            |


Nếu API trả thành công nhưng DynamoDB không có dữ liệu, lưu sai dữ liệu hoặc frontend vẫn hiển thị dữ liệu cũ, test case phải được đánh dấu `FAIL`.

---

#### API-12 — Kiểm tra hợp đồng API và kết quả trên frontend


| Trường                   | Nội dung                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `API-12`                                                                                                                                                                                                                                                                                                                                                   |
| **Tên kiểm thử**         | Kiểm tra HTTP status, JSON response và tích hợp frontend                                                                                                                                                                                                                                                                                                   |
| **Mục tiêu**             | Xác minh frontend có thể sử dụng response của API ổn định và hiển thị đúng kết quả nghiệp vụ.                                                                                                                                                                                                                                                              |
| **Điều kiện tiên quyết** | User Frontend và Admin Frontend đã tích hợp với REST API.                                                                                                                                                                                                                                                                                                  |
| **Các bước thực hiện**   | 1. Thực hiện một request lấy dữ liệu thành công. 2. Thực hiện một request tạo hoặc cập nhật thành công. 3. Thực hiện request validation thất bại. 4. Thực hiện request đến tài nguyên không tồn tại. 5. Thực hiện request không đủ quyền. 6. Kiểm tra HTTP status, `Content-Type` và JSON của từng response. 7. Kiểm tra cách frontend xử lý từng kết quả. |
| **Dữ liệu đầu vào**      | Các request đại diện cho trường hợp thành công, sai dữ liệu, không tìm thấy và không đủ quyền.                                                                                                                                                                                                                                                             |
| **Kết quả mong đợi**     | API trả đúng status như `200`, `201`, `400`, `403`, `404` hoặc `409`; `Content-Type` là `application/json`; các trường JSON có tên và kiểu dữ liệu nhất quán; frontend hiển thị dữ liệu mới sau thao tác thành công và hiển thị đúng thông báo khi thất bại; không xuất hiện lỗi JavaScript do response sai cấu trúc.                                      |
| **Kết quả thực tế**      | Điền HTTP status, cấu trúc JSON và kết quả hiển thị của từng trường hợp.                                                                                                                                                                                                                                                                                   |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                                                                             |
| **Bằng chứng**           | Network tab, request/response, ảnh frontend và console log đã loại bỏ dữ liệu nhạy cảm.                                                                                                                                                                                                                                                                    |


---

#### Ma trận chuyển trạng thái cần kiểm tra


| Trạng thái hiện tại | Thao tác                            | Trạng thái mong đợi | Kết quả  |
| ------------------- | ----------------------------------- | ------------------- | -------- |
| `SCHEDULED`         | Cập nhật thông tin hợp lệ           | `SCHEDULED`         | Cho phép |
| `SCHEDULED`         | Bắt đầu phiên                       | `ACTIVE`            | Cho phép |
| `ACTIVE`            | Bắt đầu lại                         | Không thay đổi      | Từ chối  |
| `ACTIVE`            | Kết thúc phiên                      | `ENDED`             | Cho phép |
| `ACTIVE`            | Chuyển về `SCHEDULED`               | Không thay đổi      | Từ chối  |
| `ENDED`             | Bắt đầu lại                         | Không thay đổi      | Từ chối  |
| `ENDED`             | Kết thúc lại                        | Không thay đổi      | Từ chối  |
| `ENDED`             | Chỉnh sửa dữ liệu nghiệp vụ bị khóa | Không thay đổi      | Từ chối  |


Ma trận trên cần được điều chỉnh nếu hợp đồng nghiệp vụ chính thức của dự án quy định thêm trạng thái hoặc thao tác khác.

---

#### Quy định kiểm tra DynamoDB

Đối với mỗi API làm thay đổi dữ liệu, nhóm cần kiểm tra:

- Partition key và sort key đúng với thiết kế bảng.
- ID trong response khớp với ID trong DynamoDB.
- Dữ liệu chỉ được tạo một lần.
- Trạng thái được cập nhật đúng.
- `createdAt` không bị thay đổi khi cập nhật.
- `updatedAt` thay đổi khi cập nhật thành công.
- `createdBy` hoặc `updatedBy` được lấy từ claim `sub`.
- Không sử dụng `userId` hoặc `role` từ request body làm dữ liệu đáng tin cậy.
- Không xuất hiện cập nhật một phần khi nghiệp vụ thất bại.
- Không có thuộc tính không liên quan bị xóa.
- Kiểu dữ liệu trong DynamoDB đúng với thiết kế.
- Dữ liệu đọc lại qua API khớp với dữ liệu được lưu.

Nếu hệ thống sử dụng nhiều bảng hoặc nhiều item cho một nghiệp vụ, nhóm phải kiểm tra toàn bộ dữ liệu liên quan, không chỉ item chính.

---

#### Quy định kiểm tra CloudWatch Logs

CloudWatch Logs cần giúp truy vết request nhưng không được chứa dữ liệu nhạy cảm.

Các thông tin nên được ghi nhận gồm:

- Request ID.
- Lambda Function được gọi.
- Tên nghiệp vụ.
- ID tài nguyên.
- User `sub` đã được xác minh.
- Trạng thái trước và sau nghiệp vụ.
- Kết quả thành công hoặc error code.
- Thời gian xử lý.

Không ghi vào log:

- Access Token.
- ID Token.
- Refresh Token.
- Header `Authorization`.
- Mật khẩu.
- AWS credentials.
- Toàn bộ dữ liệu cá nhân không cần thiết.

---

#### Bảng tổng hợp kết quả


| Mã       | Nội dung kiểm thử                  | HTTP status mong đợi | Kiểm tra DynamoDB   | Kiểm tra frontend | Trạng thái  |
| -------- | ---------------------------------- | -------------------- | ------------------- | ----------------- | ----------- |
| `API-01` | Lấy danh sách phiên                | `200`                | Có                  | Có                | Đã kiểm thử |
| `API-02` | Xem chi tiết phiên hoặc vật phẩm   | `200`                | Có                  | Có                | Đã kiểm thử |
| `API-03` | Admin tạo phiên                    | `201`                | Có                  | Có                | Đã kiểm thử |
| `API-04` | Admin cập nhật phiên               | `200`                | Có                  | Có                | Đã kiểm thử |
| `API-05` | Bắt đầu phiên                      | `200`                | Có                  | Có                | Đã kiểm thử |
| `API-06` | Kết thúc phiên                     | `200`                | Có                  | Có                | Đã kiểm thử |
| `API-07` | User tạo hoặc sửa phiên            | `403`                | Phải không thay đổi | Có                | Đã kiểm thử |
| `API-08` | Dữ liệu đầu vào không hợp lệ       | `400`                | Phải không thay đổi | Có                | Đã kiểm thử |
| `API-09` | Tài nguyên không tồn tại           | `404`                | Phải không thay đổi | Có                | Đã kiểm thử |
| `API-10` | Trạng thái không cho phép thao tác | `409` hoặc `400`     | Phải không thay đổi | Có                | Đã kiểm thử |
| `API-11` | Tính nhất quán của DynamoDB        | Theo nghiệp vụ       | Bắt buộc            | Đọc lại dữ liệu   | Đã kiểm thử |
| `API-12` | HTTP status, JSON và frontend      | Theo từng trường hợp | Khi có thay đổi     | Bắt buộc          | Đã kiểm thử |




