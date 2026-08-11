---
title: "AWS IAM và Amazon Cognito"
date: 2026-07-27
weight: 1
chapter: false
pre: " <b> 5.4.1. </b> "
---

## Tổng quan

Hệ thống **Live Auction** sử dụng **AWS Identity and Access Management (IAM)** và **Amazon Cognito** để thực hiện hai nhiệm vụ khác nhau:

* **AWS IAM** quản lý quyền truy cập giữa các dịch vụ AWS.
* **Amazon Cognito** quản lý tài khoản, xác thực người dùng và hỗ trợ phân biệt quyền của User với Admin.

AWS IAM không được sử dụng để tạo tài khoản đăng nhập trực tiếp cho người dùng của ứng dụng. Thay vào đó, tài khoản User và Admin được quản lý thông qua Cognito User Pool.

Toàn bộ tài nguyên IAM và Cognito của hệ thống được khai báo bằng Terraform trong module:

```text
infra/03-identity
```

Sau khi Terraform triển khai hoàn tất, nhóm kiểm tra các tài nguyên trực tiếp trên AWS Management Console để xác nhận cấu hình thực tế.


## AWS Identity and Access Management

### Vai trò của AWS IAM

AWS IAM cung cấp các Role, Policy và cơ chế phân quyền để các dịch vụ AWS trong hệ thống có thể giao tiếp với nhau.

Các quyền được cấu hình theo nguyên tắc **quyền tối thiểu — least privilege**, nghĩa là mỗi thành phần chỉ được cấp những quyền cần thiết để thực hiện đúng nhiệm vụ của mình.

Trong hệ thống Live Auction, AWS IAM được sử dụng để:

* Cho phép Lambda ghi log vào Amazon CloudWatch.
* Cho phép các Lambda cần thiết đọc và ghi dữ liệu trong Amazon DynamoDB.
* Cho phép Lambda gửi thông điệp đến Amazon SQS FIFO.
* Cho phép Lambda xử lý thông điệp được chuyển đến từ SQS FIFO.
* Cho phép Lambda gửi dữ liệu cập nhật qua API Gateway WebSocket.
* Cho phép Lambda Post Confirmation thực hiện nghiệp vụ sau khi tài khoản được xác nhận.
* Giới hạn phạm vi tài nguyên mà từng Lambda Function được phép truy cập.
* Thiết lập quan hệ tin cậy giữa IAM Role và dịch vụ sử dụng Role.
* Kiểm soát quyền truy cập giữa các thành phần trong hệ thống.

### Cơ chế phân quyền của các thành phần

| Thành phần | Cơ chế phân quyền |
| --- | --- |
| **Business Logic Lambda** | Sử dụng IAM Role để ghi log vào CloudWatch và truy cập các bảng DynamoDB cần thiết. |
| **Bid Processing Lambda** | Sử dụng IAM Role để xử lý thông điệp từ SQS FIFO, cập nhật DynamoDB và gửi kết quả qua WebSocket API. |
| **WebSocket Lambda** | Sử dụng IAM Role để lưu, đọc hoặc xóa dữ liệu kết nối WebSocket trong DynamoDB và ghi log vào CloudWatch. |
| **Cognito Post Confirmation Lambda** | Sử dụng IAM Role để ghi log và thực hiện nghiệp vụ sau khi người dùng xác nhận tài khoản. |
| **API Gateway** | Gọi Lambda thông qua Lambda Permission được cấu hình cho API, route hoặc stage tương ứng. |
| **CloudFront và Amazon S3** | CloudFront truy cập nội dung frontend trong S3 thông qua Origin Access Control và S3 Bucket Policy. |

Các IAM Role, IAM Policy và Lambda Permission được khai báo bằng Terraform. Nhờ đó, cấu hình phân quyền có thể được quản lý nhất quán và tái sử dụng trong những lần triển khai tiếp theo.

## Kiểm tra IAM Role trên AWS Management Console

### Bước 1: Truy cập dịch vụ IAM

Đăng nhập vào **AWS Management Console**.

Tại thanh tìm kiếm phía trên, nhập:

```text
IAM
```

Chọn **IAM — Identity and Access Management**.

### Bước 2: Mở danh sách IAM Role

Tại thanh điều hướng bên trái, chọn:

```text
Access management → Roles
```

Trang Roles hiển thị danh sách IAM Role hiện có trong tài khoản AWS.

### Bước 3: Tìm IAM Role của dự án

Trong ô tìm kiếm, nhập tiền tố tài nguyên của hệ thống:

```text
la-
```

Kiểm tra các IAM Role được Terraform tạo cho Lambda Function và các thành phần liên quan.

Các nội dung cần kiểm tra gồm:

* Tên Role có đúng quy ước của dự án hay không.
* Trusted entity của Role.
* Thời điểm Role được tạo.
* Role có được gán cho đúng dịch vụ hay không.
* Các Role cần thiết của Lambda đã được tạo đầy đủ hay chưa.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/iam-role-list.png"
    title="Hình 5.4.1.1: Danh sách IAM Role của hệ thống Live Auction"
    width="100%"
>}}

### Bước 4: Kiểm tra Permission Policy

Chọn một IAM Role được gán cho Lambda Function.

Trong tab **Permissions**, kiểm tra:

* Danh sách Permission Policy được gán với Role.
* Quyền ghi log vào CloudWatch.
* Quyền truy cập DynamoDB.
* Quyền truy cập SQS FIFO nếu Lambda xử lý hàng đợi.
* Quyền gửi dữ liệu qua WebSocket API nếu Lambda thực hiện cập nhật thời gian thực.
* Phạm vi Resource mà Policy cho phép truy cập.

Không nên gán quyền toàn phần như `AdministratorAccess` cho Lambda Function. Policy cần giới hạn theo đúng tài nguyên và nghiệp vụ của từng hàm.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/iam-role-permissions.png"
    title="Hình 5.4.1.2: Các Permission Policy được gán với IAM Role của Lambda"
    width="100%"
>}}

### Bước 5: Kiểm tra Trust Relationship

Trong trang chi tiết của IAM Role, mở tab:

```text
Trust relationships
```

Đối với Lambda Execution Role, phần Trusted entities phải cho phép dịch vụ:

```text
lambda.amazonaws.com
```

Trust Relationship xác định dịch vụ nào được phép sử dụng IAM Role. Cấu hình này giúp ngăn các dịch vụ không liên quan sử dụng Role của Lambda.

Có thể chọn **View policy document** để xem nội dung Trust Policy. Không chỉnh sửa trực tiếp trên Console nếu tài nguyên đang được quản lý bằng Terraform.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/iam-role-trust-relationship.png"
    title="Hình 5.4.1.3: Trust Relationship của Lambda Execution Role"
    width="100%"
>}}

## Amazon Cognito

### Vai trò của Amazon Cognito

Amazon Cognito được sử dụng để quản lý và xác thực tài khoản cho cả **User Frontend** và **Admin Frontend**.

Cognito chịu trách nhiệm:

* Đăng ký tài khoản người dùng.
* Đăng nhập và đăng xuất.
* Xác minh thông tin tài khoản.
* Quản lý mật khẩu.
* Cấp token sau khi đăng nhập thành công.
* Lưu trữ các thuộc tính cơ bản của tài khoản.
* Quản lý hai nhóm quyền `user` và `admin`.
* Cung cấp thông tin xác thực để API kiểm tra người gửi yêu cầu.
* Kích hoạt Lambda Post Confirmation sau khi người dùng xác nhận tài khoản.

### Cognito User Pool

Cognito User Pool là thư mục quản lý tài khoản của hệ thống.

User Pool lưu trữ và quản lý các thông tin như:

* Username hoặc email.
* Mật khẩu đã được Cognito quản lý an toàn.
* Trạng thái xác nhận tài khoản.
* Trạng thái kích hoạt tài khoản.
* Các thuộc tính của người dùng.
* Chính sách mật khẩu.
* Quy trình đăng ký và đăng nhập.
* Nhóm quyền của tài khoản.

Hệ thống sử dụng chung một User Pool cho User và Admin. Quyền truy cập được phân biệt thông qua hai Cognito Group:

```text
user
admin
```

### Cognito App Client

App Client cho phép frontend kết nối với Cognito User Pool để thực hiện đăng ký và đăng nhập.

Sau khi xác thực thành công, Cognito có thể trả về:

* **ID Token:** Chứa thông tin nhận dạng của tài khoản.
* **Access Token:** Được sử dụng để chứng minh quyền truy cập.
* **Refresh Token:** Được sử dụng để yêu cầu token mới khi token hiện tại hết hạn.

Frontend gửi token trong tiêu đề `Authorization` khi gọi REST API. API Gateway Authorizer hoặc Lambda Authorizer kiểm tra tính hợp lệ của token trước khi chuyển yêu cầu đến Lambda xử lý nghiệp vụ.

Đối với các chức năng quản trị, hệ thống tiếp tục kiểm tra tài khoản có thuộc nhóm `admin` hay không trước khi cho phép thực hiện yêu cầu.

### Lambda Post Confirmation

Hệ thống cấu hình Lambda Post Confirmation cho Cognito User Pool.

Sau khi người dùng xác nhận đăng ký thành công, Cognito kích hoạt Lambda này để thực hiện các bước xử lý cần thiết sau xác nhận, chẳng hạn như khởi tạo dữ liệu tài khoản liên quan trong hệ thống.

Cấu hình bao gồm:

* Lambda Function xử lý sự kiện Post Confirmation.
* IAM Role dành cho Lambda.
* Lambda Permission cho phép Cognito gọi Lambda.
* Liên kết Lambda Trigger với Cognito User Pool.
* CloudWatch Log Group để lưu log thực thi.

## Luồng xác thực tài khoản

Luồng xác thực tổng quát của hệ thống được thực hiện như sau:

1. User hoặc Admin nhập thông tin đăng nhập trên giao diện tương ứng.
2. Frontend gửi yêu cầu xác thực đến Amazon Cognito.
3. Cognito kiểm tra thông tin tài khoản trong User Pool.
4. Nếu thông tin hợp lệ, Cognito trả token về frontend.
5. Frontend gửi token kèm theo các yêu cầu API.
6. API Gateway Authorizer hoặc Lambda Authorizer kiểm tra token.
7. Backend kiểm tra nhóm `user` hoặc `admin` khi xử lý chức năng yêu cầu phân quyền.
8. Yêu cầu chỉ được xử lý khi tài khoản đã xác thực và có quyền phù hợp.
9. Các API quản trị từ chối yêu cầu từ tài khoản không thuộc nhóm `admin`.

## Kiểm tra Cognito User Pool trên AWS Management Console

### Bước 1: Truy cập Amazon Cognito

Tại thanh tìm kiếm của AWS Management Console, nhập:

```text
Cognito
```

Chọn **Amazon Cognito**.

### Bước 2: Mở danh sách User Pool

Trong giao diện Amazon Cognito, chọn:

```text
User pools
```

Kiểm tra User Pool được Terraform tạo cho hệ thống Live Auction.

Các thông tin cần kiểm tra gồm:

* Tên User Pool.
* Trạng thái User Pool.
* User Pool ID.
* AWS Region.
* Thời điểm cập nhật gần nhất.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-user-pool.png"
    title="Hình 5.4.1.4: Cognito User Pool của hệ thống Live Auction"
    width="100%"
>}}

### Bước 3: Kiểm tra cấu hình User Pool

Chọn User Pool của hệ thống.

Tại trang tổng quan và các mục cấu hình, kiểm tra:

* Tên và ID của User Pool.
* AWS Region.
* Phương thức đăng nhập.
* Các thuộc tính dùng để xác thực.
* Chính sách mật khẩu.
* Trạng thái tự đăng ký tài khoản.
* Cơ chế xác minh email.
* Cấu hình Multi-Factor Authentication nếu có.
* Cấu hình bảo mật và thời gian hiệu lực token.

Không thay đổi trực tiếp cấu hình trên Console vì tài nguyên đang được quản lý bằng Terraform. Nếu cần thay đổi, nhóm cập nhật mã Terraform và triển khai lại.

### Bước 4: Kiểm tra App Client

Trong trang chi tiết User Pool, mở:

```text
Applications → App clients
```

Kiểm tra App Client được frontend sử dụng để đăng ký và đăng nhập.

Các thông tin cần kiểm tra gồm:

* Tên App Client.
* Client ID.
* Authentication flow được bật.
* Thời gian hiệu lực của token.
* Callback URL và Sign-out URL nếu được sử dụng.
* Cấu hình App Client có phù hợp với frontend hay không.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-app-client.png"
    title="Hình 5.4.1.5: App Client được cấu hình cho hệ thống Live Auction"
    width="100%"
>}}

### Bước 5: Kiểm tra Cognito Group

Trong User Pool, mở:

```text
User management → Groups
```

Kiểm tra hai nhóm quyền của hệ thống:

```text
user
admin
```

Nhóm `user` dành cho tài khoản người dùng thông thường. Nhóm `admin` dành cho tài khoản được phép truy cập các chức năng quản trị.

Các thông tin cần kiểm tra gồm:

* Tên Group.
* Description của Group.
* Precedence nếu được cấu hình.
* Số lượng tài khoản trong từng Group.
* Tài khoản Admin có được gán đúng vào nhóm `admin` hay không.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-groups.png"
    title="Hình 5.4.1.6: Các nhóm user và admin trong Cognito User Pool"
    width="100%"
>}}

### Bước 6: Kiểm tra danh sách tài khoản

Trong User Pool, mở:

```text
User management → Users
```

Kiểm tra các tài khoản đã được tạo trong hệ thống.

Các thông tin cần kiểm tra gồm:

* Username.
* Email.
* Trạng thái xác nhận tài khoản.
* Trạng thái kích hoạt tài khoản.
* Ngày tạo tài khoản.
* Group được gán cho tài khoản.

Tài khoản quản trị phải được gán vào nhóm `admin` trước khi sử dụng các chức năng quản trị.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-users.png"
    title="Hình 5.4.1.7: Danh sách tài khoản được quản lý trong Amazon Cognito"
    width="100%"
>}}

### Bước 7: Kiểm tra Lambda Trigger

Trong trang chi tiết User Pool, tìm phần cấu hình Lambda Trigger. Tùy phiên bản giao diện AWS Console, mục này có thể nằm tại:

```text
User pool properties → Lambda triggers
```

hoặc:

```text
Extensions → Lambda triggers
```

Kiểm tra sự kiện **Post confirmation** đã được liên kết với đúng Lambda Function của dự án.

Các thông tin cần kiểm tra gồm:

* Loại Trigger là Post Confirmation.
* Tên Lambda Function.
* AWS Region của Lambda.
* User Pool đang sử dụng Trigger.
* Lambda Function có tồn tại và hoạt động hay không.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-post-confirmation-trigger.png"
    title="Hình 5.4.1.8: Lambda Post Confirmation được liên kết với Cognito User Pool"
    width="100%"
>}}

## Kết quả

Sau khi kiểm tra trực tiếp trên AWS Management Console, nhóm ghi nhận:

* Cognito User Pool đã được Terraform tạo thành công.
* App Client đã được cấu hình để frontend kết nối với Cognito.
* Hai Cognito Group `user` và `admin` đã được tạo để hỗ trợ phân quyền.
* Tài khoản người dùng được quản lý tập trung trong Cognito User Pool.
* Lambda Post Confirmation đã được liên kết với User Pool.
* Các IAM Role và IAM Policy cần thiết đã được tạo.
* Lambda Execution Role có Trust Relationship với `lambda.amazonaws.com`.
* Các Lambda Function được gán quyền phù hợp với nghiệp vụ.
* Phạm vi truy cập giữa các dịch vụ được giới hạn theo nguyên tắc quyền tối thiểu.
* Cấu hình Cognito và IAM đã sẵn sàng để tích hợp với frontend, API Gateway, Lambda và các dịch vụ tiếp theo.

Việc kiểm thử đăng ký, đăng nhập, phân quyền User/Admin và khả năng bảo vệ API được trình bày tại mục **5.5 Kiểm thử hệ thống**.