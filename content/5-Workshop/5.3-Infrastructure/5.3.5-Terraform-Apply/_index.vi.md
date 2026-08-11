---
title: "Triển khai hạ tầng"
date: 2026-07-13
weight: 5
chapter: false
pre: "<b>5.3.5. </b>"
---

## Giới thiệu

Sau khi kiểm tra và xác nhận kế hoạch triển khai, nhóm sử dụng lệnh `terraform apply` để tạo và cấu hình các tài nguyên trên Amazon Web Services.

Lệnh `terraform apply` áp dụng những thay đổi đã được Terraform xác định trong kế hoạch, bao gồm:

* Tạo mới tài nguyên.
* Cập nhật tài nguyên hiện có.
* Thay thế tài nguyên khi cấu hình bắt buộc phải tạo lại.
* Xóa tài nguyên không còn được khai báo.
* Cập nhật Terraform State sau khi triển khai thành công.

Trong hệ thống Live Auction, hạ tầng được chia thành nhiều module độc lập. Mỗi module quản lý một nhóm tài nguyên và sử dụng một Terraform State riêng trong Remote Backend.

Việc phân chia này giúp nhóm:

* Triển khai hạ tầng theo từng lớp chức năng.
* Giới hạn phạm vi thay đổi của mỗi lần triển khai.
* Giảm ảnh hưởng giữa các thành phần.
* Dễ dàng kiểm tra và xử lý lỗi.
* Cho phép các module phía sau sử dụng Output của module đã triển khai trước.

{{% notice warning %}}
Môi trường Live Auction hiện đã được triển khai và đang hoạt động trên AWS. Không chạy lại `terraform apply` chỉ để tạo ảnh minh họa. Việc apply lại một kế hoạch chưa được kiểm tra có thể cập nhật, thay thế hoặc xóa tài nguyên đang được hệ thống sử dụng.
{{% /notice %}}

---

## Chuẩn bị trước khi triển khai

Trước khi thực hiện `terraform apply`, nhóm cần bảo đảm:

* AWS CLI đã được cấu hình đúng tài khoản.
* Đang sử dụng đúng AWS Region.
* Module đã được chạy `terraform init`.
* Cấu hình đã vượt qua `terraform validate`.
* Kế hoạch đã được kiểm tra bằng `terraform plan`.
* Không có tài nguyên bị xóa hoặc thay thế ngoài dự kiến.
* Lambda package và Lambda Layer đã được build đầy đủ.
* Terraform State không bị một tiến trình khác khóa.
* Các module phụ thuộc đã được triển khai trước.

Kiểm tra danh tính AWS đang sử dụng:

```powershell
aws sts get-caller-identity
```

Kiểm tra Region:

```powershell
aws configure get region
```

Region chính được sử dụng trong dự án là:

```text
ap-southeast-1
```

{{% notice warning %}}
Kết quả của `aws sts get-caller-identity` có thể chứa AWS Account ID, User ID và ARN. Cần che các thông tin định danh này trước khi đưa ảnh chụp vào báo cáo hoặc repository công khai.
{{% /notice %}}

---

## Chuẩn bị gói triển khai Lambda

Một số module sử dụng tệp ZIP của Lambda Function hoặc Lambda Layer để tính `source_code_hash`.

Các tệp build không được lưu trực tiếp trong Git repository. Vì vậy, cần build chúng trên máy triển khai trước khi thực hiện `terraform plan` hoặc `terraform apply`.

Ví dụ, để build Lambda Post Confirmation của module Identity:

```powershell
cd "D:\ThucTap\Live-Auction"
.\backend\build.ps1 -Target function -FunctionName cognito_post_confirm
```

Kiểm tra tệp đã được tạo:

```powershell
Test-Path .\backend\build\cognito_post_confirm.zip
```

Nếu quá trình build thành công, kết quả trả về:

```text
True
```

Các Lambda Function và Lambda Layer khác cần được build tương ứng trước khi triển khai module Compute.

{{% notice info %}}
Script `backend/build.ps1` sử dụng Docker để tạo các gói Lambda theo môi trường runtime phù hợp. Docker Desktop phải được khởi động trước khi chạy script.
{{% /notice %}}

---

## Kiểm tra kế hoạch trước khi Apply

Nếu kế hoạch đã được lưu bằng:

```powershell
terraform plan -out="tfplan"
```

nhóm xem lại nội dung bằng:

```powershell
terraform show tfplan
```

Các nội dung cần kiểm tra gồm:

* Số lượng tài nguyên được tạo mới.
* Các tài nguyên được cập nhật.
* Các tài nguyên bị xóa.
* Các tài nguyên bị thay thế.
* Tên và AWS Region của tài nguyên.
* IAM Role và IAM Policy.
* Giá trị Output dự kiến.
* Thay đổi đối với Lambda Function.
* Thay đổi đối với DynamoDB Table và S3 Bucket.

{{% notice warning %}}
Không tiếp tục triển khai nếu kế hoạch xuất hiện tài nguyên bị xóa hoặc thay thế ngoài dự kiến. Cần dừng lại, kiểm tra cấu hình, Terraform State và chạy lại `terraform plan`.
{{% /notice %}}

---

## Thực hiện Terraform Apply

Nếu kế hoạch đã được lưu vào tệp `tfplan`, áp dụng chính xác kế hoạch đó bằng:

```powershell
terraform apply "tfplan"
```

Khi sử dụng tệp kế hoạch đã lưu, Terraform không yêu cầu nhập lại `yes`.

Nếu không sử dụng tệp kế hoạch, có thể chạy:

```powershell
terraform apply
```

Terraform sẽ hiển thị kế hoạch và yêu cầu xác nhận:

```text
Do you want to perform these actions?

  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Nhập:

```text
yes
```

để bắt đầu triển khai.

{{% notice info %}}
Quy trình `terraform plan -out="tfplan"` và `terraform apply "tfplan"` giúp bảo đảm Terraform chỉ áp dụng đúng kế hoạch đã được nhóm kiểm tra.
{{% /notice %}}

Khi triển khai thành công, Terraform hiển thị thông báo có dạng:

```text
Apply complete! Resources: ... added, ... changed, ... destroyed.
```

Số lượng tài nguyên phụ thuộc vào module và trạng thái hạ tầng tại thời điểm triển khai.

Không sử dụng một số lượng tài nguyên cố định làm kết quả chung cho toàn bộ hệ thống.

---

## Minh chứng quá trình Apply

Do hạ tầng đã được triển khai trước đó, ảnh kết quả `terraform apply` nên được lấy từ:

* Terminal của thành viên đã trực tiếp triển khai.
* Log của GitHub Actions hoặc hệ thống CI/CD.
* Log của AWS CodeBuild.
* Lịch sử triển khai được nhóm lưu lại.

Không chạy lại `terraform apply` chỉ để tạo ảnh cho báo cáo.

Do hạ tầng đã được một thành viên trong nhóm triển khai trước đó và Terminal của lần thực thi ban đầu không còn được lưu lại, nhóm kiểm tra kết quả triển khai thông qua Terraform State, Terraform Output và các tài nguyên thực tế trên AWS Management Console.

Các kết quả kiểm tra này xác nhận rằng tài nguyên đã được tạo, được Terraform quản lý và đang hoạt động trên môi trường AWS.

---

## Thứ tự triển khai các module

Hạ tầng được tổ chức theo các lớp chức năng và quan hệ phụ thuộc.

Thứ tự triển khai tổng quát gồm:

1. Bootstrap.
2. Foundation.
3. Identity.
4. Data.
5. Messaging.
6. Compute.
7. API.
8. Edge.
9. Observability.
10. Security.
11. Backup và Disaster Recovery.
12. CI/CD.

Các module tương ứng trong mã nguồn:

| Thứ tự | Module                            | Thành phần chính                                                  |
| ------ | --------------------------------- | ----------------------------------------------------------------- |
| 1      | `00-bootstrap`                    | S3 Remote Backend và cơ chế khóa Terraform State.                 |
| 2      | `01-foundation`                   | Các tài nguyên nền tảng dùng chung.                               |
| 3      | `03-identity`                     | Cognito, IAM, Lambda Post Confirmation và CloudWatch Log Group.   |
| 4      | `04-data`                         | DynamoDB Table và S3 Bucket lưu media.                            |
| 5      | `05-messaging`                    | SQS FIFO, Dead-letter Queue và EventBridge Scheduler.             |
| 6      | `06-compute`                      | Lambda Function, Lambda Layer, IAM Role và Event Source Mapping.  |
| 7      | `06-compute/stage3-control-plane` | Các Lambda xử lý phiên, vật phẩm, truy vấn và nghiệp vụ quản trị. |
| 8      | `07-api`                          | REST API, WebSocket API, Authorizer, Route, Integration và Stage. |
| 9      | `09-edge`                         | S3 và CloudFront cho User Frontend, Admin Frontend và media.      |
| 10     | `10-observability`                | Log, metric, dashboard và cảnh báo.                               |
| 11     | `11-security`                     | Các cấu hình bảo mật bổ sung.                                     |
| 12     | `12-backup-dr`                    | Sao lưu và hỗ trợ khôi phục sau sự cố.                            |
| 13     | `13-cicd`                         | Các tài nguyên phục vụ quy trình CI/CD.                           |

Thứ tự thực tế có thể được điều chỉnh theo pipeline triển khai, nhưng các module sử dụng `terraform_remote_state` chỉ được triển khai sau khi State của module phụ thuộc đã tồn tại.

---

## Triển khai module Identity

Module Identity triển khai các thành phần xác thực và phân quyền.

Di chuyển đến module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\03-identity"
```

Quy trình triển khai:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

Module này quản lý:

* Amazon Cognito User Pool.
* Cognito User Pool App Client.
* Cognito User Group dành cho User.
* Cognito User Group dành cho Admin.
* Lambda Post Confirmation.
* IAM Role và IAM Policy của Lambda.
* Lambda Permission cho phép Cognito gọi Lambda.
* CloudWatch Log Group.

---

## Triển khai module Data

Di chuyển đến module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\04-data"
```

Quy trình triển khai:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

Module Data triển khai các bảng DynamoDB phục vụ:

* Trạng thái vật phẩm đấu giá.
* Sự kiện đặt giá.
* Kết nối WebSocket.
* Bí danh người đặt giá.
* Cơ chế Idempotency.
* Danh mục phiên đấu giá.
* Danh mục sản phẩm.
* Nhật ký thao tác quản trị.

Module này cũng triển khai S3 Bucket dùng để lưu trữ dữ liệu media của sản phẩm và phiên đấu giá.

---

## Triển khai module Messaging

Di chuyển đến module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\05-messaging"
```

Quy trình triển khai:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

Module Messaging triển khai:

* SQS Queue tiếp nhận lệnh đặt giá.
* Dead-letter Queue dành cho thông điệp không xử lý được.
* EventBridge Scheduler Schedule Group.
* Scheduler Dead-letter Queue.
* IAM Role và IAM Policy cho Scheduler.
* Cơ chế xử lý yêu cầu theo thứ tự.

---

## Triển khai module Compute

Trước khi triển khai Compute, nhóm build các Lambda package và Lambda Layer bằng script trong thư mục `backend`.

Ví dụ, build toàn bộ package:

```powershell
cd "D:\ThucTap\Live-Auction"
.\backend\build.ps1 -Target all
```

Sau khi build hoàn tất, di chuyển đến module Compute:

```powershell
cd infra\06-compute
```

Quy trình triển khai:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

Module Compute triển khai:

* Lambda Bid Processor.
* Lambda WebSocket Authorizer.
* Lambda WebSocket Handler.
* Lambda Broadcast.
* Lambda Layer dùng chung.
* IAM Role và IAM Policy cho từng Lambda.
* CloudWatch Log Group.
* Event Source Mapping giữa SQS và Lambda.
* Environment Variables phục vụ kết nối các dịch vụ.

Module `stage3-control-plane` triển khai thêm:

* Session Service Lambda.
* Item Service Lambda.
* Query Service Lambda.
* Admin Command Lambda.
* Lambda Layer dùng chung.
* EventBridge Scheduler cho các nghiệp vụ vòng đời phiên đấu giá.

---

## Triển khai module API

Di chuyển đến module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\07-api"
```

Quy trình triển khai:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

Module API triển khai:

* REST API.
* REST API Resource và Method.
* Lambda Integration.
* API Gateway Stage.
* API Gateway Deployment.
* API Key và Usage Plan.
* WebSocket API.
* WebSocket Authorizer.
* Route `$connect`.
* Route `$disconnect`.
* Route `join_room`.
* Route `place_bid`.
* Lambda Permission cho API Gateway.
* CloudWatch Access Log.

---

## Triển khai module Edge

Module Edge được triển khai sau khi backend và API đã sẵn sàng.

Di chuyển đến module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\09-edge"
```

Quy trình triển khai:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

Module Edge triển khai:

* S3 Bucket dành cho User Frontend.
* S3 Bucket dành cho Admin Frontend.
* CloudFront Distribution dành cho User Frontend.
* CloudFront Distribution dành cho Admin Frontend.
* CloudFront Distribution dành cho media.
* Origin Access Control.
* S3 Bucket Policy.
* Server-side Encryption.
* S3 Public Access Block.
* Default Root Object.
* Quy tắc phân phối nội dung tĩnh.

---

## Triển khai các module vận hành

Sau khi các thành phần chức năng chính hoạt động, nhóm triển khai thêm:

### Observability

```text
infra/10-observability
```

Module này cung cấp log, metric, dashboard và cảnh báo để theo dõi hoạt động của hệ thống.

### Security

```text
infra/11-security
```

Module này bổ sung các cấu hình bảo mật và kiểm soát cho hạ tầng.

### Backup và Disaster Recovery

```text
infra/12-backup-dr
```

Module này cấu hình sao lưu và hỗ trợ khôi phục hệ thống khi xảy ra sự cố.

### CI/CD

```text
infra/13-cicd
```

Module này triển khai các thành phần hỗ trợ build, kiểm thử và triển khai tự động.

Mỗi module đều thực hiện quy trình cơ bản:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

---

## Kiểm tra Terraform State sau triển khai

Sau khi hạ tầng được triển khai, có thể xác nhận các tài nguyên được Terraform quản lý bằng:

```powershell
terraform state list
```

Ví dụ tại module Identity:

```powershell
cd "D:\ThucTap\Live-Auction\infra\03-identity"
terraform state list
```

Kết quả hiển thị các tài nguyên như:

```text
aws_cloudwatch_log_group.cognito_post_confirm
aws_cognito_user_group.admin
aws_cognito_user_group.user
aws_cognito_user_pool.main
aws_cognito_user_pool_client.web
aws_iam_role.cognito_post_confirm
aws_iam_role_policy.cognito_post_confirm
aws_lambda_function.cognito_post_confirm
aws_lambda_permission.cognito_post_confirm
```

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-state-list.png" alt="Danh sách tài nguyên trong Terraform State" width="85%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.18.</b> Các tài nguyên của module Identity được Terraform State quản lý.
    </figcaption>
</figure>

{{% notice warning %}}
Không chỉnh sửa trực tiếp Terraform State. Thao tác không chính xác với State có thể làm Terraform mất khả năng quản lý hạ tầng hoặc tạo ra thay đổi ngoài dự kiến.
{{% /notice %}}

---

## Kiểm tra Terraform Output

Sau khi triển khai, Terraform Output được sử dụng để lấy các thông tin cần thiết phục vụ việc kết nối giữa các module, frontend, backend và các dịch vụ AWS.

Tại module Identity, chạy:

```powershell
cd "D:\ThucTap\Live-Auction\infra\03-identity"
terraform output
```

Kết quả thu được:

```text
PS D:\ThucTap\Live-Auction\infra\03-identity> terraform output

cognito_client_id = "2ttqnjt0nmttmi655dav*******"
cognito_issuer = "https://cognito-idp.ap-southeast-1.amazonaws.com/ap-southeast-1_1Ly4*****"
cognito_jwks_url = "https://cognito-idp.ap-southeast-1.amazonaws.com/ap-southeast-1_1Ly4*****/.well-known/jwks.json"
cognito_user_pool_arn = "arn:aws:cognito-idp:ap-southeast-1:************:userpool/ap-southeast-1_1Ly4*****"
cognito_user_pool_client_id = "2tt*********************703g"
cognito_user_pool_id = "ap-southeast-1_1Ly4*****"

Warning: Deprecated Parameter

The parameter "dynamodb_table" is deprecated.
Use parameter "use_lockfile" instead.
```

Các Output trên có vai trò như sau:

| Output                        | Vai trò                                                                         |
| ----------------------------- | ------------------------------------------------------------------------------- |
| `cognito_client_id`           | ID của Cognito App Client được frontend sử dụng khi gửi yêu cầu xác thực.       |
| `cognito_issuer`              | Địa chỉ của đơn vị phát hành token JWT cho User Pool.                           |
| `cognito_jwks_url`            | Địa chỉ cung cấp public key để backend hoặc Authorizer kiểm tra chữ ký của JWT. |
| `cognito_user_pool_arn`       | ARN định danh Cognito User Pool trên AWS.                                       |
| `cognito_user_pool_client_id` | ID của Cognito App Client được xuất ra để các thành phần khác sử dụng.          |
| `cognito_user_pool_id`        | ID của Cognito User Pool được triển khai cho hệ thống.                          |

Các Output này được sử dụng để:

* Cấu hình User Frontend và Admin Frontend.
* Gửi yêu cầu đăng ký và đăng nhập đến Amazon Cognito.
* Kiểm tra token JWT tại backend hoặc API Authorizer.
* Truyền thông tin của module Identity sang các module phụ thuộc.
* Xác định User Pool và App Client đang được sử dụng.
* Kết nối các dịch vụ backend với hệ thống xác thực.

Để lấy riêng một Output, có thể sử dụng:

```powershell
terraform output cognito_user_pool_id
```

Để lấy giá trị không kèm dấu nháy, sử dụng tùy chọn `-raw`:

```powershell
terraform output -raw cognito_user_pool_id
```

Để xuất toàn bộ Output dưới dạng JSON:

```powershell
terraform output -json
```

Kết quả JSON có thể được sử dụng cho script triển khai hoặc truyền giá trị sang các bước cấu hình tiếp theo.

{{% notice info %}}
Cảnh báo `dynamodb_table is deprecated` xuất hiện vì Terraform Backend của module hiện vẫn sử dụng bảng DynamoDB để khóa Terraform State. Cấu hình này vẫn hoạt động tại thời điểm kiểm tra, nhưng phiên bản Terraform mới khuyến nghị sử dụng tùy chọn `use_lockfile` của S3 Backend. Đây là cảnh báo về khả năng tương thích trong tương lai, không phải lỗi của lệnh `terraform output`.
{{% /notice %}}

{{% notice warning %}}
Cognito User Pool ID, App Client ID và ARN không phải mật khẩu hoặc Secret Key. Tuy nhiên, các giá trị này vẫn làm lộ thông tin định danh và cấu trúc tài nguyên AWS của hệ thống. Vì repository báo cáo được đặt ở chế độ Public, nhóm đã che một phần Account ID, User Pool ID, App Client ID và ARN trước khi đưa kết quả vào báo cáo. Tuyệt đối không công khai Client Secret, Access Token, Refresh Token, mật khẩu hoặc thông tin xác thực AWS.
{{% /notice %}}

Kết quả `terraform output` xác nhận module Identity đã được triển khai, Terraform State đang lưu các giá trị đầu ra và các thành phần khác có thể sử dụng những giá trị này để kết nối với Amazon Cognito.

---

## Xác nhận tính đồng bộ sau triển khai

Sau khi triển khai, nhóm chạy lại:

```powershell
terraform plan -no-color
```

Nếu hạ tầng phù hợp với cấu hình, Terraform hiển thị:

```text
No changes. Your infrastructure matches the configuration.
```

Kết quả này xác nhận:

* Tài nguyên trên AWS đã được triển khai theo cấu hình.
* Terraform State đã được cập nhật.
* Không còn thay đổi chưa được áp dụng.
* Mã Terraform và hạ tầng thực tế đang đồng bộ.

Ảnh xác nhận `No changes` đã được trình bày trong mục **5.3.4 – Kiểm tra kế hoạch triển khai**, vì vậy không cần chèn lại cùng một ảnh trong mục này.

---

## Kiểm tra tài nguyên trên AWS Management Console

Ngoài Terraform State, nhóm đăng nhập AWS Management Console và kiểm tra:

* Cognito User Pool và App Client.
* IAM Role và IAM Policy.
* DynamoDB Table.
* SQS FIFO Queue và Dead-letter Queue.
* Lambda Function và Lambda Layer.
* REST API và WebSocket API.
* S3 Bucket.
* CloudFront Distribution.
* CloudWatch Log Group và Alarm.
* Các tài nguyên CI/CD.

Việc kiểm tra trên AWS Management Console xác nhận các tài nguyên đã được tạo và đang tồn tại trong tài khoản AWS.

Danh sách tổng hợp các tài nguyên được trình bày tại mục **5.4 – Các dịch vụ AWS được triển khai**.

---

## Một số lỗi thường gặp

### Thiếu Lambda package

Terraform có thể báo lỗi:

```text
Call to function "filebase64sha256" failed
```

hoặc:

```text
The system cannot find the file specified.
```

Nguyên nhân là tệp ZIP của Lambda chưa được build trên máy hiện tại.

Ví dụ, build Lambda Post Confirmation:

```powershell
cd "D:\ThucTap\Live-Auction"
.\backend\build.ps1 -Target function -FunctionName cognito_post_confirm
```

### Kế hoạch đã hết hiệu lực

Terraform có thể báo:

```text
Saved plan is stale
```

Nguyên nhân là cấu hình hoặc State đã thay đổi sau khi tệp `tfplan` được tạo.

Cần tạo lại kế hoạch:

```powershell
terraform plan -out="tfplan"
```

Sau đó kiểm tra lại trước khi apply.

### Không đủ quyền AWS

Thông báo:

```text
AccessDenied
```

Kiểm tra danh tính AWS:

```powershell
aws sts get-caller-identity
```

Sau đó kiểm tra IAM Policy của tài khoản hoặc Role đang triển khai.

### State đang bị khóa

Terraform có thể báo lỗi khi một tiến trình khác đang thao tác với cùng State.

Không tự ý xóa khóa hoặc sử dụng `force-unlock` khi chưa xác nhận tiến trình trước đã kết thúc.

### Tài nguyên đã tồn tại ngoài State

Nếu tài nguyên được tạo thủ công nhưng chưa được Terraform quản lý, quá trình apply có thể báo lỗi trùng tên.

Cần kiểm tra tài nguyên thực tế và cân nhắc sử dụng `terraform import` thay vì tạo lại.

### CloudFront chưa cập nhật ngay

Sau khi triển khai, CloudFront cần thời gian chuyển sang trạng thái `Deployed`. Cần chờ quá trình phân phối hoàn tất trước khi kiểm tra frontend.

---

## Lưu ý về việc hủy tài nguyên

Lệnh:

```powershell
terraform destroy
```

sẽ xóa các tài nguyên do module hiện tại quản lý.

{{% notice danger %}}
Không chạy `terraform destroy` trên môi trường Live Auction đang hoạt động. Lệnh này có thể xóa Cognito User Pool, Lambda Function, DynamoDB Table, SQS Queue, API Gateway, S3 Bucket hoặc những tài nguyên quan trọng khác, gây gián đoạn hệ thống và mất dữ liệu.
{{% /notice %}}

Việc hủy tài nguyên chỉ được thực hiện trong môi trường thử nghiệm, sau khi:

* Xác định chính xác module và phạm vi tài nguyên.
* Sao lưu dữ liệu cần thiết.
* Kiểm tra kế hoạch hủy.
* Nhận được sự thống nhất của nhóm.
* Bảo đảm không ảnh hưởng đến người dùng.

---

## Kết quả

Sau khi hoàn tất quá trình triển khai:

* Remote Backend và Terraform State đã được thiết lập.
* Các module được triển khai theo quan hệ phụ thuộc.
* Amazon Cognito và các tài nguyên IAM đã được tạo.
* Lambda Post Confirmation đã được tích hợp với Cognito.
* Các bảng Amazon DynamoDB đã được triển khai.
* S3 Bucket lưu trữ media đã được cấu hình.
* Amazon SQS FIFO và Dead-letter Queue đã được tạo.
* EventBridge Scheduler đã được cấu hình cho các nghiệp vụ theo thời gian.
* Các AWS Lambda Function và Lambda Layer đã được triển khai.
* REST API và WebSocket API đã được tạo trên Amazon API Gateway.
* User Frontend và Admin Frontend được lưu trữ trên các S3 Bucket riêng.
* Các CloudFront Distribution đã được cấu hình để phân phối frontend và media.
* Các thành phần giám sát, bảo mật, sao lưu và CI/CD đã được bổ sung.
* Terraform State đã ghi nhận các tài nguyên được quản lý.
* Terraform Output cung cấp các giá trị cần thiết cho những thành phần phụ thuộc.
* Kết quả kiểm tra sau triển khai xác nhận mã Terraform và hạ tầng AWS đang đồng bộ.
* Hạ tầng đã sẵn sàng để kiểm tra chi tiết từng dịch vụ trong mục tiếp theo.