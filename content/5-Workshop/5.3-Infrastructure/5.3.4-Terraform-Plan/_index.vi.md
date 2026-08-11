---
title: "Kiểm tra kế hoạch triển khai"
date: 2026-07-13
weight: 4
chapter: false
pre: "<b>5.3.4. </b>"
---

## Giới thiệu

Sau khi hoàn tất quá trình khởi tạo môi trường Terraform, nhóm tiến hành kiểm tra cấu hình và lập kế hoạch triển khai bằng các lệnh `terraform fmt -check`, `terraform validate` và `terraform plan`.

Lệnh `terraform plan` đọc mã nguồn Terraform, trạng thái hạ tầng được lưu trong Remote Backend và trạng thái thực tế của các tài nguyên trên AWS. Terraform sau đó so sánh trạng thái hiện tại với cấu hình mong muốn để xác định những tài nguyên cần được tạo mới, cập nhật, thay thế hoặc xóa.

Việc kiểm tra kế hoạch trước khi triển khai giúp nhóm:

* Phát hiện lỗi cú pháp và cấu hình không hợp lệ.
* Kiểm tra các thay đổi dự kiến trước khi tác động đến AWS.
* Phát hiện tài nguyên có nguy cơ bị xóa hoặc thay thế.
* Hạn chế thay đổi ngoài ý muốn đối với hệ thống đang hoạt động.
* Xác nhận Terraform State đang đồng bộ với hạ tầng thực tế.
* Kiểm tra lại hạ tầng sau khi quá trình triển khai hoàn tất.

{{% notice info %}}
Các lệnh `terraform validate` và `terraform plan` không trực tiếp tạo, cập nhật hoặc xóa tài nguyên AWS. Tài nguyên chỉ bị thay đổi khi thực hiện `terraform apply` và xác nhận kế hoạch triển khai.
{{% /notice %}}

---

## Di chuyển đến module cần kiểm tra

Trong nội dung này, nhóm sử dụng module `03-identity` làm ví dụ đại diện cho quá trình kiểm tra cấu hình và kế hoạch triển khai.

Mở PowerShell tại thư mục gốc của dự án:

```powershell
cd "D:\ThucTap\Live-Auction"
```

Di chuyển đến module Identity:

```powershell
cd infra\03-identity
```

Kiểm tra vị trí hiện tại:

```powershell
Get-Location
```

Kết quả cần cho thấy Terminal đang làm việc tại:

```text
D:\ThucTap\Live-Auction\infra\03-identity
```

Nếu module chưa được khởi tạo trên máy hiện tại, chạy:

```powershell
terraform init
```

---

## Kiểm tra định dạng mã Terraform

Trước khi kiểm tra cấu hình và tạo kế hoạch, nhóm sử dụng lệnh sau để kiểm tra định dạng của các tệp Terraform:

```powershell
terraform fmt -check
```

Lệnh `terraform fmt -check` chỉ kiểm tra định dạng và không tự động chỉnh sửa các tệp `.tf`.

Nếu các tệp đã đúng định dạng, lệnh hoàn tất mà không hiển thị lỗi. Nếu Terminal trả về tên của một hoặc nhiều tệp, các tệp đó chưa tuân theo định dạng chuẩn của Terraform.

Trong quá trình phát triển, có thể sử dụng lệnh sau để tự động chuẩn hóa định dạng:

```powershell
terraform fmt
```

Tuy nhiên, không nên chạy `terraform fmt` chỉ để chụp ảnh nếu không có ý định cập nhật mã nguồn, vì lệnh này có thể làm thay đổi nội dung của các tệp đang được Git quản lý.

Có thể kiểm tra lại trạng thái Git sau khi thực hiện:

```powershell
git status
```

---

## Kiểm tra cấu hình Terraform

Sau khi kiểm tra định dạng, chạy:

```powershell
terraform validate
```

Lệnh `terraform validate` kiểm tra:

* Cú pháp của các tệp `.tf`.
* Tên và kiểu dữ liệu của biến.
* Các thuộc tính bắt buộc của tài nguyên.
* Cách tham chiếu giữa các tài nguyên.
* Cấu hình Provider và module.
* Tính nhất quán của toàn bộ cấu hình Terraform.

Khi cấu hình hợp lệ, Terraform hiển thị:

```text
Success! The configuration is valid.
```

{{% notice warning %}}
Lệnh `terraform validate` chỉ kiểm tra cú pháp và tính nhất quán nội bộ của cấu hình. Kết quả thành công không khẳng định tài khoản AWS đang sử dụng có đủ quyền để truy cập Backend hoặc quản lý các tài nguyên được khai báo.
{{% /notice %}}

---

## Tạo kế hoạch triển khai

Sau khi cấu hình được xác nhận hợp lệ, chạy:

```powershell
terraform plan -no-color
```

Tùy chọn `-no-color` loại bỏ mã màu khỏi kết quả, giúp nội dung trên Terminal dễ đọc và thuận tiện hơn khi chụp ảnh cho báo cáo.

Khi thực hiện, Terraform sẽ:

1. Đọc các tệp cấu hình trong module.
2. Kết nối đến Remote Backend.
3. Đọc Terraform State hiện tại.
4. Truy vấn trạng thái tài nguyên trên AWS.
5. So sánh hạ tầng thực tế với cấu hình mong muốn.
6. Xác định các tài nguyên cần tạo, cập nhật, thay thế hoặc xóa.
7. Hiển thị bản kế hoạch để nhóm kiểm tra trước khi triển khai.

Terraform sử dụng các ký hiệu sau trong kết quả kế hoạch:

| Ký hiệu | Ý nghĩa                              |
| ------- | ------------------------------------ |
| `+`     | Tài nguyên sẽ được tạo mới.          |
| `~`     | Tài nguyên sẽ được cập nhật tại chỗ. |
| `-`     | Tài nguyên sẽ bị xóa.                |
| `-/+`   | Tài nguyên sẽ bị xóa và tạo lại.     |
| `<=`    | Dữ liệu sẽ được đọc từ Data Source.  |

Ví dụ, trước lần triển khai đầu tiên, Terraform có thể hiển thị:

```text
Plan: 3 to add, 0 to change, 0 to destroy.
```

Trong đó:

* `3 to add`: có ba tài nguyên sẽ được tạo.
* `0 to change`: không có tài nguyên cần cập nhật.
* `0 to destroy`: không có tài nguyên bị xóa.

Số lượng tài nguyên thực tế phụ thuộc vào mã nguồn và trạng thái hạ tầng tại thời điểm chạy lệnh. Vì vậy, không sử dụng số lượng trong ví dụ trên làm kết quả chính thức của hệ thống.

---

## Kiểm tra kế hoạch sau khi hệ thống đã triển khai

Hệ thống Live Auction đã được triển khai trên AWS. Vì vậy, khi chạy lại `terraform plan` với mã nguồn và Terraform State mới nhất, kết quả mong đợi là:

```text
No changes. Your infrastructure matches the configuration.
```

Thông báo này cho biết:

* Terraform State phù hợp với cấu hình hiện tại.
* Các tài nguyên trên AWS phù hợp với trạng thái mong muốn.
* Không có tài nguyên cần tạo mới.
* Không có tài nguyên cần cập nhật.
* Không có tài nguyên cần xóa hoặc thay thế.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-plan-no-changes.png" alt="Terraform Plan không phát hiện thay đổi" width="80%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.16.</b> Terraform xác nhận hạ tầng hiện tại phù hợp với cấu hình của module Identity.
    </figcaption>
</figure>

{{% notice warning %}}
Nếu kết quả xuất hiện `to add`, `to change`, `to destroy` hoặc ký hiệu `-/+`, không tiếp tục chạy `terraform apply` ngay. Cần kiểm tra mã nguồn, Terraform State và tài nguyên thực tế để xác định nguyên nhân của thay đổi.
{{% /notice %}}

---

## Lưu kế hoạch triển khai

Trong quá trình triển khai ban đầu, nhóm có thể lưu kế hoạch vào tệp `tfplan` bằng lệnh:

```powershell
terraform plan -out="tfplan"
```

Tùy chọn `-out` lưu chính xác kế hoạch tại thời điểm kiểm tra. Tệp này có thể được sử dụng ở bước triển khai:

```powershell
terraform apply "tfplan"
```

Để xem lại kế hoạch đã lưu:

```powershell
terraform show tfplan
```

Tệp `tfplan` đang xuất hiện trong module `03-identity` là tệp được tạo từ quá trình lập kế hoạch trước đó. Tệp này không được tạo bởi `terraform init`.

{{% notice warning %}}
Tệp `tfplan` ở dạng nhị phân và có thể chứa tên tài nguyên, ARN, cấu hình hạ tầng hoặc các giá trị nhạy cảm. Không chỉnh sửa thủ công và không đẩy tệp này lên repository công khai. Kế hoạch đã lưu cũng có thể không còn phù hợp nếu mã nguồn hoặc trạng thái hạ tầng đã thay đổi.
{{% /notice %}}

Do hệ thống đã được triển khai, nhóm không cần tạo lại tệp `tfplan` chỉ để chụp ảnh. Kết quả `terraform plan -no-color` báo không có thay đổi là đủ để chứng minh mã nguồn và hạ tầng hiện tại đang đồng bộ.

---

## Các module Terraform của hệ thống

Hạ tầng hiện tại được chia thành các module sau:

| Module             | Vai trò                                                                                              |
| ------------------ | ---------------------------------------------------------------------------------------------------- |
| `00-bootstrap`     | Khởi tạo S3 Backend và cơ chế quản lý Terraform State.                                               |
| `01-foundation`    | Chuẩn bị các thành phần nền tảng dùng chung của hệ thống.                                            |
| `03-identity`      | Triển khai Cognito User Pool, App Client, User Group, IAM và Lambda Post Confirmation.               |
| `04-data`          | Triển khai các bảng DynamoDB và S3 Bucket lưu trữ dữ liệu media.                                     |
| `05-messaging`     | Triển khai SQS FIFO, Dead-letter Queue và các thành phần EventBridge Scheduler.                      |
| `06-compute`       | Triển khai Lambda Function, Lambda Layer, IAM Role và Event Source Mapping.                          |
| `07-api`           | Triển khai REST API, WebSocket API, Authorizer, Route, Integration và Stage.                         |
| `09-edge`          | Triển khai S3 Bucket và CloudFront Distribution cho User Frontend, Admin Frontend và nội dung media. |
| `10-observability` | Cấu hình giám sát, log, cảnh báo và khả năng quan sát hệ thống.                                      |
| `11-security`      | Bổ sung các cấu hình và biện pháp bảo mật cho hạ tầng.                                               |
| `12-backup-dr`     | Cấu hình sao lưu và hỗ trợ khôi phục khi xảy ra sự cố.                                               |
| `13-cicd`          | Triển khai các tài nguyên phục vụ quy trình tích hợp và triển khai liên tục.                         |

Mỗi module có Remote Backend và Terraform State riêng. Việc chia State theo module giúp nhóm:

* Giới hạn phạm vi thay đổi.
* Giảm ảnh hưởng giữa các lớp hạ tầng.
* Dễ dàng kiểm tra kế hoạch của từng thành phần.
* Hỗ trợ triển khai theo thứ tự phụ thuộc.
* Giảm nguy cơ nhiều thành viên cùng thay đổi một State.

---

## Nội dung cần kiểm tra trong kế hoạch

Trước khi chấp nhận một kế hoạch triển khai, nhóm kiểm tra các nội dung sau.

### Tên tài nguyên

Tên S3 Bucket, DynamoDB Table, Lambda Function, API Gateway, SQS Queue và các tài nguyên khác phải tuân theo quy ước đặt tên của dự án.

Các tài nguyên chính của hệ thống sử dụng tiền tố:

```text
la-
```

### AWS Region

Các tài nguyên theo Region được triển khai tại:

```text
ap-southeast-1
```

Region này tương ứng với khu vực **Asia Pacific (Singapore)**.

Một số dịch vụ toàn cầu như AWS IAM và Amazon CloudFront không được quản lý hoàn toàn theo phạm vi Region giống các dịch vụ như Lambda hoặc DynamoDB.

### Quyền IAM

IAM Policy cần tuân theo nguyên tắc cấp quyền tối thiểu, chỉ cho phép mỗi dịch vụ truy cập các tài nguyên cần thiết để thực hiện nghiệp vụ.

### Tài nguyên bị xóa hoặc thay thế

Nếu kế hoạch xuất hiện:

```text
Plan: 0 to add, 0 to change, 1 to destroy.
```

hoặc ký hiệu:

```text
-/+
```

nhóm phải kiểm tra kỹ trước khi tiếp tục. Việc xóa hoặc thay thế tài nguyên có thể:

* Làm gián đoạn hệ thống.
* Thay đổi Endpoint hoặc ARN.
* Làm mất dữ liệu nếu tài nguyên không được bảo vệ.
* Ảnh hưởng đến những module phụ thuộc.
* Làm frontend mất kết nối với backend.

### Giá trị đầu ra

Các Output cần được kiểm tra bao gồm:

* Cognito User Pool ID.
* Cognito App Client ID.
* Tên các bảng DynamoDB.
* SQS Queue URL.
* REST API Endpoint.
* WebSocket Endpoint.
* S3 Bucket name.
* CloudFront domain name.
* Tên Lambda Function.
* ARN của các tài nguyên được module khác sử dụng.

Không đưa toàn bộ Account ID, ARN hoặc giá trị xác thực nhạy cảm vào ảnh chụp của báo cáo công khai.

---

## Một số lỗi thường gặp

### Terraform chưa được khởi tạo

Thông báo:

```text
Backend initialization required
```

Khởi tạo module bằng:

```powershell
terraform init
```

Nếu cấu hình Backend vừa được thay đổi và đã được nhóm xác nhận:

```powershell
terraform init -reconfigure
```

### Không đủ quyền truy cập AWS

Thông báo có thể xuất hiện:

```text
AccessDenied
```

Kiểm tra danh tính AWS CLI:

```powershell
aws sts get-caller-identity
```

Sau đó kiểm tra IAM Policy của tài khoản hoặc Role đang sử dụng.

### Không truy cập được Remote Backend

Cần kiểm tra:

* S3 Bucket lưu Terraform State còn tồn tại hay không.
* Tên Bucket trong `backend.tf` có chính xác không.
* AWS Region có phù hợp không.
* Tài khoản có quyền đọc State trong S3 không.
* Bảng DynamoDB dùng để khóa State có hoạt động không.

### Terraform State đang bị khóa

Khi một tiến trình khác đang sử dụng State, Terraform có thể trả về lỗi khóa.

Không tự ý xóa hoặc mở khóa State khi chưa xác nhận tiến trình trước đã kết thúc. Việc nhiều thành viên đồng thời thao tác trên cùng một module có thể gây xung đột và làm sai lệch trạng thái hạ tầng.

### Kế hoạch xuất hiện thay đổi ngoài dự kiến

Nếu hệ thống đã triển khai nhưng `terraform plan` vẫn phát hiện thay đổi, cần kiểm tra:

* Code local có đúng commit mới nhất không.
* Đang đứng đúng module hay không.
* AWS credentials có trỏ đúng tài khoản không.
* Backend có trỏ đúng State hay không.
* Tài nguyên có bị chỉnh sửa thủ công trên AWS Console không.
* File build hoặc gói triển khai Lambda có thay đổi không.
* Biến môi trường và cấu hình Provider có phù hợp không.

Không thực hiện `terraform apply` cho đến khi xác định được nguyên nhân.

---

## Kết quả

Sau khi hoàn tất bước kiểm tra kế hoạch:

* Định dạng của các tệp Terraform đã được kiểm tra bằng `terraform fmt -check`.
* Cấu hình module Identity đã được xác nhận hợp lệ bằng `terraform validate`.
* Terraform đã kết nối thành công với Remote Backend.
* Terraform đã đọc trạng thái hạ tầng hiện tại từ Terraform State và AWS.
* Lệnh `terraform plan` đã so sánh mã nguồn với các tài nguyên đang hoạt động.
* Kết quả không phát hiện tài nguyên cần tạo mới, cập nhật, thay thế hoặc xóa.
* Terraform State và hạ tầng hiện tại đang đồng bộ.
* Quá trình kiểm tra không làm thay đổi các tài nguyên AWS đã triển khai.
* Hệ thống sẵn sàng để tiếp tục kiểm tra quy trình triển khai trong mục tiếp theo.