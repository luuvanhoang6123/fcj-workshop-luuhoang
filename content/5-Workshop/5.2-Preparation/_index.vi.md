---
title: "Các bước chuẩn bị môi trường"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

## Giới thiệu

Trước khi triển khai hệ thống **Live Auction** trên nền tảng **Amazon Web Services (AWS)**, nhóm cần chuẩn bị môi trường phát triển, tài khoản AWS và các công cụ hỗ trợ. Việc chuẩn bị thống nhất giúp quá trình phát triển ứng dụng, triển khai hạ tầng bằng **Terraform** và kiểm thử hệ thống diễn ra thuận lợi.

## Link mã nguồn dự án

Mã nguồn Live Auction được lưu trữ trên GitHub. Nhóm sử dụng nhánh `develop` để cập nhật và đồng bộ mã nguồn trong quá trình thực hiện.

Link GitHub: [GitHub Repository – Live Auction](https://github.com/CallmeSen/Live-Auction/tree/develop)

## Các công cụ cần chuẩn bị

- Tài khoản AWS.
- Git.
- AWS CLI.
- Terraform.
- Docker Desktop.
- Node.js.
- Python 3.
- Visual Studio Code hoặc IDE tương đương.

## Chuẩn bị mã nguồn

Clone mã nguồn của dự án từ nhánh `develop`:

```powershell
git clone -b develop https://github.com/CallmeSen/Live-Auction.git
cd Live-Auction
```

Sau khi clone thành công, cấu trúc chính của dự án bao gồm:

```text
backend/
frontend/
admin-frontend/
infra/
docker-compose.yml
```

Trong đó:

- `backend/`: Mã nguồn Backend sử dụng FastAPI.
- `frontend/`: Giao diện dành cho người dùng và thành viên.
- `admin-frontend/`: Giao diện dành cho quản trị viên.
- `infra/`: Mã nguồn Terraform dùng để triển khai hạ tầng AWS.
- `docker-compose.yml`: Cấu hình chạy các thành phần trong môi trường cục bộ.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/project-structure.png"
    title="Hình 5.2.1: Cấu trúc thư mục chính của dự án"
    width="60%"
>}}

## Tạo và cấu hình IAM User quản trị `la-admin`

Để quản lý môi trường AWS trong quá trình thực hiện workshop, nhóm tạo một IAM User có tên `la-admin`. Tài khoản này được sử dụng để đăng nhập AWS Management Console, tạo IAM User cho các thành viên và thực hiện các thao tác quản trị cần thiết trong phạm vi dự án.

Tài khoản Root chỉ được sử dụng cho các thiết lập cấp cao của AWS Account. Trong các công việc hằng ngày, nhóm sử dụng IAM User để hạn chế rủi ro bảo mật.

### Bước 1: Mở dịch vụ IAM

Đăng nhập AWS Management Console bằng tài khoản có quyền quản trị.

Tại thanh tìm kiếm dịch vụ, nhập:

```text
IAM
```

Sau đó mở dịch vụ:

```text
Identity and Access Management (IAM)
```

Trong menu bên trái, chọn:

```text
Access management → Users
```

### Bước 2: Tạo IAM User mới

Tại trang danh sách IAM User, chọn:

```text
Create user
```

Trong phần **User details**, nhập tên người dùng:

```text
la-admin
```

Chọn cấp quyền truy cập AWS Management Console cho tài khoản. Tài khoản `la-admin` cần có mật khẩu đăng nhập Console để quản lý các tài nguyên AWS và hỗ trợ tạo tài khoản cho các thành viên khác.

Khi thiết lập mật khẩu, có thể sử dụng mật khẩu do AWS tự tạo hoặc mật khẩu tùy chỉnh đáp ứng chính sách bảo mật. Người dùng cần thay đổi mật khẩu sau lần đăng nhập đầu tiên nếu AWS yêu cầu.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-create-user.jpg"
    title="Hình 5.2.2: Khai báo thông tin IAM User la-admin"
    width="90%"
>}}


### Bước 3: Cấp quyền cho `la-admin`

Sau khi khai báo tên người dùng, chuyển sang bước:

```text
Set permissions
```

Trong giai đoạn thực hiện workshop, tài khoản `la-admin` được gán các Policy sau:

- `AdministratorAccess`: Cho phép quản lý các tài nguyên AWS cần thiết trong môi trường workshop.
- `IAMUserChangePassword`: Cho phép tài khoản thay đổi mật khẩu đăng nhập của chính mình.

Sau khi kiểm tra lại tên User và các quyền được gán, chọn:

```text
Create user
```

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-permissions.png"
    title="Hình 5.2.3: Các Policy được gán cho IAM User la-admin"
    width="90%"
>}}

Quyền quản trị chỉ phù hợp với tài khoản quản lý trong môi trường học tập và thử nghiệm. Trong môi trường thực tế, cần áp dụng nguyên tắc **least privilege**, nghĩa là mỗi tài khoản chỉ được cấp đúng quyền cần thiết cho nhiệm vụ được giao.

### Bước 4: Lưu thông tin đăng nhập Console

Sau khi tạo thành công, AWS hiển thị thông tin đăng nhập AWS Management Console của IAM User `la-admin`.

Thông tin gồm:

- Console sign-in URL.
- User name.
- Mật khẩu đăng nhập ban đầu hoặc hướng dẫn tải tệp thông tin đăng nhập.

Nhóm lưu các thông tin này ở nơi an toàn để đăng nhập bằng tài khoản `la-admin`. Mật khẩu không được lưu trong GitHub, tệp mã nguồn hoặc chia sẻ công khai.

Không cần đưa ảnh hiển thị mật khẩu vào báo cáo. Nếu sử dụng ảnh xác nhận tạo User, cần che Console sign-in URL chứa Account ID và toàn bộ thông tin mật khẩu.

### Bước 5: Cấu hình MFA cho `la-admin`

Sau khi tạo User, mở:

```text
IAM → Users → la-admin → Security credentials
```

Tại phần **Multi-factor authentication (MFA)**, chọn:

```text
Assign MFA device
```

Nhóm chọn hình thức:

```text
Authenticator app
```

Sau đó sử dụng ứng dụng xác thực trên điện thoại để quét mã QR và nhập các mã xác thực được yêu cầu bởi AWS.

MFA giúp bảo vệ tài khoản vì khi đăng nhập, ngoài mật khẩu, người dùng phải cung cấp thêm mã xác thực từ thiết bị MFA.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-mfa-setup.jpg"
    title="Hình 5.2.4: Lựa chọn Authenticator app để cấu hình MFA cho la-admin"
    width="90%"
>}}

Sau khi xác thực thành công, kiểm tra lại trạng thái MFA tại tab **Security credentials**. Tài khoản cần hiển thị MFA device đã được gán.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-mfa-enabled.jpg"
    title="Hình 5.2.5: MFA đã được cấu hình cho IAM User la-admin"
    width="90%"
>}}

### Bước 6: Tạo Access Key cho AWS CLI và Terraform

Để sử dụng AWS CLI và Terraform trên máy tính cục bộ, nhóm tạo Access Key cho IAM User `la-admin`.

Mở:

```text
IAM → Users → la-admin → Security credentials
```

Tại phần **Access keys**, chọn:

```text
Create access key
```

Trong phần lựa chọn mục đích sử dụng, chọn:

```text
Command Line Interface (CLI)
```

Xác nhận đã đọc khuyến nghị bảo mật của AWS, sau đó chọn:

```text
Next
```

Có thể thêm Description tag để mô tả mục đích sử dụng Access Key, ví dụ:

```text
AWS CLI and Terraform for Live Auction workshop
```

Sau đó chọn:

```text
Create access key
```

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-access-key-purpose.jpg"
    title="Hình 5.2.6: Lựa chọn mục đích sử dụng Access Key cho AWS CLI"
    width="90%"
>}}

Sau khi Access Key được tạo, AWS chỉ hiển thị **Secret Access Key** một lần duy nhất. Nhóm lưu thông tin này trong môi trường cá nhân để cấu hình AWS CLI và không đưa lên GitHub.

### Bước 7: Kiểm tra IAM User sau khi cấu hình

Sau khi hoàn tất, mở lại:

```text
IAM → Users
```

Kiểm tra IAM User:

```text
la-admin
```

Tài khoản cần đáp ứng các điều kiện sau:

- Đã được tạo thành công.
- Có quyền phù hợp để quản lý môi trường workshop.
- Có thể đăng nhập AWS Management Console.
- Đã cấu hình MFA.
- Có Access Key phục vụ AWS CLI và Terraform nếu cần triển khai từ máy cục bộ.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-user-list.jpg"
    title="Hình 5.2.7: IAM User la-admin trong danh sách người dùng AWS"
    width="90%"
>}}

## Ước tính chi phí

Nhóm sử dụng AWS Billing Dashboard và Cost Explorer để theo dõi chi phí phát sinh trong quá trình triển khai hệ thống Live Auction. Biểu đồ dưới đây thể hiện chi phí được phân loại theo từng dịch vụ AWS.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/aws-cost-breakdown.jpg"
    title="Hình 5.2.8: Chi phí AWS được phân loại theo từng dịch vụ"
    width="100%"
>}}

Theo biểu đồ, các khoản chi phí chính phát sinh từ Amazon API Gateway và AWS Config. Amazon S3, Amazon DynamoDB và AWS Secrets Manager chỉ phát sinh chi phí nhỏ trong giai đoạn thử nghiệm.

Với quy mô workshop và số lượng người dùng thấp, chi phí dự kiến ở mức thấp. Các dịch vụ như AWS Lambda, Amazon DynamoDB on-demand, Amazon S3, Amazon SQS FIFO và Amazon Cognito có thể nằm trong hạn mức miễn phí hoặc chỉ phát sinh một khoản nhỏ.

Chi phí thực tế phụ thuộc vào:

- Thời gian tài nguyên hoạt động.
- Số lượng request đến API Gateway và Lambda.
- Dung lượng dữ liệu lưu trữ trên S3 và DynamoDB.
- Lưu lượng dữ liệu phân phối qua CloudFront.
- Số lượng message xử lý bởi SQS FIFO.
- Thời gian lưu trữ log trên CloudWatch.
- Số lượng tài nguyên được theo dõi bởi AWS Config.

### Trường hợp hệ thống được mở rộng

Nếu hệ thống được sử dụng với số lượng người dùng và lượt đặt giá tăng cao, chi phí có thể tăng theo các yếu tố sau:

- Amazon API Gateway tăng chi phí theo số lượng request.
- AWS Lambda tăng chi phí theo số lần gọi và thời gian thực thi.
- Amazon DynamoDB tăng chi phí theo lượng đọc, ghi và dung lượng lưu trữ.
- Amazon SQS FIFO tăng chi phí theo số lượng message.
- Amazon CloudFront tăng chi phí theo lưu lượng dữ liệu phân phối.
- Amazon S3 tăng chi phí theo dung lượng lưu trữ và số lần truy cập.
- CloudWatch tăng chi phí nếu lưu trữ nhiều log và metrics trong thời gian dài.
- AWS Config tăng chi phí khi theo dõi nhiều tài nguyên và lưu nhiều bản ghi cấu hình.

Để kiểm soát chi phí khi hệ thống mở rộng, nhóm có thể sử dụng billing alarm, giới hạn concurrency cho Lambda, thiết lập thời gian lưu log CloudWatch, áp dụng S3 Lifecycle Policy và xóa các tài nguyên không còn sử dụng.

Do đó, chi phí thực tế không chỉ phụ thuộc vào số lượng dịch vụ được triển khai mà còn phụ thuộc vào lưu lượng truy cập, số lượng giao dịch và thời gian duy trì tài nguyên AWS.

## Cài đặt và cấu hình AWS CLI

Sau khi chuẩn bị tài khoản IAM, nhóm kiểm tra phiên bản AWS CLI:

```powershell
aws --version
```

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/aws-version.png"
    title="Hình 5.2.9: Kiểm tra phiên bản AWS CLI"
    width="80%"
>}}

Tiếp theo, nhóm cấu hình thông tin tài khoản AWS:

```powershell
aws configure
```

Lệnh yêu cầu nhập:

- AWS Access Key ID.
- AWS Secret Access Key.
- Default Region, ví dụ `ap-southeast-1`.
- Default Output Format.

Thông tin xác thực chỉ được lưu trong môi trường cá nhân và không được đưa vào GitHub, hình ảnh hoặc mã nguồn báo cáo.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/aws-configure.png"
    title="Hình 5.2.10: Cấu hình AWS CLI bằng lệnh aws configure"
    width="80%"
>}}

## Cài đặt Terraform

Sau khi cài đặt Terraform, nhóm kiểm tra phiên bản bằng lệnh:

```powershell
terraform version
```

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/terraform-version.png"
    title="Hình 5.2.11: Kiểm tra phiên bản Terraform"
    width="60%"
>}}

Terraform được sử dụng để khởi tạo, kiểm tra, triển khai và quản lý hạ tầng AWS theo mô hình Infrastructure as Code.

## Kết quả

Sau khi hoàn thành các bước chuẩn bị, nhóm đã:

- Clone và quản lý mã nguồn dự án trên GitHub.
- Chuẩn bị môi trường Backend, User Frontend và Admin Frontend.
- Tạo IAM User `la-admin` để quản lý môi trường AWS.
- Cấu hình quyền truy cập và MFA cho tài khoản quản trị.
- Tạo Access Key phục vụ AWS CLI và Terraform.
- Cài đặt và cấu hình AWS CLI.
- Cài đặt Terraform để triển khai hạ tầng.
- Chuẩn bị Docker Desktop, Node.js, Python và Git phục vụ phát triển hệ thống.
- Theo dõi chi phí và các tài nguyên AWS đang hoạt động.

Trong phần tiếp theo, nhóm tiến hành khởi tạo môi trường Terraform, kiểm tra cấu hình và lập kế hoạch triển khai hạ tầng AWS cho hệ thống Live Auction.