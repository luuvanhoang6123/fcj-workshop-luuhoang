---
title: "Phương pháp và định dạng test case"
date: 2026-08-03
weight: 2
chapter: false
pre: " <b> 5.5.2. </b> "
---

### Phương pháp và định dạng test case

#### Phương pháp kiểm thử

Nhóm thực hiện kiểm thử hệ thống theo phương pháp **kiểm thử hộp đen**, kết hợp với việc kiểm tra dữ liệu và nhật ký trên các dịch vụ AWS.

Đối với mỗi test case, nhóm cung cấp dữ liệu đầu vào và thực hiện thao tác thông qua:

- User Frontend.
- Admin Frontend.
- REST API.
- Kết nối WebSocket.
- AWS Management Console.
- Công cụ kiểm thử API như Postman hoặc `curl`.
- Công cụ kiểm thử tải nếu có.

Sau đó, kết quả thực tế được so sánh với kết quả mong đợi để xác định trạng thái của test case.

Quy trình kiểm thử được thực hiện theo trình tự:

```text
Chuẩn bị môi trường và dữ liệu kiểm thử
1. Xác nhận điều kiện tiên quyết
2. Thực hiện các bước kiểm thử
3. Quan sát kết quả trên frontend hoặc API
4. Kiểm tra dữ liệu và log trên AWS
5. So sánh với kết quả mong đợi
6. Ghi nhận trạng thái
7. Lưu bằng chứng kiểm thử
```

Việc đánh giá không chỉ dựa trên nội dung hiển thị ở frontend. Tùy theo test case, nhóm còn kiểm tra:

- HTTP status và response body của REST API.
- Thông điệp gửi và nhận qua WebSocket.
- Bản ghi được tạo hoặc cập nhật trong Amazon DynamoDB.
- Trạng thái message trong Amazon SQS FIFO.
- CloudWatch Logs của AWS Lambda.
- CloudWatch Metrics của Lambda, API Gateway, DynamoDB và SQS.
- Object hoặc phiên bản object trong Amazon S3.
- Nội dung được phân phối qua Amazon CloudFront.
- Trạng thái người dùng và nhóm quyền trong Amazon Cognito.

#### Phân loại test case

Các test case được chia thành các nhóm tương ứng với chức năng và thành phần của hệ thống:


| Tiền tố    | Nhóm kiểm thử                         | Ví dụ         |
| ---------- | ------------------------------------- | ------------- |
| `AUTH`     | Xác thực và phân quyền                | `AUTH-01`     |
| `API`      | REST API và nghiệp vụ quản lý đấu giá | `API-01`      |
| `WS`       | WebSocket và cập nhật thời gian thực  | `WS-01`       |
| `BID`      | Luồng đặt giá đầu cuối                | `BID-01`      |
| `FIFO`     | Thứ tự và xử lý message bằng SQS FIFO | `FIFO-01`     |
| `DB`       | DynamoDB và tính toàn vẹn dữ liệu     | `DB-01`       |
| `STORAGE`  | Amazon S3 và CloudFront               | `STORAGE-01`  |
| `RECOVERY` | Xử lý lỗi và khả năng phục hồi        | `RECOVERY-01` |
| `PERF`     | Hiệu năng và tải đồng thời            | `PERF-01`     |
| `SEC`      | Bảo mật hệ thống                      | `SEC-01`      |


Khi test case có trạng thái `FAIL`, nhóm cần ghi nhận thêm:

- Bước xảy ra lỗi.
- Thời điểm xảy ra lỗi.
- Thông báo lỗi quan sát được.
- HTTP status nếu liên quan đến REST API.
- Mã request hoặc request ID nếu có.
- Lambda Function liên quan.
- CloudWatch Log Group hoặc Log Stream liên quan.
- Ảnh hưởng của lỗi đến hệ thống.
- Hướng xử lý hoặc nhiệm vụ sửa lỗi.

Khi test case có trạng thái `BLOCKED`, nhóm phải ghi rõ:

- Thành phần đang thiếu.
- Chức năng phụ thuộc chưa hoàn thành.
- Cấu hình chưa được triển khai.
- Quyền truy cập chưa được cấp.
- Điều kiện cần hoàn thành trước khi kiểm thử lại.

#### Quy định thu thập bằng chứng


| Loại bằng chứng          | Trường hợp sử dụng                                                |
| ------------------------ | ----------------------------------------------------------------- |
| Ảnh User Frontend        | Chứng minh chức năng dành cho người dùng hoạt động.               |
| Ảnh Admin Frontend       | Chứng minh chức năng quản trị và phân quyền.                      |
| HTTP request và response | Chứng minh REST API trả về status và dữ liệu đúng.                |
| WebSocket message        | Chứng minh dữ liệu được gửi và nhận theo thời gian thực.          |
| CloudWatch Logs          | Chứng minh Lambda được kích hoạt và xử lý nghiệp vụ.              |
| CloudWatch Metrics       | Chứng minh số request, lỗi, độ trễ hoặc lượng message được xử lý. |
| DynamoDB item            | Chứng minh dữ liệu được tạo hoặc cập nhật chính xác.              |
| SQS Metrics              | Chứng minh message được gửi, nhận và xóa khỏi Queue.              |
| Nội dung DLQ             | Chứng minh message thất bại được chuyển vào Dead-letter Queue.    |
| S3 object                | Chứng minh tệp được tải lên và lưu đúng bucket.                   |
| CloudFront response      | Chứng minh frontend hoặc nội dung tĩnh được phân phối thành công. |
| Cognito User Pool        | Chứng minh trạng thái tài khoản hoặc nhóm quyền.                  |


Mỗi hình ảnh nên có tiêu đề và mô tả rõ test case tương ứng, ví dụ:

```text
Hình 5.5.2.1: Kết quả thực hiện test case AUTH-01
```

Không nên sử dụng một ảnh chung cho nhiều test case nếu ảnh đó không chứng minh rõ kết quả của từng trường hợp.

#### Quản lý dữ liệu kiểm thử

Để kết quả có thể được kiểm tra lại, nhóm cần chuẩn bị dữ liệu kiểm thử trước khi thực hiện:

- Tài khoản User đã được xác nhận.
- Tài khoản Admin thuộc đúng nhóm quyền.
- Tài khoản User không có quyền Admin.
- Phiên đấu giá ở trạng thái `SCHEDULED`.
- Phiên đấu giá ở trạng thái `ACTIVE`.
- Phiên đấu giá ở trạng thái `ENDED`.
- Vật phẩm có giá khởi điểm và bước giá tối thiểu.
- ID phiên và ID vật phẩm hợp lệ.
- ID tài nguyên không tồn tại.
- Mức giá hợp lệ và không hợp lệ.
- Hình ảnh đúng và sai định dạng.
- Tệp có kích thước nằm trong và vượt quá giới hạn.
- Kết nối WebSocket đang hoạt động.
- Message trùng lặp để kiểm tra idempotency nếu chức năng này đã được triển khai.

Các dữ liệu kiểm thử phải được phân biệt với dữ liệu thật. Sau khi hoàn thành kiểm thử, nhóm cần xóa hoặc đánh dấu dữ liệu kiểm thử nếu dữ liệu đó không còn cần thiết.

#### Trình tự thực hiện các nhóm kiểm thử

Các nhóm test case nên được thực hiện theo thứ tự phụ thuộc sau:

1. Kiểm tra môi trường và các endpoint.
2. Kiểm thử xác thực và phân quyền.
3. Kiểm thử REST API.
4. Kiểm thử dữ liệu trong DynamoDB.
5. Kiểm thử kết nối WebSocket.
6. Kiểm thử luồng đặt giá đầu cuối.
7. Kiểm thử SQS FIFO và xử lý đồng thời.
8. Kiểm thử S3 và CloudFront.
9. Kiểm thử xử lý lỗi và khả năng phục hồi.
10. Kiểm thử bảo mật.
11. Kiểm thử hiệu năng.
12. Tổng hợp kết quả.

#### Bảo mật thông tin kiểm thử

nhóm không được hiển thị:

- Mật khẩu tài khoản kiểm thử.
- Access Token, ID Token hoặc Refresh Token.
- Header `Authorization`.
- AWS Access Key ID.
- AWS Secret Access Key.
- AWS Session Token.
- Cognito Client Secret.
- Cookie hoặc thông tin phiên đăng nhập.
- Nội dung tệp `.env`.
- Presigned URL còn hiệu lực.
- Dữ liệu cá nhân không cần thiết.

Nếu request hoặc log chứa thông tin nhạy cảm, nhóm phải che hoặc loại bỏ phần dữ liệu đó trước khi đưa vào báo cáo. Chỉ giữ lại những thông tin cần thiết để chứng minh test case đã được thực hiện và cho kết quả tương ứng.