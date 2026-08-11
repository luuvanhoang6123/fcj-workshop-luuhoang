---
title: "Khởi tạo môi trường Terraform"
date: 2026-07-13
weight: 3
chapter: false
pre: "<b>5.3.3. </b>"
---

## Giới thiệu

Sau khi hoàn thiện cấu trúc thư mục Infrastructure, nhóm tiến hành khởi tạo môi trường làm việc Terraform cho từng module bằng lệnh `terraform init`.

Đây là bước đầu tiên cần thực hiện trước khi sử dụng các lệnh như `terraform validate`, `terraform plan` hoặc `terraform apply`. Lệnh này giúp Terraform chuẩn bị thư mục làm việc, tải Provider cần thiết và thiết lập Backend dùng để lưu trữ trạng thái của hạ tầng.

Trong hệ thống Live Auction, các module Terraform được triển khai riêng biệt theo từng lớp chức năng. Vì vậy, lệnh `terraform init` cần được thực hiện tại thư mục của module tương ứng trước khi lập kế hoạch và triển khai tài nguyên.

---

## Kiểm tra Terraform

Mở PowerShell hoặc Terminal tại thư mục gốc của dự án và kiểm tra phiên bản Terraform:

```powershell
terraform version
```

Nếu Terraform đã được cài đặt thành công, Terminal sẽ hiển thị phiên bản Terraform và nền tảng đang sử dụng.

<!--
HƯỚNG DẪN CHỤP ẢNH:
1. Chạy lệnh: terraform version
2. Chụp cửa sổ Terminal có cả câu lệnh và kết quả.
3. Lưu ảnh tại:
static/images/5-Workshop/5.3-Infrastructure/terraform-version.png
-->

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-version.png" alt="Kiểm tra phiên bản Terraform" width="75%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.12.</b> Kiểm tra phiên bản Terraform trên môi trường triển khai.
    </figcaption>
</figure>

{{% notice info %}}
Phiên bản Terraform thực tế có thể khác tùy theo thời điểm cài đặt. Phiên bản đang sử dụng cần đáp ứng yêu cầu được khai báo trong tệp `versions.tf`.
{{% /notice %}}

---

## Kiểm tra kết nối với AWS

Trước khi khởi tạo Terraform, kiểm tra thông tin xác thực và khả năng kết nối đến AWS:

```powershell
aws sts get-caller-identity
```

Nếu AWS CLI đã được cấu hình chính xác, kết quả sẽ trả về các thông tin cơ bản gồm:

* User ID.
* AWS Account ID.
* ARN của IAM User hoặc IAM Role đang sử dụng.

Ví dụ:

```json
{                                                                                                                                              
    "UserId": "AIDATMVS2AD************",
    "Account": "2333********",
    "Arn": "arn:aws:iam::23********:user/la-frontend-1"
}

```

{{% notice warning %}}
**Lưu ý bảo mật:** Không đưa **Access Key**, **Secret Access Key** hoặc **Session Token** vào hình ảnh, mã nguồn hay repository báo cáo. Đây là các thông tin xác thực có thể được sử dụng để truy cập tài khoản AWS, gọi dịch vụ và tạo hoặc thay đổi tài nguyên trong phạm vi quyền đã cấp. Nếu bị công khai, các thông tin này có thể dẫn đến truy cập trái phép, mất dữ liệu hoặc phát sinh chi phí AWS ngoài ý muốn. Kết quả của lệnh `aws sts get-caller-identity` không hiển thị Secret Access Key nhưng có chứa **AWS Account ID**, **User ID** và **ARN**. Các thông tin này không phải mật khẩu, tuy nhiên vẫn có thể làm lộ danh tính tài khoản, tên IAM User hoặc IAM Role và cấu trúc tài nguyên của hệ thống. Vì vậy, nên che một phần Account ID, User ID và ARN trước khi đưa ảnh chụp vào báo cáo hoặc repository công khai.
{{% /notice %}}

---

## Di chuyển đến module cần khởi tạo

Từ thư mục gốc của dự án, di chuyển đến thư mục Infrastructure:

```powershell
cd infra
```

Trong lần khởi tạo đầu tiên, nhóm bắt đầu với module `03-identity`:

```powershell
cd 03-identity
```

Module `03-identity` chứa các tệp cấu hình Terraform được sử dụng để triển khai lớp xác thực và phân quyền của hệ thống. Các tệp trong module khai báo Terraform Backend, AWS Provider, biến đầu vào, giá trị đầu ra và các tài nguyên liên quan đến Amazon Cognito và AWS IAM.

Có thể kiểm tra vị trí thư mục hiện tại bằng lệnh:

```powershell
Get-Location
```

Kết quả cần cho thấy Terminal đang làm việc tại thư mục:

```text
D:\ThucTap\Live-Auction\infra\03-identity
```

Sau khi di chuyển đúng vào module, nhóm tiếp tục thực hiện bước khởi tạo Terraform Backend và môi trường làm việc.

---

## Kiểm tra kết quả khởi tạo

Sau khi lệnh `terraform init` hoàn tất, nhóm sử dụng lệnh sau để kiểm tra các tệp trong module:

```powershell
Get-ChildItem -Force
```

Kết quả cho thấy các tệp cấu hình Terraform ban đầu vẫn được giữ nguyên, đồng thời Terraform đã tạo thêm thư mục `.terraform/` và tệp `.terraform.lock.hcl`.

Các thành phần chính trong thư mục bao gồm:

* `backend.tf`: Khai báo Remote Backend dùng để lưu trữ Terraform State.
* `main.tf`: Khai báo các tài nguyên chính của module Identity.
* `outputs.tf`: Khai báo các giá trị đầu ra của module.
* `providers.tf`: Cấu hình AWS Provider.
* `variables.tf`: Khai báo các biến đầu vào.
* `versions.tf`: Khai báo phiên bản Terraform và Provider được yêu cầu.
* `.terraform/`: Chứa Provider, module phụ thuộc và thông tin Backend được Terraform sử dụng.
* `.terraform.lock.hcl`: Khóa phiên bản Provider đã được Terraform lựa chọn.
* `tfplan`: Tệp kế hoạch triển khai được tạo trong quá trình thực hiện `terraform plan` trước đó.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-init-files.png" alt="Các tệp trong module Identity sau khi Terraform Init" width="80%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.14.</b> Các tệp trong module <code>03-identity</code> sau khi khởi tạo Terraform.
    </figcaption>
</figure>

{{% notice info %}}
Tệp `tfplan` không được tạo bởi lệnh `terraform init`. Đây là tệp kế hoạch triển khai đã được tạo trong quá trình thực hiện `terraform plan` trước đó. Tệp này xuất hiện trong ảnh do nhóm kiểm tra lại thư mục sau khi hệ thống đã hoàn tất quá trình triển khai.
{{% /notice %}}

Kết quả trên cho thấy:

* Terraform đã nhận diện thành công các tệp cấu hình của module.
* AWS Provider đã được tải về máy.
* Remote Backend đã được kết nối.
* Phiên bản Provider đã được khóa trong `.terraform.lock.hcl`.
* Module `03-identity` đã sẵn sàng cho các bước `terraform validate`, `terraform plan` và `terraform apply`.

Việc chạy lại `terraform init` không tạo mới, cập nhật hoặc xóa các tài nguyên AWS đang hoạt động. Lệnh này chỉ chuẩn bị lại môi trường làm việc Terraform và kết nối đến Remote Backend đã được cấu hình.

---

## Khởi tạo Terraform Backend

Để quản lý trạng thái của hạ tầng sau khi triển khai, dự án sử dụng **Remote Backend** thay vì lưu Terraform State trực tiếp trên máy của từng thành viên. Cấu hình Backend được khai báo trong tệp `backend.tf` của từng module Terraform.

Trong hệ thống Live Auction, Terraform State được lưu trữ trên **Amazon S3**. Mỗi module sử dụng một `key` riêng để tách biệt tệp trạng thái, giúp việc quản lý các thành phần hạ tầng trở nên rõ ràng và hạn chế ảnh hưởng lẫn nhau giữa các module.

Ví dụ, cấu hình Backend của module **07-api** như sau:

```hcl
terraform {
  backend "s3" {
    bucket         = "la-tfstate-233376973052"
    key            = "07-api/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "la-tflock"
    encrypt        = true
  }
}
```

Các thuộc tính trong cấu hình có ý nghĩa như sau:

| Thuộc tính       | Chức năng                                                               |
| ---------------- | ----------------------------------------------------------------------- |
| `bucket`         | Tên Amazon S3 Bucket được sử dụng để lưu trữ Terraform State.           |
| `key`            | Đường dẫn của tệp Terraform State tương ứng với module đang triển khai. |
| `region`         | AWS Region chứa S3 Bucket dùng làm Terraform Backend.                   |
| `dynamodb_table` | Tên bảng Amazon DynamoDB được cấu hình để hỗ trợ cơ chế khóa State.     |
| `encrypt`        | Cho phép mã hóa Terraform State khi được lưu trữ trên Amazon S3.        |

Trong cấu hình trên, Terraform State của module **07-api** được lưu tại `07-api/terraform.tfstate` trong S3 Bucket `la-tfstate-233376973052` tại Region `ap-southeast-1`.

Việc sử dụng Remote Backend giúp Terraform State không phụ thuộc vào máy tính của một thành viên cụ thể. Nhờ đó, các thành viên trong nhóm có thể làm việc trên cùng trạng thái hạ tầng và giảm nguy cơ xảy ra sai lệch giữa các môi trường triển khai.

{{% notice note %}}
Mỗi module Terraform có thể sử dụng một `key` khác nhau trong cùng S3 Bucket để quản lý Terraform State riêng cho từng thành phần hạ tầng.
{{% /notice %}}

Nếu tài nguyên Remote Backend chưa được tạo, di chuyển đến thư mục `00-bootstrap`:

```powershell
cd ..\00-bootstrap
```

Chạy tập lệnh bootstrap:

```powershell
.\bootstrap-remote-state.ps1
```

Tập lệnh này chuẩn bị các tài nguyên cần thiết để lưu trữ và quản lý Terraform State từ xa.

Sau khi hoàn thành, quay lại module Identity:

```powershell
cd ..\03-identity
```

---

## Thực hiện Terraform Init

Tại thư mục `03-identity`, chạy lệnh:

```powershell
terraform init
```

Khi thực thi, Terraform sẽ:

1. Đọc các tệp cấu hình trong thư mục hiện tại.
2. Khởi tạo Terraform Backend.
3. Kết nối đến nơi lưu trữ Terraform State.
4. Tải AWS Provider theo phiên bản được khai báo.
5. Khởi tạo các module phụ thuộc nếu có.
6. Tạo thư mục `.terraform/`.
7. Tạo hoặc cập nhật tệp `.terraform.lock.hcl`.

Khi quá trình khởi tạo thành công, Terminal hiển thị thông báo:

```text
Terraform has been successfully initialized!
```

<!--
HƯỚNG DẪN CHỤP ẢNH:
1. Mở Terminal tại infra/03-identity.
2. Chạy lệnh: terraform init
3. Chụp phần kết quả có dòng:
   Terraform has been successfully initialized!
4. Lưu ảnh tại:
static/images/5-Workshop/5.3-Infrastructure/terraform-init-success.png
-->

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-init-success.png" alt="Terraform Init thành công" width="85%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.13.</b> Khởi tạo module Terraform thành công bằng lệnh <code>terraform init</code>.
    </figcaption>
</figure>

---

## Khởi tạo lại Backend

Khi nội dung trong tệp `backend.tf` thay đổi, Terraform có thể yêu cầu khởi tạo lại Backend.

Sử dụng lệnh:

```powershell
terraform init -reconfigure
```

Tùy chọn `-reconfigure` yêu cầu Terraform bỏ qua cấu hình Backend đã lưu trước đó và đọc lại cấu hình hiện tại.

Nếu cần chuyển Terraform State từ Backend cũ sang Backend mới, sử dụng:

```powershell
terraform init -migrate-state
```

{{% notice warning %}}
Chỉ sử dụng `-migrate-state` khi cần chuyển Terraform State. Nên kiểm tra và sao lưu State trước khi thực hiện để tránh ảnh hưởng đến trạng thái hạ tầng.
{{% /notice %}}

---

## Kiểm tra kết quả khởi tạo

Sau khi `terraform init` hoàn tất, kiểm tra lại nội dung thư mục:

```powershell
Get-ChildItem -Force
```

Terraform sẽ tạo thêm:

```text
.terraform/
.terraform.lock.hcl
```

Trong đó:

* `.terraform/` chứa Provider, module và thông tin Backend được Terraform sử dụng.
* `.terraform.lock.hcl` khóa phiên bản Provider đã được lựa chọn.
* Các tệp `.tf` ban đầu vẫn được giữ nguyên.
* Module đã sẵn sàng cho các bước kiểm tra và triển khai tiếp theo.

<!--
HƯỚNG DẪN CHỤP ẢNH:
1. Trong VS Code Explorer, mở thư mục infra/03-identity.
2. Chụp cấu trúc có:
   .terraform/
   .terraform.lock.hcl
   backend.tf
   main.tf
   outputs.tf
   providers.tf
   variables.tf
   versions.tf
3. Lưu ảnh tại:
static/images/5-Workshop/5.3-Infrastructure/terraform-init-files.png
-->

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-init-files.png" alt="Các tệp sau Terraform Init" width="65%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.14.</b> Thư mục <code>.terraform</code> và tệp <code>.terraform.lock.hcl</code> sau khi khởi tạo.
    </figcaption>
</figure>

---

## Khởi tạo các module còn lại

Sau khi khởi tạo module Identity, thực hiện tương tự với các module còn lại.

Khởi tạo module Data:

```powershell
cd ..\04-data
terraform init
```

Khởi tạo module Messaging:

```powershell
cd ..\05-messaging
terraform init
```

Khởi tạo module Compute:

```powershell
cd ..\06-compute
terraform init
```

Khởi tạo module API:

```powershell
cd ..\07-api
terraform init
```

Khởi tạo module Edge:

```powershell
cd ..\09-edge
terraform init
```

Mỗi module có cấu hình Backend và Terraform State riêng. Việc phân chia State theo module giúp nhóm kiểm soát phạm vi thay đổi và hạn chế ảnh hưởng giữa các lớp hạ tầng.

Thứ tự thực hiện của nhóm gồm:

1. Identity.
2. Data.
3. Messaging.
4. Compute.
5. API.
6. Edge.

Lệnh `terraform init` chỉ chuẩn bị môi trường làm việc và chưa tạo tài nguyên AWS. Các tài nguyên chỉ được tạo khi thực hiện `terraform apply`.

---

## Một số lỗi thường gặp

### Terraform không được nhận diện

Thông báo lỗi:

```text
terraform: The term 'terraform' is not recognized
```

Nguyên nhân có thể do Terraform chưa được cài đặt hoặc chưa được thêm vào biến môi trường `PATH`.

Kiểm tra lại bằng lệnh:

```powershell
terraform version
```

### Không tìm thấy thông tin xác thực AWS

Thông báo lỗi:

```text
No valid credential sources found
```

Kiểm tra cấu hình AWS CLI:

```powershell
aws configure list
```

Sau đó kiểm tra kết nối:

```powershell
aws sts get-caller-identity
```

### Không tìm thấy S3 Backend

Thông báo có thể xuất hiện:

```text
Failed to get existing workspaces
```

Cần kiểm tra:

* S3 Bucket lưu Terraform State đã được tạo chưa.
* Tên Bucket trong `backend.tf` có chính xác không.
* AWS Region có đúng không.
* IAM User hoặc IAM Role có quyền truy cập S3 không.

### Không đủ quyền truy cập

Nếu xuất hiện lỗi `AccessDenied`, cần kiểm tra IAM Policy của tài khoản đang sử dụng. Tài khoản cần có quyền truy cập Terraform Backend và các quyền cần thiết để quản lý tài nguyên trong module.

---

## Kết quả

Sau khi hoàn tất quá trình khởi tạo:

* Terraform đã nhận diện các tệp cấu hình trong từng module.
* AWS Provider đã được tải theo phiên bản yêu cầu.
* Terraform Backend đã được kết nối thành công.
* Thư mục `.terraform/` và tệp `.terraform.lock.hcl` đã được tạo.
* Các module đã sẵn sàng cho bước kiểm tra cấu hình.
* Nhóm có thể tiếp tục lập kế hoạch triển khai bằng lệnh `terraform plan`.