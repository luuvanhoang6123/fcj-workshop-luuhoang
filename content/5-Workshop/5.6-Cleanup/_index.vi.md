---
title: "Dọn dẹp tài nguyên"
date: 2026-08-09
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

## Tổng quan

Sau khi hoàn thành Workshop, các tài nguyên AWS không còn sử dụng nên được dọn dẹp để tránh tiếp tục phát sinh chi phí.

Do hệ thống Live Auction vẫn cần được duy trì để trình diễn, kiểm tra và chấm đồ án, nhóm **chưa thực hiện xóa tài nguyên tại thời điểm viết báo cáo**. Nội dung dưới đây trình bày quy trình dọn dẹp sẽ được thực hiện sau khi đồ án đã được nghiệm thu và không còn yêu cầu duy trì website.

{{% notice warning %}}
Không thực hiện `terraform destroy` vì hệ thống vẫn đang được sử dụng để trình diễn hoặc chấm đồ án. Thao tác này có thể xóa website, API, Lambda Function, dữ liệu DynamoDB, tài khoản Cognito và các tài nguyên liên quan, khiến hệ thống không thể tiếp tục truy cập. Nhóm chỉ thực hiện dọn dẹp sau khi đồ án đã được nghiệm thu và được xác nhận không còn cần duy trì hệ thống.
{{% /notice %}}

## Lưu ý trước khi dọn dẹp

Trước khi xóa tài nguyên, cần xác nhận:

- Đồ án đã được nghiệm thu hoặc chấm hoàn tất.
- Website không còn được sử dụng để trình diễn.
- Source code đã được lưu trên GitHub.
- Báo cáo và hình ảnh đã được sao lưu.
- Không còn thành viên nào đang sử dụng môi trường AWS.
- Các dữ liệu cần giữ lại đã được sao lưu.
- AWS CLI đang sử dụng đúng tài khoản và Region.
- Terraform State vẫn còn tồn tại và có thể truy cập.
- Không có quá trình `terraform apply` hoặc CI/CD nào đang chạy.

Kiểm tra tài khoản AWS hiện tại:

```powershell
aws sts get-caller-identity
```

Kiểm tra Region:

```powershell
aws configure get region
```

Region được sử dụng trong dự án:

```text
ap-southeast-1
```


## Sao lưu dữ liệu

Trước khi dọn dẹp, nhóm cần lưu lại những dữ liệu quan trọng.

### Source code và báo cáo

Kiểm tra toàn bộ thay đổi đã được commit và đẩy lên GitHub:

```powershell
git status
git add .
git commit -m "Complete Live Auction workshop"
git push
```

Nếu `git status` hiển thị:

```text
nothing to commit, working tree clean
```

thì các thay đổi hiện tại đã được commit.

### Dữ liệu Amazon S3

Tải về máy các tệp cần giữ trong:

- User Frontend Bucket.
- Admin Frontend Bucket.
- Item Media Bucket.
- Hình ảnh vật phẩm.
- Tệp cấu hình hoặc artifact cần thiết.

Có thể sử dụng AWS CLI:

```powershell
aws s3 sync "s3://<bucket-name>" ".\backup\<bucket-name>"
```

Thay `<bucket-name>` bằng tên Bucket cần sao lưu.

### Dữ liệu Amazon DynamoDB

Các bảng quan trọng cần được sao lưu gồm:

```text
la_item_auction_state
la_bid_events
la_auction_catalog
la_category_catalog
la_admin_audit_events
```

Có thể sử dụng:

- Point-in-Time Recovery.
- AWS Backup.
- Export to Amazon S3.
- Bản sao lưu theo yêu cầu.

Cần kiểm tra trạng thái bản sao lưu là:

```text
Available
```

trước khi tiếp tục dọn dẹp.

### Cấu hình Terraform

Cần giữ lại:

```text
*.tf
*.tfvars
.terraform.lock.hcl
```

Không đưa các tệp chứa thông tin nhạy cảm lên GitHub. Nếu file `terraform.tfvars` có dữ liệu bí mật thì chỉ lưu ở nơi an toàn và bảo đảm file đã được khai báo trong `.gitignore`.

## Kiểm tra kế hoạch hủy tài nguyên

Trước khi xóa tài nguyên của một module, mở đúng thư mục Terraform và chạy:

```powershell
terraform init
```

Sau đó tạo kế hoạch hủy:

```powershell
terraform plan -destroy
```

Lệnh trên chỉ hiển thị kế hoạch, chưa xóa tài nguyên.

Cần kiểm tra:

- Đúng AWS Account.
- Đúng Region.
- Đúng Terraform Workspace.
- Đúng Backend State.
- Danh sách tài nguyên sẽ bị xóa.
- Không có tài nguyên ngoài phạm vi dự án.
- Không có thay đổi ngoài dự kiến.

{{% notice info %}}
Nên chạy `terraform plan -destroy` trước mỗi lần `terraform destroy`. Chỉ tiếp tục khi đã xác nhận toàn bộ tài nguyên trong kế hoạch thuộc hệ thống Live Auction.
{{% /notice %}}

## Hủy tài nguyên bằng Terraform

Do hạ tầng được chia thành nhiều Terraform module, tài nguyên cần được hủy theo **thứ tự ngược với quá trình triển khai**. Việc này giúp hạn chế lỗi phụ thuộc giữa các module.

Thứ tự tham khảo:

```text
13-cicd
12-backup-dr
11-security
10-observability
09-edge
07-api
06-compute/stage3-control-plane
06-compute
05-messaging
04-data
03-identity
01-foundation
00-bootstrap
```

Tại từng module, chạy:

```powershell
terraform init
terraform plan -destroy
terraform destroy
```

Terraform yêu cầu xác nhận:

```text
Enter a value:
```

Chỉ nhập:

```text
yes
```

sau khi đã kiểm tra chính xác kế hoạch hủy.

Ví dụ:

```powershell
cd "D:\ThucTap\Live-Auction\infra\09-edge"
terraform init
terraform plan -destroy
terraform destroy
```

Sau khi hoàn tất một module, tiếp tục với module phụ thuộc tiếp theo.

{{% notice warning %}}
Không sao chép và chạy toàn bộ lệnh hủy cho tất cả module cùng lúc. Cần xử lý lần lượt từng module và kiểm tra kết quả trước khi chuyển sang module tiếp theo.
{{% /notice %}}

## Một số tài nguyên cần xử lý trước

Một số tài nguyên có thể khiến `terraform destroy` thất bại nếu vẫn còn dữ liệu hoặc đang được sử dụng.

### Amazon S3

S3 Bucket có thể không xóa được nếu vẫn còn:

- Object.
- Object Version.
- Delete Marker.
- Multipart Upload chưa hoàn thành.

Cần sao lưu dữ liệu cần giữ và làm trống Bucket trước khi hủy module chứa Bucket.

### Amazon CloudFront

CloudFront Distribution phải được vô hiệu hóa trước khi bị xóa. Quá trình cập nhật và xóa Distribution có thể mất vài phút.

### Amazon DynamoDB

Trước khi xóa bảng:

- Kiểm tra bản sao lưu.
- Kiểm tra Point-in-Time Recovery.
- Xác nhận dữ liệu không còn cần sử dụng.
- Không xóa bảng khi hệ thống vẫn đang ghi dữ liệu.

### Amazon Cognito

Xóa Cognito User Pool sẽ xóa tài khoản người dùng và cấu hình App Client liên quan. Cần chắc chắn hệ thống không còn yêu cầu đăng nhập hoặc trình diễn.

### Lambda và API Gateway

Sau khi Lambda Function và API Gateway bị xóa:

- REST API không còn hoạt động.
- WebSocket không thể kết nối.
- Frontend không thể gọi backend.
- Chức năng đấu giá thời gian thực ngừng hoạt động.

### Terraform Backend

Module `00-bootstrap` có thể chứa S3 Bucket lưu Terraform State và tài nguyên khóa State. Vì vậy, module này phải được xử lý **cuối cùng**.

Trước khi xóa Backend:

- Bảo đảm các module khác đã được hủy.
- Sao lưu Terraform State.
- Kiểm tra không còn tiến trình Terraform đang chạy.
- Không xóa Backend khi các module khác vẫn phụ thuộc vào State.

## Kiểm tra sau khi dọn dẹp

Sau khi Terraform hoàn tất, đăng nhập vào AWS Management Console và kiểm tra các dịch vụ đã sử dụng:

- Amazon Cognito.
- AWS IAM.
- Amazon S3.
- Amazon CloudFront.
- AWS Lambda.
- Amazon API Gateway.
- Amazon DynamoDB.
- Amazon SQS.
- Amazon CloudWatch.
- AWS Backup.
- AWS CodeBuild và các tài nguyên CI/CD.

Kiểm tra bằng AWS Resource Explorer với tiền tố tài nguyên của dự án:

```text
la-
```

Các nội dung cần xác nhận:

- Không còn tài nguyên không cần thiết.
- Không còn CloudFront Distribution đang hoạt động.
- Không còn Lambda Function của dự án.
- Không còn API Gateway REST API và WebSocket API.
- Không còn SQS Queue.
- Không còn DynamoDB Table chưa được sao lưu.
- Không còn S3 Bucket chứa dữ liệu không cần thiết.
- Không còn CloudWatch Log Group có thời gian lưu trữ không giới hạn.
- Không còn Pipeline hoặc Build Project tiếp tục chạy.

Cuối cùng, mở:

```text
AWS Billing and Cost Management
```

Kiểm tra:

- Bills.
- Cost Explorer.
- Free Tier.
- Budgets.
- Các dịch vụ vẫn đang phát sinh chi phí.

{{% notice info %}}
Thông tin chi phí trên AWS có thể cập nhật chậm. Sau khi dọn dẹp, cần tiếp tục kiểm tra Billing trong những ngày tiếp theo để xác nhận không còn tài nguyên phát sinh chi phí ngoài dự kiến.
{{% /notice %}}

## Kết quả dự kiến

Sau khi quy trình dọn dẹp được thực hiện:

- Các tài nguyên AWS của hệ thống Live Auction được xóa theo đúng thứ tự.
- Dữ liệu quan trọng đã được sao lưu.
- Terraform State được xử lý sau cùng.
- Không còn dịch vụ không cần thiết tiếp tục phát sinh chi phí.
- Source code, báo cáo và tài liệu triển khai vẫn được lưu giữ trên GitHub.
- Hệ thống có thể được triển khai lại bằng Terraform khi cần thiết.

Tại thời điểm hoàn thành báo cáo, nhóm chỉ trình bày quy trình dọn dẹp và **chưa thực hiện `terraform destroy`** để bảo đảm website vẫn có thể được truy cập phục vụ quá trình trình diễn và đánh giá đồ án.