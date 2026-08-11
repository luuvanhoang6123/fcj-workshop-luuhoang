---
title: "Kiểm thử xác thực và phân quyền"
date: 2026-08-03
weight: 3
chapter: false
pre: " <b> 5.5.3. </b> "
---

### Kiểm thử xác thực và phân quyền

#### Mục tiêu kiểm thử

Phần này kiểm tra khả năng xác thực và phân quyền của hệ thống Live Auction ở cấp độ hệ thống, bao gồm:

- Đăng ký, xác nhận và đăng nhập người dùng bằng Amazon Cognito.
- Đồng bộ thông tin người dùng từ Cognito sang Amazon DynamoDB.
- Hoạt động của Lambda Post Confirmation Trigger.
- Xác thực JWT bằng API Gateway Authorizer.
- Phân quyền giữa tài khoản `User` và `Admin`.
- Sử dụng các claim đã được Cognito xác minh như `sub` và `cognito:groups`.
- Ngăn người dùng giả mạo danh tính hoặc quyền thông qua request body.
- Xác minh Lambda chỉ truy cập tài nguyên AWS trong phạm vi IAM Policy được cấp.

Các test case được đánh mã từ `AUTH-01` đến `AUTH-13`. Yêu cầu kiểm tra Post Confirmation và xử lý lặp được gộp trong `AUTH-03`; việc sử dụng claim đã xác minh được kiểm tra trong `AUTH-10`; hành vi không tin tưởng `userId` và `role` từ request body được kiểm tra trong `AUTH-13`.

#### Điều kiện kiểm thử chung

Trước khi thực hiện các test case, hệ thống cần đáp ứng các điều kiện sau:

- Amazon Cognito User Pool đã được triển khai.
- User Pool App Client đã được cấu hình cho frontend.
- Lambda Post Confirmation đã được liên kết với Cognito User Pool.
- REST API đã được bảo vệ bằng API Gateway Authorizer.
- Các Lambda Function đã được gán IAM Execution Role phù hợp.
- DynamoDB có bảng lưu thông tin người dùng.
- Có User Frontend và Admin Frontend hoạt động.
- Có ít nhất một API thông thường dành cho User.
- Có ít nhất một API quản trị chỉ dành cho Admin.
- CloudWatch Logs đã được bật cho các Lambda Function liên quan.
- Có tài khoản User và Admin riêng phục vụ kiểm thử.

#### Dữ liệu kiểm thử


| Dữ liệu            | Mô tả                                                            |
| ------------------ | ---------------------------------------------------------------- |
| User mới           | Email chưa tồn tại trong Cognito User Pool                       |
| User đã xác nhận   | Tài khoản có trạng thái `CONFIRMED`                              |
| User chưa xác nhận | Tài khoản chưa hoàn thành bước xác nhận                          |
| Admin              | Tài khoản thuộc Cognito Group `Admin`                            |
| Mật khẩu đúng      | Mật khẩu hợp lệ của tài khoản kiểm thử                           |
| Mật khẩu sai       | Mật khẩu không khớp với tài khoản                                |
| Token hợp lệ       | JWT còn hạn do đúng Cognito User Pool phát hành                  |
| Token không hợp lệ | Token bị thay đổi nội dung, sai chữ ký hoặc không đúng định dạng |
| Token hết hạn      | JWT có thời gian `exp` nhỏ hơn thời gian hiện tại                |
| User ID giả mạo    | ID của một người dùng khác được gửi trong request body           |
| Role giả mạo       | Giá trị `Admin` được User gửi trong request body                 |


Không đưa email thật, mật khẩu, Access Token, ID Token hoặc Refresh Token vào tài liệu kiểm thử.

---

#### AUTH-01 — Đăng ký tài khoản User thành công


| Trường                   | Nội dung                                                                                                                                                              |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-01`                                                                                                                                                             |
| **Tên kiểm thử**         | Đăng ký tài khoản User thành công                                                                                                                                     |
| **Mục tiêu**             | Xác minh người dùng mới có thể đăng ký tài khoản thông qua User Frontend và Amazon Cognito.                                                                           |
| **Điều kiện tiên quyết** | Email kiểm thử chưa tồn tại trong Cognito User Pool; User Frontend có thể kết nối với Cognito.                                                                        |
| **Các bước thực hiện**   | 1. Mở trang đăng ký. 2. Nhập email và mật khẩu hợp lệ. 3. Nhập các thông tin bắt buộc. 4. Nhấn nút đăng ký. 5. Kiểm tra trạng thái tài khoản trong Cognito User Pool. |
| **Dữ liệu đầu vào**      | Email kiểm thử mới, mật khẩu đáp ứng Password Policy và thông tin hồ sơ hợp lệ.                                                                                       |
| **Kết quả mong đợi**     | Yêu cầu đăng ký thành công; tài khoản được tạo trong Cognito; giao diện yêu cầu người dùng nhập mã xác nhận; tài khoản chưa được phép đăng nhập trước khi xác nhận.   |
| **Kết quả thực tế**      | Điền sau khi thực hiện kiểm thử.                                                                                                                                      |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                        |
| **Bằng chứng**           | Ảnh giao diện đăng ký và tài khoản trong Cognito User Pool đã che dữ liệu nhạy cảm.                                                                                   |


---

#### AUTH-02 — Xác nhận tài khoản thành công


| Trường                   | Nội dung                                                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `AUTH-02`                                                                                                                                                    |
| **Tên kiểm thử**         | Xác nhận tài khoản bằng mã hợp lệ                                                                                                                            |
| **Mục tiêu**             | Xác minh tài khoản mới có thể được xác nhận bằng mã xác nhận hợp lệ.                                                                                         |
| **Điều kiện tiên quyết** | Tài khoản đã được đăng ký nhưng chưa xác nhận; người dùng đã nhận được mã xác nhận.                                                                          |
| **Các bước thực hiện**   | 1. Mở trang xác nhận tài khoản. 2. Nhập email kiểm thử. 3. Nhập mã xác nhận hợp lệ. 4. Gửi yêu cầu xác nhận. 5. Kiểm tra trạng thái trong Cognito User Pool. |
| **Dữ liệu đầu vào**      | Email kiểm thử và mã xác nhận còn hiệu lực.                                                                                                                  |
| **Kết quả mong đợi**     | Cognito xác nhận tài khoản thành công; trạng thái tài khoản chuyển thành `CONFIRMED`; người dùng được phép chuyển sang bước đăng nhập.                       |
| **Kết quả thực tế**      | Điền sau khi thực hiện kiểm thử.                                                                                                                             |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                               |
| **Bằng chứng**           | Ảnh giao diện xác nhận và trạng thái `CONFIRMED` trong Cognito User Pool.                                                                                    |


---

#### AUTH-03 — Post Confirmation Lambda hoạt động và không tạo dữ liệu trùng


| Trường                   | Nội dung                                                                                                                                                                                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `AUTH-03`                                                                                                                                                                                                                                                                                        |
| **Tên kiểm thử**         | Kích hoạt Post Confirmation và kiểm tra idempotency                                                                                                                                                                                                                                              |
| **Mục tiêu**             | Xác minh Cognito kích hoạt Post Confirmation Lambda sau khi xác nhận tài khoản và việc xử lý lặp không tạo nhiều bản ghi cho cùng một người dùng.                                                                                                                                                |
| **Điều kiện tiên quyết** | Post Confirmation Lambda đã được cấu hình cho Cognito User Pool; Lambda có quyền ghi vào bảng Users trong DynamoDB.                                                                                                                                                                              |
| **Các bước thực hiện**   | 1. Đăng ký và xác nhận một User mới. 2. Kiểm tra CloudWatch Logs của Post Confirmation Lambda. 3. Kiểm tra bản ghi người dùng trong DynamoDB. 4. Thực hiện lại cùng một event trong môi trường kiểm thử hoặc chạy lại cơ chế xử lý với cùng `sub`. 5. Kiểm tra số bản ghi sau lần xử lý thứ hai. |
| **Dữ liệu đầu vào**      | Post Confirmation event chứa cùng một Cognito `sub`.                                                                                                                                                                                                                                             |
| **Kết quả mong đợi**     | Lambda được kích hoạt sau khi xác nhận; DynamoDB có đúng một bản ghi tương ứng với Cognito `sub`; lần xử lý lặp không tạo bản ghi thứ hai và không làm hỏng dữ liệu hiện có.                                                                                                                     |
| **Kết quả thực tế**      | Điền số lần Lambda được gọi và số bản ghi quan sát được.                                                                                                                                                                                                                                         |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                                                                   |
| **Bằng chứng**           | CloudWatch Logs của Lambda và bản ghi DynamoDB đã che thông tin cá nhân.                                                                                                                                                                                                                         |


> Post Confirmation có thể được AWS gọi lại khi xảy ra lỗi hoặc retry. Vì vậy, Lambda cần thực hiện thao tác idempotent dựa trên Cognito `sub`, không dựa trên email.

---

#### AUTH-04 — Đăng nhập bằng thông tin hợp lệ


| Trường                   | Nội dung                                                                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-04`                                                                                                                                         |
| **Tên kiểm thử**         | Đăng nhập bằng tài khoản đã xác nhận                                                                                                              |
| **Mục tiêu**             | Xác minh User có thể đăng nhập bằng thông tin hợp lệ.                                                                                             |
| **Điều kiện tiên quyết** | Tài khoản có trạng thái `CONFIRMED` và chưa bị vô hiệu hóa.                                                                                       |
| **Các bước thực hiện**   | 1. Mở trang đăng nhập. 2. Nhập email và mật khẩu đúng. 3. Nhấn nút đăng nhập. 4. Quan sát kết quả. 5. Gọi một API thông thường sau khi đăng nhập. |
| **Dữ liệu đầu vào**      | Email và mật khẩu hợp lệ.                                                                                                                         |
| **Kết quả mong đợi**     | Cognito xác thực thành công; frontend chuyển người dùng vào hệ thống; token hợp lệ được sử dụng để gọi API; API trả về HTTP `200`.                |
| **Kết quả thực tế**      | Điền sau khi thực hiện kiểm thử.                                                                                                                  |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                    |
| **Bằng chứng**           | Ảnh giao diện sau đăng nhập và response HTTP `200`, không hiển thị token.                                                                         |


---

#### AUTH-05 — Đăng nhập bằng mật khẩu sai


| Trường                   | Nội dung                                                                                                                                                                                                                 |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Mã kiểm thử**          | `AUTH-05`                                                                                                                                                                                                                |
| **Tên kiểm thử**         | Từ chối đăng nhập khi mật khẩu không đúng                                                                                                                                                                                |
| **Mục tiêu**             | Xác minh Cognito không cấp token khi người dùng nhập sai mật khẩu.                                                                                                                                                       |
| **Điều kiện tiên quyết** | Tài khoản User đã được xác nhận.                                                                                                                                                                                         |
| **Các bước thực hiện**   | 1. Mở trang đăng nhập. 2. Nhập email đúng. 3. Nhập mật khẩu sai. 4. Gửi yêu cầu đăng nhập. 5. Quan sát phản hồi.                                                                                                         |
| **Dữ liệu đầu vào**      | Email hợp lệ và mật khẩu không chính xác.                                                                                                                                                                                |
| **Kết quả mong đợi**     | Đăng nhập bị từ chối; hệ thống không cấp token; frontend hiển thị thông báo phù hợp và không tiết lộ thông tin nội bộ. Tùy hợp đồng của frontend hoặc API xác thực, response được chuẩn hóa thành HTTP `400` hoặc `401`. |
| **Kết quả thực tế**      | Điền HTTP status và thông báo quan sát được.                                                                                                                                                                             |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                           |
| **Bằng chứng**           | Ảnh thông báo đăng nhập thất bại hoặc response đã che dữ liệu nhạy cảm.                                                                                                                                                  |


---

#### AUTH-06 — Đăng nhập bằng tài khoản chưa xác nhận


| Trường                   | Nội dung                                                                                                              |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-06`                                                                                                             |
| **Tên kiểm thử**         | Từ chối tài khoản chưa xác nhận                                                                                       |
| **Mục tiêu**             | Xác minh tài khoản chưa xác nhận không thể đăng nhập vào hệ thống.                                                    |
| **Điều kiện tiên quyết** | Tài khoản đã đăng ký nhưng chưa nhập mã xác nhận.                                                                     |
| **Các bước thực hiện**   | 1. Mở trang đăng nhập. 2. Nhập thông tin của tài khoản chưa xác nhận. 3. Gửi yêu cầu đăng nhập. 4. Quan sát phản hồi. |
| **Dữ liệu đầu vào**      | Email và mật khẩu đúng của tài khoản chưa xác nhận.                                                                   |
| **Kết quả mong đợi**     | Cognito từ chối xác thực; không có token được cấp; giao diện thông báo tài khoản cần được xác nhận.                   |
| **Kết quả thực tế**      | Điền HTTP status hoặc thông báo quan sát được.                                                                        |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                        |
| **Bằng chứng**           | Trạng thái tài khoản trong Cognito và ảnh thông báo trên frontend.                                                    |


---

#### AUTH-07 — Gọi API không có token


| Trường                   | Nội dung                                                                                                                                 |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-07`                                                                                                                                |
| **Tên kiểm thử**         | Từ chối request không có Authorization token                                                                                             |
| **Mục tiêu**             | Xác minh API Gateway Authorizer bảo vệ API yêu cầu xác thực.                                                                             |
| **Điều kiện tiên quyết** | Endpoint kiểm thử đã được liên kết với Authorizer.                                                                                       |
| **Các bước thực hiện**   | 1. Chuẩn bị request đến endpoint được bảo vệ. 2. Không thêm header `Authorization`. 3. Gửi request. 4. Ghi nhận status và response body. |
| **Dữ liệu đầu vào**      | Request không có Bearer token.                                                                                                           |
| **Kết quả mong đợi**     | API Gateway từ chối request và trả HTTP `401`; Lambda nghiệp vụ không được thực thi; không có dữ liệu bị thay đổi.                       |
| **Kết quả thực tế**      | Điền HTTP status và response quan sát được.                                                                                              |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                           |
| **Bằng chứng**           | HTTP response `401` và CloudWatch Logs hoặc metrics chứng minh Lambda nghiệp vụ không chạy.                                              |


---

#### AUTH-08 — Gọi API bằng token không hợp lệ


| Trường                   | Nội dung                                                                                                                                                             |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-08`                                                                                                                                                            |
| **Tên kiểm thử**         | Từ chối JWT không hợp lệ                                                                                                                                             |
| **Mục tiêu**             | Xác minh Authorizer kiểm tra định dạng, chữ ký, issuer và audience/client của token.                                                                                 |
| **Điều kiện tiên quyết** | Endpoint được bảo vệ bằng Authorizer.                                                                                                                                |
| **Các bước thực hiện**   | 1. Tạo token kiểm thử bị sửa nội dung hoặc sai chữ ký. 2. Gửi token trong header `Authorization: Bearer <token>`. 3. Gọi endpoint được bảo vệ. 4. Ghi nhận phản hồi. |
| **Dữ liệu đầu vào**      | JWT giả mạo, bị thay đổi hoặc không do đúng Cognito User Pool phát hành.                                                                                             |
| **Kết quả mong đợi**     | Authorizer từ chối token; API trả HTTP `401`; Lambda nghiệp vụ không được thực thi; hệ thống không tin các claim bên trong token giả.                                |
| **Kết quả thực tế**      | Điền HTTP status và thông báo quan sát được.                                                                                                                         |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                       |
| **Bằng chứng**           | HTTP response `401`, không đưa toàn bộ token vào ảnh.                                                                                                                |


---

#### AUTH-09 — Gọi API bằng token hết hạn


| Trường                   | Nội dung                                                                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-09`                                                                                                                                         |
| **Tên kiểm thử**         | Từ chối JWT hết hạn                                                                                                                               |
| **Mục tiêu**             | Xác minh Authorizer kiểm tra claim `exp` trước khi cho phép truy cập API.                                                                         |
| **Điều kiện tiên quyết** | Có token hợp lệ trước đây nhưng đã hết thời hạn sử dụng.                                                                                          |
| **Các bước thực hiện**   | 1. Chuẩn bị token đã hết hạn. 2. Gửi request đến API được bảo vệ. 3. Quan sát response. 4. Kiểm tra Lambda nghiệp vụ có được kích hoạt hay không. |
| **Dữ liệu đầu vào**      | Bearer token có claim `exp` nhỏ hơn thời gian hiện tại.                                                                                           |
| **Kết quả mong đợi**     | API Gateway Authorizer từ chối request và trả HTTP `401`; Lambda nghiệp vụ không được thực thi; dữ liệu không bị thay đổi.                        |
| **Kết quả thực tế**      | Điền HTTP status và kết quả quan sát được.                                                                                                        |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                    |
| **Bằng chứng**           | HTTP response `401` và log hoặc metrics liên quan đã che token.                                                                                   |


---

#### AUTH-10 — User gọi API thông thường bằng danh tính đã xác minh


| Trường                   | Nội dung                                                                                                                                                                                       |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-10`                                                                                                                                                                                      |
| **Tên kiểm thử**         | User truy cập API thông thường bằng claim Cognito                                                                                                                                              |
| **Mục tiêu**             | Xác minh User có thể gọi API thông thường và backend sử dụng claim `sub` đã được Authorizer xác minh làm danh tính hiện tại.                                                                   |
| **Điều kiện tiên quyết** | User đã đăng nhập; token còn hạn; endpoint cho phép tài khoản User truy cập.                                                                                                                   |
| **Các bước thực hiện**   | 1. Đăng nhập bằng tài khoản User. 2. Gửi token hợp lệ đến API thông thường. 3. Quan sát response. 4. Kiểm tra CloudWatch Logs hoặc dữ liệu tạo ra. 5. Đối chiếu ID chủ sở hữu với claim `sub`. |
| **Dữ liệu đầu vào**      | Token hợp lệ của User và request nghiệp vụ hợp lệ.                                                                                                                                             |
| **Kết quả mong đợi**     | API trả HTTP `200`; backend nhận danh tính từ claim `sub` đã xác minh; tài nguyên được truy xuất hoặc tạo cho đúng người dùng; không yêu cầu client tự khai báo danh tính đáng tin cậy.        |
| **Kết quả thực tế**      | Điền HTTP status, dữ liệu trả về và kết quả đối chiếu.                                                                                                                                         |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                 |
| **Bằng chứng**           | HTTP response, CloudWatch Logs đã che token và bản ghi DynamoDB liên quan.                                                                                                                     |


---

#### AUTH-11 — User gọi API quản trị bị từ chối


| Trường                   | Nội dung                                                                                                                    |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-11`                                                                                                                   |
| **Tên kiểm thử**         | Từ chối User truy cập API Admin                                                                                             |
| **Mục tiêu**             | Xác minh tài khoản đã xác thực nhưng không thuộc nhóm Admin không thể sử dụng chức năng quản trị.                           |
| **Điều kiện tiên quyết** | User đã đăng nhập; User không thuộc Cognito Group `Admin`; endpoint chỉ dành cho Admin.                                     |
| **Các bước thực hiện**   | 1. Đăng nhập bằng tài khoản User. 2. Gửi token hợp lệ đến API quản trị. 3. Quan sát response. 4. Kiểm tra dữ liệu mục tiêu. |
| **Dữ liệu đầu vào**      | Token hợp lệ của User và request quản trị hợp lệ về mặt định dạng.                                                          |
| **Kết quả mong đợi**     | API trả HTTP `403`; thao tác quản trị không được thực hiện; dữ liệu không bị tạo, sửa hoặc xóa.                             |
| **Kết quả thực tế**      | Điền HTTP status và trạng thái dữ liệu sau request.                                                                         |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                              |
| **Bằng chứng**           | HTTP response `403`, trạng thái Cognito Group và dữ liệu DynamoDB không thay đổi.                                           |


---

#### AUTH-12 — Admin gọi API quản trị thành công


| Trường                   | Nội dung                                                                                                                                                                           |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-12`                                                                                                                                                                          |
| **Tên kiểm thử**         | Admin truy cập API quản trị                                                                                                                                                        |
| **Mục tiêu**             | Xác minh người dùng thuộc Cognito Group `Admin` có thể thực hiện chức năng quản trị.                                                                                               |
| **Điều kiện tiên quyết** | Tài khoản Admin đã xác nhận và thuộc nhóm `Admin`; endpoint quản trị đã được triển khai.                                                                                           |
| **Các bước thực hiện**   | 1. Đăng nhập bằng tài khoản Admin. 2. Gửi token hợp lệ đến API quản trị. 3. Thực hiện một thao tác quản trị hợp lệ. 4. Kiểm tra response và dữ liệu được cập nhật.                 |
| **Dữ liệu đầu vào**      | Token hợp lệ chứa claim `cognito:groups` có giá trị `Admin` và request hợp lệ.                                                                                                     |
| **Kết quả mong đợi**     | Authorizer xác minh token thành công; backend xác nhận quyền từ claim `cognito:groups`; API trả HTTP `200` hoặc mã thành công phù hợp; thao tác quản trị được thực hiện chính xác. |
| **Kết quả thực tế**      | Điền HTTP status và dữ liệu quan sát được.                                                                                                                                         |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                     |
| **Bằng chứng**           | Cognito Group của tài khoản, response thành công, CloudWatch Logs và bản ghi DynamoDB liên quan.                                                                                   |


---

#### AUTH-13 — Không tin userId hoặc role trong request body


| Trường                   | Nội dung                                                                                                                                                                                                                                                  |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mã kiểm thử**          | `AUTH-13`                                                                                                                                                                                                                                                 |
| **Tên kiểm thử**         | Ngăn giả mạo danh tính và quyền qua request body                                                                                                                                                                                                          |
| **Mục tiêu**             | Xác minh backend chỉ sử dụng `sub` và `cognito:groups` từ token đã được xác minh, không tin `userId` hoặc `role` do client gửi.                                                                                                                           |
| **Điều kiện tiên quyết** | Có tài khoản User thông thường; biết ID của một User khác; có endpoint tạo hoặc cập nhật tài nguyên theo người dùng.                                                                                                                                      |
| **Các bước thực hiện**   | 1. Đăng nhập bằng tài khoản User A. 2. Gửi request kèm `userId` của User B. 3. Kiểm tra chủ sở hữu của dữ liệu được tạo hoặc response từ API. 4. Gửi request khác với `role: Admin` trong body. 5. Thử gọi API quản trị. 6. Kiểm tra response và dữ liệu. |
| **Dữ liệu đầu vào**      | Token hợp lệ của User A; `userId` của User B; `role: Admin` trong request body.                                                                                                                                                                           |
| **Kết quả mong đợi**     | Hệ thống bỏ qua hoặc từ chối `userId` và `role` không đáng tin cậy; không cho phép User A thao tác dữ liệu của User B; không cấp quyền Admin; API quản trị trả HTTP `403`; không có thay đổi trái phép trong DynamoDB.                                    |
| **Kết quả thực tế**      | Điền HTTP status, owner ID được lưu và trạng thái dữ liệu sau kiểm thử.                                                                                                                                                                                   |
| **Trạng thái**           | `PASS`, `FAIL` hoặc `BLOCKED`.                                                                                                                                                                                                                            |
| **Bằng chứng**           | Request body đã che dữ liệu nhạy cảm, response `403`, CloudWatch Logs và bản ghi DynamoDB chứng minh chủ sở hữu thực tế.                                                                                                                                  |


---

#### Kiểm tra IAM liên quan

Ngoài 13 test case trên, nhóm kiểm tra IAM Execution Role của các Lambda Function liên quan đến xác thực:

- Post Confirmation Lambda chỉ được ghi vào đúng bảng DynamoDB cần thiết.
- Lambda không được cấp quyền quản trị toàn bộ DynamoDB nếu không cần thiết.
- Lambda chỉ được ghi log vào CloudWatch Logs theo phạm vi được cấu hình.
- API Gateway chỉ được phép gọi đúng Lambda Function.
- Cognito chỉ được phép kích hoạt đúng Post Confirmation Lambda.
- Không lưu AWS Access Key hoặc Secret Access Key trong mã nguồn hay biến môi trường Lambda.

Có thể kiểm tra quyền tối thiểu bằng cách thử cho Lambda truy cập một bảng hoặc tài nguyên không nằm trong IAM Policy. Kết quả mong đợi là thao tác bị AWS từ chối với lỗi `AccessDenied`, trong khi luồng hợp lệ vẫn hoạt động bình thường.

#### Bảng tổng hợp kết quả


| Mã        | Nội dung kiểm thử                | HTTP status mong đợi              | Trạng thái    |
| --------- | -------------------------------- | --------------------------------- | ------------- |
| `AUTH-01` | Đăng ký User thành công          | `200` hoặc mã thành công theo API | Chưa kiểm thử |
| `AUTH-02` | Xác nhận tài khoản thành công    | `200`                             | Chưa kiểm thử |
| `AUTH-03` | Post Confirmation và idempotency | Thành công, không trùng dữ liệu   | Chưa kiểm thử |
| `AUTH-04` | Đăng nhập đúng thông tin         | `200`                             | Chưa kiểm thử |
| `AUTH-05` | Đăng nhập sai mật khẩu           | `400` hoặc `401`                  | Chưa kiểm thử |
| `AUTH-06` | Đăng nhập khi chưa xác nhận      | `400` hoặc `401`                  | Chưa kiểm thử |
| `AUTH-07` | API không có token               | `401`                             | Chưa kiểm thử |
| `AUTH-08` | API dùng token không hợp lệ      | `401`                             | Chưa kiểm thử |
| `AUTH-09` | API dùng token hết hạn           | `401`                             | Chưa kiểm thử |
| `AUTH-10` | User gọi API thông thường        | `200`                             | Chưa kiểm thử |
| `AUTH-11` | User gọi API Admin               | `403`                             | Chưa kiểm thử |
| `AUTH-12` | Admin gọi API Admin              | `200` hoặc mã thành công phù hợp  | Chưa kiểm thử |
| `AUTH-13` | Giả mạo `userId` hoặc `role`     | `403` hoặc bỏ qua dữ liệu giả mạo | Chưa kiểm thử |


