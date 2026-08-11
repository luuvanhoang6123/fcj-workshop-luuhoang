---
title: "Kiểm thử bảo mật hệ thống"
date: 2026-08-03
weight: 8
chapter: false
pre: " <b> 5.5.8. </b> "
---


Mục tiêu của kiểm thử bảo mật là xác minh hệ thống thực sự từ chối các hành vi không hợp lệ, thay vì chỉ kiểm tra rằng các cơ chế như JWT, IAM, CORS hoặc S3 Block Public Access đã được cấu hình.

Nội dung kiểm thử tập trung vào:

- Xác thực JWT cho REST API. 
- Phân quyền giữa User và Admin. 
- Phân quyền trên từng tài nguyên. 
- Bảo vệ S3 và CORS. 
- Giới hạn quyền IAM của Lambda. 
- Xác thực kết nối và message WebSocket. 
- Bảo vệ dữ liệu nhạy cảm trong logs. 
- Kiểm soát thông tin được trả về trong lỗi.

Các test case về thay đổi ID tài nguyên cần kiểm tra quyền trên chính đối tượng được truy cập, vì API nhận ID nhưng không kiểm tra quyền sở hữu có thể dẫn đến Broken Object Level Authorization. Kiểm tra chữ ký và thời hạn JWT cũng là yêu cầu quan trọng để phòng tránh Broken Authentication. 

---

#### Phạm vi kiểm thử

Phạm vi gồm:

```
Amazon API Gateway
AWS Lambda
JWT authentication
User/Admin authorization
Auction sessions
Auction items
Bids
Image upload
Amazon S3
CloudFront
CORS
API Gateway WebSocket API
AWS IAM
AWS Secrets Manager
Amazon CloudWatch Logs
AWS CloudTrail
```

Không thực hiện kiểm thử phá hoại trên production. Các test case thử truy cập tài nguyên ngoài IAM Policy phải được thực hiện bằng test function, test role hoặc môi trường development/staging.

---

#### Điều kiện tiên quyết

Trước khi kiểm thử, cần bảo đảm:

- REST API đã được triển khai qua API Gateway và Lambda. 
- Các API được bảo vệ đã sử dụng authorizer hoặc cơ chế xác thực tương đương. 
- JWT có chữ ký và thời gian hết hạn. 
- Hệ thống có ít nhất hai vai trò `USER` và `ADMIN`. 
- Có ít nhất hai tài khoản User khác nhau. 
- Có dữ liệu thuộc quyền sở hữu riêng của từng User. 
- S3 Block Public Access đã được cấu hình. 
- CORS đã có danh sách trusted origins. 
- Lambda execution role đã được tạo. 
- WebSocket `$connect` và các message route đã được triển khai. 
- CloudWatch Logs đã được bật. 
- Môi trường kiểm thử được tách khỏi production. 
- Người kiểm thử có quyền đọc cấu hình và logs cần thiết.

AWS khuyến nghị cấu hình authorizer cho route `$connect` của WebSocket API. Lambda authorizer của WebSocket chỉ được áp dụng tại `$connect`, vì vậy ứng dụng vẫn phải kiểm tra identity, quyền truy cập và định dạng message trong các route tiếp theo.

Nếu thành phần cần thiết chưa được triển khai, test case tương ứng phải được đánh dấu `BLOCKED`.

---

#### Dữ liệu kiểm thử


| Dữ liệuMô tả                |                                                        |
| --------------------------- | ------------------------------------------------------ |
| Protected API               | API yêu cầu đăng nhập, ví dụ `GET /users/me`           |
| Admin API                   | API chỉ dành cho Admin                                 |
| Valid User Token            | JWT còn hiệu lực của User                              |
| Valid Admin Token           | JWT còn hiệu lực của Admin                             |
| Forged Token                | JWT đã bị sửa payload hoặc ký bằng secret khác         |
| Expired Token               | JWT có claim `exp` đã hết hạn                          |
| Unsupported Algorithm Token | Token dùng thuật toán không được hệ thống cho phép     |
| User A                      | User sở hữu tài nguyên A                               |
| User B                      | User sở hữu tài nguyên B                               |
| Resource A                  | Session, item, ảnh hoặc dữ liệu thuộc User A           |
| Resource B                  | Session, item, ảnh hoặc dữ liệu thuộc User B           |
| Trusted Origin              | Origin frontend nằm trong CORS allowlist               |
| Untrusted Origin            | Origin không nằm trong CORS allowlist                  |
| Private S3 Object URL       | URL trực tiếp đến object trong private bucket          |
| Allowed AWS Resource        | Bucket, secret hoặc prefix Lambda được phép truy cập   |
| Restricted AWS Resource     | Resource nằm ngoài IAM Policy của Lambda               |
| Valid WebSocket Message     | JSON đúng action, field và kiểu dữ liệu                |
| Invalid WebSocket Message   | Message sai JSON, thiếu field hoặc action không hợp lệ |
| Oversized WebSocket Message | Message vượt giới hạn của ứng dụng                     |
| Correlation ID              | ID dùng để tìm request tương ứng trong logs            |


Không sử dụng token, tài khoản hoặc dữ liệu production.

---

### SEC-01 — API từ chối request không có token


| TrườngNội dung           |                                                                                                                                                                                                                                                       |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-01`                                                                                                                                                                                                                                              |
| **Tên kiểm thử**         | Từ chối request không có access token                                                                                                                                                                                                                 |
| **Mục tiêu**             | Xác minh API được bảo vệ không cho phép anonymous user truy cập.                                                                                                                                                                                      |
| **Điều kiện tiên quyết** | Có ít nhất một protected API đang hoạt động.                                                                                                                                                                                                          |
| **Các bước thực hiện**   | 1. Gửi request đến protected API mà không có header `Authorization`. 2. Gửi request với header `Authorization` rỗng. 3. Gửi `Authorization: Basic abc`. 4. Gửi `Authorization: Bearer` nhưng không có token. 5. Kiểm tra response và CloudWatch Logs. |
| **Kết quả mong đợi**     | Tất cả request bị từ chối với `401 Unauthorized`; Lambda nghiệp vụ không thực hiện thay đổi dữ liệu; response không trả thông tin của User; lỗi có mã ổn định như `AUTH_TOKEN_REQUIRED`.                                                              |
| **Kết quả thực tế**      | Điền endpoint, trường hợp kiểm thử, status code và error code.                                                                                                                                                                                        |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                        |
| **Bằng chứng**           | Request/response đã che dữ liệu nhạy cảm và CloudWatch Logs.                                                                                                                                                                                          |


---

### SEC-02 — Token giả mạo bị từ chối


| TrườngNội dung           |                                                                                                                                                                                                                                                                                                  |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `SEC-02`                                                                                                                                                                                                                                                                                         |
| **Tên kiểm thử**         | Từ chối JWT có chữ ký hoặc thuật toán không hợp lệ                                                                                                                                                                                                                                               |
| **Mục tiêu**             | Xác minh hệ thống kiểm tra chữ ký và chỉ chấp nhận thuật toán JWT được cấu hình.                                                                                                                                                                                                                 |
| **Điều kiện tiên quyết** | Có Valid User Token và công cụ tạo token kiểm thử.                                                                                                                                                                                                                                               |
| **Các bước thực hiện**   | 1. Giải mã Valid User Token trong môi trường kiểm thử. 2. Thay claim `sub` hoặc `role` nhưng giữ nguyên chữ ký. 3. Gửi token đã sửa đến protected API. 4. Tạo token ký bằng secret khác và gửi lại. 5. Thử token dùng `alg: none` hoặc thuật toán không được phép. 6. Kiểm tra response và logs. |
| **Kết quả mong đợi**     | Tất cả token giả mạo bị từ chối với `401 Unauthorized`; hệ thống không tin payload trước khi xác minh chữ ký; chỉ thuật toán đã cấu hình, ví dụ `HS256`, được chấp nhận; không có dữ liệu bị thay đổi.                                                                                           |
| **Kết quả thực tế**      | Điền loại token, algorithm, status và error code; không ghi toàn bộ token.                                                                                                                                                                                                                       |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                   |
| **Bằng chứng**           | Request/response đã che token và log xác minh JWT.                                                                                                                                                                                                                                               |


> Không đưa access token hoàn chỉnh vào tài liệu hoặc ảnh bằng chứng.

---

### SEC-03 — Token hết hạn bị từ chối


| TrườngNội dung           |                                                                                                                                                                                                               |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-03`                                                                                                                                                                                                      |
| **Tên kiểm thử**         | Từ chối JWT đã hết hạn                                                                                                                                                                                        |
| **Mục tiêu**             | Xác minh claim `exp` được kiểm tra tại thời điểm request.                                                                                                                                                     |
| **Điều kiện tiên quyết** | Có Expired Token hoặc có thể tạo token với thời hạn rất ngắn trong môi trường kiểm thử.                                                                                                                       |
| **Các bước thực hiện**   | 1. Gửi token khi còn hiệu lực nếu sử dụng token thời hạn ngắn. 2. Chờ token hết hạn. 3. Gửi lại cùng request. 4. Thử token không có claim `exp` nếu `exp` là claim bắt buộc. 5. Kiểm tra response và dữ liệu. |
| **Kết quả mong đợi**     | Token còn hạn được xử lý theo quyền của User; token hết hạn hoặc thiếu claim bắt buộc bị từ chối với `401 Unauthorized`; không có thao tác nghiệp vụ được thực hiện.                                          |
| **Kết quả thực tế**      | Điền thời điểm hết hạn đã làm mờ, thời điểm kiểm thử, status và error code.                                                                                                                                   |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                |
| **Bằng chứng**           | Response và CloudWatch Logs không chứa token đầy đủ.                                                                                                                                                          |


---

### SEC-04 — User không thể truy cập API Admin


| TrườngNội dung           |                                                                                                                                                                                                                   |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-04`                                                                                                                                                                                                          |
| **Tên kiểm thử**         | Kiểm tra phân quyền Admin API                                                                                                                                                                                     |
| **Mục tiêu**             | Xác minh User đã đăng nhập nhưng không có vai trò Admin vẫn bị từ chối.                                                                                                                                           |
| **Điều kiện tiên quyết** | Có Valid User Token, Valid Admin Token và Admin API.                                                                                                                                                              |
| **Các bước thực hiện**   | 1. Gọi Admin API bằng Valid User Token. 2. Thử các method `GET`, `POST`, `PUT`, `PATCH` hoặc `DELETE` có trong Admin API. 3. Gọi cùng chức năng bằng Valid Admin Token. 4. Kiểm tra dữ liệu trước và sau request. |
| **Kết quả mong đợi**     | User thường nhận `403 Forbidden`; Admin hợp lệ được xử lý theo nghiệp vụ; User không đọc hoặc thay đổi dữ liệu Admin; kiểm tra quyền được thực hiện phía server.                                                  |
| **Kết quả thực tế**      | Điền endpoint, method, caller role, status và thay đổi dữ liệu.                                                                                                                                                   |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                    |
| **Bằng chứng**           | API responses, dữ liệu trước/sau và authorization logs.                                                                                                                                                           |


---

### SEC-05 — Không thể thay đổi quyền bằng trường `role`


| TrườngNội dung           |                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-05`                                                                                                                                                                                                                                                                                                                  |
| **Tên kiểm thử**         | Ngăn mass assignment và role escalation                                                                                                                                                                                                                                                                                   |
| **Mục tiêu**             | Xác minh User không thể tự nâng quyền bằng dữ liệu trong request.                                                                                                                                                                                                                                                         |
| **Điều kiện tiên quyết** | Có API đăng ký, cập nhật profile hoặc cập nhật User.                                                                                                                                                                                                                                                                      |
| **Các bước thực hiện**   | 1. Gửi request hợp lệ bằng User thường. 2. Thêm `"role": "ADMIN"` vào request body. 3. Thử thêm `"is_admin": true`, `"status": "ACTIVE"` hoặc trường quyền tương tự. 4. Thử gửi role qua query parameter hoặc header tùy chỉnh. 5. Đăng nhập lại và kiểm tra quyền trong database hoặc API profile. 6. Thử gọi Admin API. |
| **Kết quả mong đợi**     | Server từ chối trường không được phép với `400/422`, hoặc bỏ qua trường đó theo contract; role trong database không thay đổi; JWT mới không chứa quyền Admin; User vẫn nhận `403` khi gọi Admin API.                                                                                                                      |
| **Kết quả thực tế**      | Điền payload đã che, response, role trước/sau và kết quả Admin API.                                                                                                                                                                                                                                                       |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                                            |
| **Bằng chứng**           | Request body, API response và bản ghi User trước/sau.                                                                                                                                                                                                                                                                     |


Identity và role phải đến từ JWT đã xác minh và dữ liệu phía server, không lấy từ request body.

---

### SEC-06 — Không thể thao tác tài nguyên của User khác bằng cách đổi ID


| TrườngNội dung           |                                                                                                                                                                                                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-06`                                                                                                                                                                                                                                                                                                |
| **Tên kiểm thử**         | Kiểm tra Object-Level Authorization                                                                                                                                                                                                                                                                     |
| **Mục tiêu**             | Xác minh User A không thể đọc, sửa hoặc xóa tài nguyên riêng của User B.                                                                                                                                                                                                                                |
| **Điều kiện tiên quyết** | User A và User B có tài nguyên riêng biệt.                                                                                                                                                                                                                                                              |
| **Các bước thực hiện**   | 1. Đăng nhập bằng User A. 2. Thực hiện thao tác hợp lệ trên Resource A. 3. Thay resource ID bằng ID của Resource B. 4. Thử `GET`, `PUT/PATCH` và `DELETE` nếu các method tồn tại. 5. Thử thay `userId`, `ownerId`, `itemId`, `sessionId` trong path, query và body. 6. Kiểm tra Resource B sau request. |
| **Kết quả mong đợi**     | Request trên Resource A được xử lý theo nghiệp vụ; truy cập Resource B bị từ chối với `403 Forbidden` hoặc `404 Not Found` theo contract; Resource B không bị đọc, thay đổi hoặc xóa; UUID khó đoán không được xem là cơ chế phân quyền.                                                                |
| **Kết quả thực tế**      | Điền loại tài nguyên, caller, owner, operation, status và trạng thái dữ liệu.                                                                                                                                                                                                                           |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                          |
| **Bằng chứng**           | Responses và dữ liệu trước/sau của hai tài nguyên.                                                                                                                                                                                                                                                      |


---

### SEC-07 — S3 bucket không cho phép truy cập công khai


| TrườngNội dung           |                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `SEC-07`                                                                                                                                                                                                                                                                                                                                               |
| **Tên kiểm thử**         | Chặn truy cập công khai vào S3                                                                                                                                                                                                                                                                                                                         |
| **Mục tiêu**             | Xác minh Internet user không thể đọc hoặc liệt kê bucket trực tiếp.                                                                                                                                                                                                                                                                                    |
| **Điều kiện tiên quyết** | Bucket và ít nhất một object kiểm thử đã tồn tại.                                                                                                                                                                                                                                                                                                      |
| **Các bước thực hiện**   | 1. Mở S3 object URL trực tiếp trong trình duyệt ẩn danh. 2. Gửi request không có chữ ký đến object. 3. Thử truy cập bucket root hoặc thực hiện `ListBucket` không xác thực. 4. Kiểm tra bốn thiết lập Block Public Access. 5. Kiểm tra bucket policy và object ACL. 6. Truy cập object qua CloudFront hoặc presigned URL hợp lệ nếu thiết kế cho phép. |
| **Kết quả mong đợi**     | Truy cập S3 trực tiếp bị từ chối với `403 AccessDenied`; không thể liệt kê bucket; Block Public Access được bật; policy/ACL không cấp quyền cho public principal; CloudFront OAC hoặc presigned URL hợp lệ vẫn hoạt động theo thiết kế.                                                                                                                |
| **Kết quả thực tế**      | Điền bucket đã che, loại request, status và public access settings.                                                                                                                                                                                                                                                                                    |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                                                                         |
| **Bằng chứng**           | Response `403`, Block Public Access, bucket policy và ACL.                                                                                                                                                                                                                                                                                             |


AWS khuyến nghị bật toàn bộ bốn thiết lập S3 Block Public Access và kiểm tra thêm bucket policy cùng identity policy liên quan.

---

### SEC-08 — CORS chỉ cho phép origin được tin cậy


| TrườngNội dung           |                                                                                                                                                                                                                                                                                                                    |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `SEC-08`                                                                                                                                                                                                                                                                                                           |
| **Tên kiểm thử**         | Kiểm tra CORS allowlist                                                                                                                                                                                                                                                                                            |
| **Mục tiêu**             | Xác minh trình duyệt chỉ cho phép frontend origin đã cấu hình truy cập API hoặc S3.                                                                                                                                                                                                                                |
| **Điều kiện tiên quyết** | Trusted Origin, Untrusted Origin và CORS configuration đã tồn tại.                                                                                                                                                                                                                                                 |
| **Các bước thực hiện**   | 1. Gửi preflight `OPTIONS` từ Trusted Origin. 2. Kiểm tra allow-origin, allow-methods, allow-headers và credentials. 3. Gửi request thật từ Trusted Origin. 4. Lặp lại với Untrusted Origin. 5. Thử origin gần giống như subdomain giả hoặc domain có hậu tố bổ sung. 6. Kiểm tra response trong trình duyệt thật. |
| **Kết quả mong đợi**     | Trusted Origin nhận đúng CORS headers; Untrusted Origin không nhận `Access-Control-Allow-Origin`; không phản chiếu tùy ý giá trị `Origin`; không dùng wildcard với credentialed requests; chỉ method và header cần thiết được cho phép.                                                                            |
| **Kết quả thực tế**      | Điền origin, requested method, CORS headers và kết quả trình duyệt.                                                                                                                                                                                                                                                |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                                     |
| **Bằng chứng**           | Preflight request/response và browser console.                                                                                                                                                                                                                                                                     |


> CORS không phải cơ chế authentication hoặc authorization. Request ngoài trình duyệt vẫn phải bị kiểm soát bằng JWT, presigned URL, IAM hoặc chính sách tài nguyên.

AWS mô tả CORS là cơ chế để trình duyệt cho phép các ứng dụng từ một domain tương tác với tài nguyên ở domain khác; nó không thay thế kiểm soát truy cập. 

---

### SEC-09 — Lambda bị từ chối khi truy cập ngoài IAM Policy


| TrườngNội dung           |                                                                                                                                                                                                                                                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-09`                                                                                                                                                                                                                                                                                                                                    |
| **Tên kiểm thử**         | Kiểm tra least-privilege IAM của Lambda                                                                                                                                                                                                                                                                                                     |
| **Mục tiêu**             | Xác minh Lambda chỉ truy cập resource và action đã được cấp quyền.                                                                                                                                                                                                                                                                          |
| **Điều kiện tiên quyết** | Có test Lambda role, Allowed AWS Resource và Restricted AWS Resource.                                                                                                                                                                                                                                                                       |
| **Các bước thực hiện**   | 1. Cho test Lambda thực hiện action được phép trên Allowed Resource. 2. Xác nhận thao tác thành công. 3. Cho Lambda thực hiện cùng action trên Restricted Resource. 4. Thử action không được cấp trên Allowed Resource. 5. Kiểm tra execution role, permission boundary, resource policy và CloudTrail. 6. Kiểm tra dữ liệu sau thử nghiệm. |
| **Kết quả mong đợi**     | Action được cấp trên đúng resource thành công; action hoặc resource ngoài policy bị từ chối với `AccessDenied`; không dùng quyền rộng như `s3:`*, `secretsmanager:`* hoặc resource `*` khi không cần; Lambda xử lý lỗi an toàn.                                                                                                             |
| **Kết quả thực tế**      | Điền role, action, resource đã che, decision và AWS error code.                                                                                                                                                                                                                                                                             |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                                                              |
| **Bằng chứng**           | IAM policy, Lambda logs, CloudTrail event và trạng thái resource.                                                                                                                                                                                                                                                                           |


> Không sửa Lambda production để tạo hành vi truy cập trái phép. Sử dụng test function hoặc test role có kiểm soát.

---

### SEC-10 — WebSocket message được xác thực và kiểm tra định dạng


| TrườngNội dung           |                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-10`                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Tên kiểm thử**         | Kiểm tra bảo mật kết nối và message WebSocket                                                                                                                                                                                                                                                                                                                                                            |
| **Mục tiêu**             | Xác minh chỉ client hợp lệ được kết nối và message không hợp lệ không được xử lý.                                                                                                                                                                                                                                                                                                                        |
| **Điều kiện tiên quyết** | WebSocket API, `$connect` authorization và message handlers đã triển khai.                                                                                                                                                                                                                                                                                                                               |
| **Các bước thực hiện**   | 1. Kết nối WebSocket không có credential. 2. Kết nối bằng token giả hoặc hết hạn. 3. Kết nối bằng Valid User Token. 4. Gửi JSON hợp lệ. 5. Gửi chuỗi không phải JSON. 6. Gửi JSON thiếu `action` hoặc trường bắt buộc. 7. Gửi action không được hỗ trợ. 8. Gửi sai kiểu dữ liệu, field dư hoặc message quá lớn. 9. Thử gửi `userId`, `role` hoặc `itemId` không được phép. 10. Kiểm tra dữ liệu và logs. |
| **Kết quả mong đợi**     | Kết nối không xác thực bị từ chối; kết nối hợp lệ thành công; identity được gắn từ kết quả xác thực; message sai trả lỗi có kiểm soát hoặc bị đóng kết nối theo contract; không có thay đổi dữ liệu; client không thể giả danh User khác bằng field trong message.                                                                                                                                       |
| **Kết quả thực tế**      | Điền loại kết nối/message, response event, close code và kết quả xử lý.                                                                                                                                                                                                                                                                                                                                  |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                                                                                                                           |
| **Bằng chứng**           | WebSocket client output, API Gateway access logs và Lambda logs.                                                                                                                                                                                                                                                                                                                                         |


Các route gửi dữ liệu qua API Gateway Management API cần quyền `execute-api:ManageConnections` đúng phạm vi. 

---

### SEC-11 — Logs không chứa dữ liệu nhạy cảm


| TrườngNội dung           |                                                                                                                                                                                                                                                                                                                                     |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-11`                                                                                                                                                                                                                                                                                                                            |
| **Tên kiểm thử**         | Kiểm tra rò rỉ secret trong logs                                                                                                                                                                                                                                                                                                    |
| **Mục tiêu**             | Xác minh logs không chứa password, token, secret key hoặc presigned signature.                                                                                                                                                                                                                                                      |
| **Điều kiện tiên quyết** | CloudWatch Logs đã bật và người kiểm thử có quyền đọc logs.                                                                                                                                                                                                                                                                         |
| **Các bước thực hiện**   | 1. Thực hiện đăng ký, đăng nhập và refresh token. 2. Gọi protected API. 3. Tạo presigned upload nếu có. 4. Kích hoạt lỗi xác thực, S3 và database có kiểm soát. 5. Tìm trong logs theo các chuỗi nhạy cảm. 6. Kiểm tra API Gateway access logs, Lambda logs và application logs. 7. Kiểm tra retention và quyền truy cập log group. |
| **Kết quả mong đợi**     | Không có password, access token, refresh token, cookie, AWS access key, secret key, database password, JWT secret hoặc toàn bộ presigned URL; chỉ identifier cần thiết được ghi; dữ liệu nhạy cảm được che; quyền đọc logs được giới hạn.                                                                                           |
| **Kết quả thực tế**      | Điền log group, time range, từ khóa kiểm tra và số kết quả; không sao chép secret vào báo cáo.                                                                                                                                                                                                                                      |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                                                      |
| **Bằng chứng**           | Truy vấn log đã làm mờ và cấu hình log group.                                                                                                                                                                                                                                                                                       |


Các chuỗi cần tìm tối thiểu:

```
password
passwd
Authorization
Bearer
access_token
refresh_token
id_token
secret
secret_key
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
X-Amz-Signature
X-Amz-Credential
Cookie
Set-Cookie
```

Không ghi chuỗi secret thật vào câu truy vấn hoặc tài liệu kiểm thử.

---

### SEC-12 — Thông báo lỗi không tiết lộ cấu trúc hệ thống


| Trường                   | Nội dung                                                                                                                                                                                                                                                                               |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `SEC-12`                                                                                                                                                                                                                                                                               |
| **Tên kiểm thử**         | Kiểm tra information disclosure trong error response                                                                                                                                                                                                                                   |
| **Mục tiêu**             | Xác minh lỗi phía client không làm lộ thông tin nội bộ.                                                                                                                                                                                                                                |
| **Điều kiện tiên quyết** | Có thể tạo lỗi validation, authentication, authorization, not found và internal error trong môi trường kiểm thử.                                                                                                                                                                       |
| **Các bước thực hiện**   | 1. Gửi JSON sai định dạng. 2. Gửi thiếu field bắt buộc. 3. Gửi ID không tồn tại. 4. Gửi token không hợp lệ. 5. Kích hoạt database/S3 error có kiểm soát. 6. Gọi route không tồn tại. 7. Kiểm tra response body, headers và logs.                                                       |
| **Kết quả mong đợi**     | Client chỉ nhận status, error code, thông báo an toàn và correlation ID; response không chứa stack trace, đường dẫn file, tên bảng, SQL, database host, bucket nội bộ, Lambda ARN, secret name, source code hoặc dependency version; chi tiết kỹ thuật chỉ nằm trong logs được bảo vệ. |
| **Kết quả thực tế**      | Điền loại lỗi, status, error code và trường dữ liệu được trả về.                                                                                                                                                                                                                       |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                         |
| **Bằng chứng**           | Error responses đã làm mờ và logs tương ứng theo correlation ID.                                                                                                                                                                                                                       |


Response lỗi an toàn có dạng:

```
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred.",
    "requestId": "req-123456"
  }
}
```

Không được trả về:

```
{
  "error": "OperationalError",
  "sql": "SELECT * FROM users...",
  "databaseHost": "auction-db.xxxx.ap-southeast-1.rds.amazonaws.com",
  "file": "/var/task/app/infrastructure/database.py",
  "stackTrace": "..."
}
```

---

### Ma trận kiểm thử bảo mật


| Mã       | Nội dung kiểm thử           | Thành phần chính         | Kết quả mong đợi              |
| -------- | --------------------------- | ------------------------ | ----------------------------- |
| `SEC-01` | Request không có token      | API Gateway/Lambda       | `401 Unauthorized`            |
| `SEC-02` | Token giả mạo               | JWT verifier             | `401 Unauthorized`            |
| `SEC-03` | Token hết hạn               | JWT verifier             | `401 Unauthorized`            |
| `SEC-04` | User gọi Admin API          | Authorization guard      | `403 Forbidden`               |
| `SEC-05` | Thay đổi role trong request | Validation/authorization | Role không thay đổi           |
| `SEC-06` | Thay ID tài nguyên          | Object authorization     | `403` hoặc `404`              |
| `SEC-07` | Truy cập S3 công khai       | S3 policy/BPA            | `403 AccessDenied`            |
| `SEC-08` | Origin không tin cậy        | CORS                     | Không có CORS permission      |
| `SEC-09` | Lambda vượt IAM Policy      | IAM                      | `AccessDenied`                |
| `SEC-10` | WebSocket không hợp lệ      | WebSocket API/Lambda     | Kết nối/message bị từ chối    |
| `SEC-11` | Dữ liệu nhạy cảm trong logs | CloudWatch/API Gateway   | Không tìm thấy secret         |
| `SEC-12` | Thông tin nội bộ trong lỗi  | Error handler            | Không tiết lộ cấu trúc nội bộ |


---

### Bảng tổng hợp kết quả


| Mã       | Nội dung kiểm thử                        | Kết quả thực tế | Trạng thái    |
| -------- | ---------------------------------------- | --------------- | ------------- |
| `SEC-01` | API từ chối request không có token       | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-02` | Token giả mạo bị từ chối                 | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-03` | Token hết hạn bị từ chối                 | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-04` | User không truy cập được Admin API       | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-05` | Không thể thay đổi role từ request       | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-06` | Không thao tác được tài nguyên User khác | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-07` | S3 không cho phép public access          | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-08` | CORS chỉ cho phép trusted origin         | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-09` | Lambda bị giới hạn bởi IAM Policy        | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-10` | WebSocket xác thực và validate message   | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-11` | Logs không chứa dữ liệu nhạy cảm         | Chưa kiểm thử   | Chưa kiểm thử |
| `SEC-12` | Lỗi không tiết lộ thông tin nội bộ       | Chưa kiểm thử   | Chưa kiểm thử |


---

### Quy định mã trạng thái HTTP


| Tình huống                     | Status mong đợi                                     |
| ------------------------------ | --------------------------------------------------- |
| Thiếu access token             | `401 Unauthorized`                                  |
| Token sai chữ ký               | `401 Unauthorized`                                  |
| Token hết hạn                  | `401 Unauthorized`                                  |
| Token thiếu claim bắt buộc     | `401 Unauthorized`                                  |
| Đã đăng nhập nhưng thiếu quyền | `403 Forbidden`                                     |
| Không sở hữu tài nguyên        | `403 Forbidden` hoặc `404 Not Found` theo contract  |
| Request body sai định dạng     | `400 Bad Request` hoặc `422 Unprocessable Entity`   |
| Tài nguyên không tồn tại       | `404 Not Found`                                     |
| Request quá lớn                | `413 Payload Too Large`                             |
| Message/action không hỗ trợ    | Error event hoặc close code theo WebSocket contract |
| Lỗi nội bộ                     | `500 Internal Server Error` với thông báo an toàn   |


Không dùng `500 Internal Server Error` cho mọi lỗi xác thực hoặc validation.

---

### Yêu cầu kiểm tra JWT

JWT verifier phải xác minh tối thiểu:

```
signature
allowed algorithm
expiration
required claims
subject/user ID
issuer, nếu hệ thống sử dụng
audience, nếu hệ thống sử dụng
token type, nếu phân biệt access và refresh token
```

Không được tin các giá trị sau từ request body:

```
userId
ownerId
email
role
isAdmin
status
```

Access token và refresh token không được dùng thay thế cho nhau.

---

### Yêu cầu kiểm tra phân quyền

Mỗi API truy cập tài nguyên bằng ID phải kiểm tra:

- User đã xác thực. 
- User còn ở trạng thái `ACTIVE`. 
- User có vai trò phù hợp. 
- User có quyền thực hiện action. 
- User sở hữu tài nguyên hoặc được cấp quyền trên tài nguyên. 
- Tài nguyên thuộc đúng session/item liên quan. 
- Quyền được kiểm tra lại tại server cho mỗi request. 
- Không dựa vào việc frontend ẩn nút chức năng. 
- Không dựa vào UUID khó đoán để thay thế authorization.

---

### Yêu cầu kiểm tra WebSocket

WebSocket phải kiểm tra:

- Xác thực tại `$connect`. 
- Token không hợp lệ hoặc hết hạn bị từ chối. 
- Connection được gắn với identity đã xác minh. 
- User không được tự khai báo `userId` hoặc `role`. 
- User chỉ được join room được phép. 
- Mỗi message phải là JSON đúng định dạng. 
- `action` phải nằm trong allowlist. 
- Field bắt buộc phải tồn tại và đúng kiểu. 
- Giới hạn kích thước message. 
- Không thực thi nội dung message như code hoặc câu lệnh. 
- Rate limit hoặc throttling được cấu hình nếu cần. 
- Connection ID không được dùng như bằng chứng duy nhất về identity. 
- Message lỗi không làm Lambda crash hoặc trả stack trace.

---

### Yêu cầu kiểm tra logs

Logs chứa:

```
requestId
correlationId
verified userId
action
resourceId
route
HTTP method
status code
authorization decision
AWS error code
Lambda request ID
processing duration
```

Logs không được chứa:

```
password
password hash
access token
refresh token
ID token
session cookie
JWT secret
AWS access key
AWS secret key
database password
Secrets Manager secret value
presigned URL đầy đủ
X-Amz-Signature
Authorization header đầy đủ
ảnh dạng Base64
```

Nếu cần nhận diện token để điều tra, chỉ ghi fingerprint/hash không thể đảo ngược hoặc một phần đã che, không ghi token hoàn chỉnh.