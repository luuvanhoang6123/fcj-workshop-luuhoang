---
title: "Tổng quan và môi trường kiểm thử"
date: 2026-08-03
weight: 1
chapter: false
pre: " <b> 5.5.1. </b> "
---

### Tổng quan và môi trường kiểm thử

#### Tổng quan

Sau khi hoàn thành việc triển khai các dịch vụ AWS ở mục **5.4**, nhóm tiến hành kiểm thử hệ thống Live Auction nhằm xác minh khả năng hoạt động của toàn bộ hệ thống trong môi trường AWS.

Quá trình kiểm thử không chỉ kiểm tra sự tồn tại hoặc cấu hình riêng lẻ của từng tài nguyên mà còn tập trung xác minh các luồng nghiệp vụ đầu cuối giữa frontend, API Gateway, Lambda, Amazon SQS FIFO, DynamoDB, S3 và CloudFront.

Các kết quả kiểm thử được ghi nhận thông qua:

- Giao diện User Frontend và Admin Frontend.
- HTTP response từ REST API.
- Thông điệp nhận được qua WebSocket.
- Dữ liệu được lưu trong Amazon DynamoDB.
- Trạng thái thông điệp trong Amazon SQS FIFO.
- Tệp và phiên bản đối tượng trong Amazon S3.
- Nhật ký thực thi Lambda trong Amazon CloudWatch Logs.
- Các chỉ số giám sát trên Amazon CloudWatch Metrics.

Mục tiêu của quá trình kiểm thử hệ thống bao gồm:

- Xác minh các chức năng chính của hệ thống Live Auction hoạt động đúng với yêu cầu.
- Kiểm tra khả năng xác thực và phân quyền giữa người dùng thông thường và quản trị viên.
- Xác minh REST API trả về đúng dữ liệu, mã trạng thái HTTP và thông báo lỗi.
- Kiểm tra kết nối WebSocket và khả năng cập nhật dữ liệu đấu giá theo thời gian thực.
- Xác minh yêu cầu đặt giá được đưa vào Amazon SQS FIFO và xử lý theo đúng thứ tự trong cùng một nhóm thông điệp.
- Kiểm tra dữ liệu phiên đấu giá, vật phẩm và lịch sử đặt giá được lưu chính xác trong Amazon DynamoDB.
- Xác minh User Frontend và Admin Frontend có thể được truy cập thông qua Amazon CloudFront.
- Kiểm tra khả năng tải lên, lưu trữ và hiển thị hình ảnh vật phẩm từ Amazon S3.
- Kiểm tra cách hệ thống xử lý dữ liệu đầu vào không hợp lệ, lỗi dịch vụ và yêu cầu trùng lặp.
- Xác minh nhật ký và các chỉ số giám sát cung cấp đủ thông tin để theo dõi và xử lý sự cố.
- Đánh giá mức độ đáp ứng của hệ thống trước khi đưa vào sử dụng.

#### Phạm vi kiểm thử

Phạm vi kiểm thử bao gồm các nhóm chức năng sau:

1. Xác thực và phân quyền người dùng bằng Amazon Cognito.
2. Các chức năng nghiệp vụ được cung cấp thông qua Amazon API Gateway REST.
3. Kết nối và trao đổi dữ liệu theo thời gian thực thông qua API Gateway WebSocket.
4. Quá trình thực thi nghiệp vụ của các AWS Lambda Function.
5. Luồng đặt giá đầu cuối của hệ thống Live Auction.
6. Thứ tự và khả năng xử lý bất đồng bộ của Amazon SQS FIFO.
7. Tính toàn vẹn và nhất quán của dữ liệu trong Amazon DynamoDB.
8. Khả năng lưu trữ và phân phối nội dung bằng Amazon S3 và Amazon CloudFront.
9. Khả năng giám sát, ghi nhật ký và truy vết lỗi bằng Amazon CloudWatch.
10. Các trường hợp lỗi, yêu cầu không hợp lệ và hành vi truy cập trái phép.
11. Khả năng hoạt động của User Frontend và Admin Frontend trên môi trường AWS.

Việc kiểm tra cấu hình tài nguyên bằng AWS Management Console hoặc Terraform chỉ được sử dụng làm bằng chứng bổ trợ. Kết quả kiểm thử hệ thống phải dựa trên hành vi thực tế của luồng nghiệp vụ và dữ liệu được tạo ra sau khi thực hiện kiểm thử.

#### Kiến trúc được kiểm thử

Kiến trúc kiểm thử của hệ thống Live Auction gồm hai nhóm luồng chính.

##### Luồng REST API

```text
User Frontend hoặc Admin Frontend
        ↓
Amazon CloudFront
        ↓
Amazon API Gateway REST
        ↓
Amazon Cognito Authorizer
        ↓
AWS Lambda
        ↓
Amazon DynamoDB hoặc Amazon S3
        ↓
Phản hồi về Frontend
```

Luồng này được sử dụng cho các chức năng như:

- Đăng ký và đăng nhập.
- Lấy danh sách phiên đấu giá.
- Xem thông tin phiên và vật phẩm.
- Quản lý phiên đấu giá.
- Quản lý vật phẩm.
- Tải lên và truy xuất hình ảnh.
- Thực hiện các chức năng quản trị.

##### Luồng WebSocket và đặt giá

```text
User Frontend
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
Cập nhật dữ liệu trên User Frontend
```

Luồng này được sử dụng để:

- Thiết lập kết nối WebSocket.
- Tham gia phòng đấu giá.
- Gửi yêu cầu đặt giá.
- Xử lý yêu cầu đặt giá theo thứ tự.
- Cập nhật giá hiện tại và người đặt giá cao nhất.
- Lưu lịch sử đặt giá.
- Gửi kết quả đến những người dùng đang theo dõi phiên đấu giá.

#### Môi trường kiểm thử

Hệ thống được kiểm thử trên hạ tầng AWS tại Region:

```text
Asia Pacific (Singapore) – ap-southeast-1
```

Thông tin môi trường kiểm thử được tổng hợp như sau:


| Thành phần                   | Môi trường kiểm thử                        |
| ---------------------------- | ------------------------------------------ |
| AWS Region                   | `ap-southeast-1`                           |
| Giao diện người dùng         | User Frontend                              |
| Giao diện quản trị           | Admin Frontend                             |
| Phân phối frontend           | Amazon CloudFront                          |
| Lưu trữ frontend và hình ảnh | Amazon S3                                  |
| Xác thực người dùng          | Amazon Cognito                             |
| REST API                     | Amazon API Gateway REST                    |
| Giao tiếp thời gian thực     | API Gateway WebSocket                      |
| Xử lý nghiệp vụ              | AWS Lambda                                 |
| Xử lý hàng đợi đặt giá       | Amazon SQS FIFO                            |
| Lưu trữ dữ liệu              | Amazon DynamoDB                            |
| Nhật ký và giám sát          | Amazon CloudWatch                          |
| Trình duyệt kiểm thử         | Trình duyệt hỗ trợ JavaScript và WebSocket |
| Kết nối mạng                 | Kết nối Internet ổn định                   |


Các địa chỉ CloudFront, API Gateway và WebSocket API phải được lấy từ môi trường triển khai thực tế. Khi đưa vào báo cáo, nhóm có thể che một phần thông tin nếu địa chỉ đó không cần thiết phải công khai.

#### Các dịch vụ AWS tham gia kiểm thử


| Dịch vụ                     | Vai trò trong quá trình kiểm thử                                                       |
| --------------------------- | -------------------------------------------------------------------------------------- |
| **Amazon Cognito**          | Xác thực người dùng, phát hành token và quản lý nhóm quyền User/Admin.                 |
| **Amazon API Gateway REST** | Tiếp nhận các yêu cầu HTTP từ User Frontend và Admin Frontend.                         |
| **API Gateway WebSocket**   | Duy trì kết nối và truyền dữ liệu đấu giá theo thời gian thực.                         |
| **AWS Lambda**              | Xử lý xác thực bổ sung, nghiệp vụ đấu giá, đặt giá và broadcast dữ liệu.               |
| **Amazon DynamoDB**         | Lưu dữ liệu người dùng, phiên đấu giá, vật phẩm, kết nối WebSocket và lịch sử đặt giá. |
| **Amazon SQS FIFO**         | Tiếp nhận và duy trì thứ tự các yêu cầu đặt giá trong cùng một Message Group.          |
| **Amazon S3**               | Lưu User Frontend, Admin Frontend và hình ảnh vật phẩm đấu giá.                        |
| **Amazon CloudFront**       | Phân phối nội dung frontend và nội dung tĩnh từ các S3 bucket riêng tư.                |
| **Amazon CloudWatch**       | Lưu log, theo dõi chỉ số và hỗ trợ truy vết lỗi của hệ thống.                          |


#### Giao diện được kiểm thử

Hệ thống có hai giao diện chính:

##### User Frontend

User Frontend được sử dụng để kiểm thử các chức năng dành cho người dùng thông thường, bao gồm:

- Đăng ký và đăng nhập.
- Xem danh sách phiên đấu giá.
- Xem thông tin chi tiết vật phẩm.
- Tham gia phiên đấu giá.
- Thiết lập kết nối WebSocket.
- Gửi yêu cầu đặt giá.
- Nhận giá mới theo thời gian thực.
- Xem hình ảnh và trạng thái của vật phẩm.

##### Admin Frontend

Admin Frontend được sử dụng để kiểm thử các chức năng quản trị, bao gồm:

- Đăng nhập bằng tài khoản Admin.
- Tạo và cập nhật phiên đấu giá.
- Quản lý vật phẩm đấu giá.
- Quản lý hình ảnh vật phẩm.
- Bắt đầu hoặc kết thúc phiên đấu giá theo quyền được cấp.
- Truy cập các REST API dành riêng cho quản trị viên.
- Xác minh tài khoản User không thể sử dụng các chức năng Admin.

#### Tài khoản kiểm thử

Quá trình kiểm thử sử dụng ít nhất hai loại tài khoản:


| Loại tài khoản | Mục đích                                                                     |
| -------------- | ---------------------------------------------------------------------------- |
| **User**       | Kiểm thử đăng nhập, xem phiên đấu giá, tham gia WebSocket và đặt giá.        |
| **Admin**      | Kiểm thử các chức năng tạo, cập nhật và quản lý phiên hoặc vật phẩm đấu giá. |


Ngoài hai tài khoản hợp lệ, một số trường hợp kiểm thử có thể sử dụng:

- Tài khoản chưa được xác nhận.
- Tài khoản nhập sai mật khẩu.
- Tài khoản không thuộc nhóm Admin.
- Token không hợp lệ.
- Token đã hết hạn.
- Request không có token.

Các tài khoản kiểm thử phải được tạo riêng cho mục đích kiểm thử. Không sử dụng tài khoản cá nhân hoặc tài khoản có dữ liệu quan trọng.



