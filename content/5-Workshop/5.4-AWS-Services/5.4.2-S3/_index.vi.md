---
title: "Amazon S3"
date: 2026-07-27
weight: 2
chapter: false
pre: " <b> 5.4.2. </b> "
---

## Tổng quan

Hệ thống **Live Auction** sử dụng **Amazon Simple Storage Service (Amazon S3)** để lưu trữ hai giao diện frontend và các tệp phương tiện của vật phẩm đấu giá.

Ba nhóm nội dung chính được lưu trên Amazon S3 gồm:

* Giao diện dành cho người dùng — User Frontend.
* Giao diện dành cho quản trị viên — Admin Frontend.
* Hình ảnh và tệp phương tiện của các vật phẩm đấu giá.

Các S3 Bucket được tạo và cấu hình bằng Terraform trong hai module:

```text
infra/04-data
infra/09-edge
```

Module `04-data` tạo Media Bucket dùng để lưu hình ảnh vật phẩm. Module `09-edge` tạo hai bucket frontend và kết nối các bucket với Amazon CloudFront.


## Vai trò của Amazon S3

Amazon S3 là dịch vụ lưu trữ đối tượng có khả năng mở rộng cao trên AWS.

Trong hệ thống Live Auction, Amazon S3 được sử dụng để:

* Lưu các tệp HTML, CSS, JavaScript và tài nguyên tĩnh của User Frontend.
* Lưu riêng các tệp của Admin Frontend.
* Lưu hình ảnh và tệp phương tiện của vật phẩm đấu giá.
* Làm origin cho các CloudFront Distribution.
* Hỗ trợ phân phối nội dung frontend đến người dùng.
* Hỗ trợ quản lý nhiều phiên bản của đối tượng.
* Mã hóa dữ liệu được lưu trong bucket.
* Ngăn truy cập công khai trực tiếp vào bucket.
* Cho phép CloudFront truy cập đối tượng thông qua Origin Access Control.

## Các S3 Bucket của hệ thống

Hệ thống sử dụng ba S3 Bucket chính.

| S3 Bucket                 | Vai trò                                                    |
| ------------------------- | ---------------------------------------------------------- |
| **User Frontend Bucket**  | Lưu các tệp đã build của giao diện dành cho người dùng.    |
| **Admin Frontend Bucket** | Lưu các tệp đã build của giao diện dành cho quản trị viên. |
| **Item Media Bucket**     | Lưu hình ảnh và tệp phương tiện của các vật phẩm đấu giá.  |

Tên bucket được tạo theo quy ước của dự án và có chứa AWS Account ID để bảo đảm tên bucket là duy nhất trên toàn bộ Amazon S3.

Cấu trúc tên tổng quát như sau:

```text
<name-prefix>-<environment>-frontend-<account-id>
<name-prefix>-<environment>-admin-frontend-<account-id>
<name-prefix>-item-media-<account-id>-<aws-region>
```

Ví dụ sau khi che Account ID:

```text
la-dev-frontend-************
la-dev-admin-frontend-************
la-item-media-************-ap-southeast-1
```

## User Frontend Bucket

User Frontend Bucket lưu kết quả build của giao diện dành cho người dùng.

Các tệp thường có trong bucket gồm:

```text
index.html
assets/
favicon.ico
```

Thư mục `assets` có thể chứa:

* Các tệp JavaScript đã được build.
* Các tệp CSS.
* Hình ảnh tĩnh.
* Font chữ và các tài nguyên giao diện khác.

Người dùng không truy cập trực tiếp S3 Bucket. Thay vào đó, nội dung được phân phối thông qua Amazon CloudFront.

## Admin Frontend Bucket

Admin Frontend Bucket được triển khai riêng để lưu giao diện quản trị.

Việc sử dụng bucket riêng giúp:

* Tách biệt User Frontend và Admin Frontend.
* Triển khai hai giao diện độc lập.
* Quản lý nội dung của từng giao diện riêng biệt.
* Cấu hình CloudFront Distribution riêng.
* Giảm nguy cơ nhầm lẫn tài nguyên giữa User và Admin.
* Dễ dàng cập nhật một giao diện mà không ảnh hưởng đến giao diện còn lại.

Admin Frontend Bucket cũng được cấu hình chặn truy cập công khai và chỉ cung cấp nội dung thông qua CloudFront.

## Item Media Bucket

Item Media Bucket lưu hình ảnh và tệp phương tiện của các vật phẩm được đưa vào phiên đấu giá.

Tên bucket có cấu trúc:

```text
<name-prefix>-item-media-<account-id>-<aws-region>
```

Media Bucket được cấu hình:

* Chặn toàn bộ truy cập công khai.
* Sử dụng Bucket Owner Enforced.
* Bật Versioning.
* Mã hóa phía máy chủ bằng `AES256`.
* Cấu hình CORS cho các thao tác được cho phép.
* Xóa multipart upload chưa hoàn thành sau 7 ngày.
* Cho phép CloudFront đọc đối tượng thông qua Bucket Policy.
* Không tự động xóa bucket khi Terraform hủy tài nguyên.

Cấu hình `force_destroy = false` giúp hạn chế việc vô tình xóa bucket khi bên trong vẫn còn dữ liệu.

## Cơ chế truy cập S3

Hai frontend và nội dung media không được công khai trực tiếp từ S3.

Luồng truy cập nội dung được thực hiện như sau:

1. Người dùng truy cập tên miền của CloudFront Distribution.
2. CloudFront tiếp nhận yêu cầu.
3. CloudFront sử dụng Origin Access Control để yêu cầu nội dung từ S3.
4. S3 Bucket Policy kiểm tra CloudFront Distribution gửi yêu cầu.
5. Nếu yêu cầu hợp lệ, S3 trả đối tượng về CloudFront.
6. CloudFront phân phối nội dung đến trình duyệt người dùng.

Cơ chế này giúp giữ S3 Bucket ở trạng thái private nhưng nội dung vẫn có thể được phân phối thông qua CloudFront.

{{% notice info %}}
Hai frontend bucket không cần bật **Static website hosting** vì CloudFront sử dụng S3 Regional Domain Name và Origin Access Control để truy cập bucket. Việc bật Static website hosting sẽ không phù hợp với cơ chế OAC đang được sử dụng.
{{% /notice %}}

## Các cấu hình bảo mật

### Block Public Access

Các bucket của hệ thống được bật toàn bộ thiết lập Block Public Access:

```text
Block public ACLs
Ignore public ACLs
Block public bucket policies
Restrict public buckets
```

Cấu hình này ngăn người dùng truy cập trực tiếp nội dung thông qua URL của S3.

### Server-side Encryption

Các bucket được cấu hình mã hóa phía máy chủ bằng:

```text
Amazon S3 managed keys — SSE-S3
AES256
```

Amazon S3 tự động mã hóa đối tượng khi lưu trữ và giải mã khi đối tượng được truy xuất bởi một yêu cầu hợp lệ.

### Bucket Versioning

Versioning được bật để Amazon S3 lưu giữ nhiều phiên bản của một đối tượng.

Cấu hình này hỗ trợ:

* Khôi phục tệp sau khi bị ghi đè.
* Hạn chế ảnh hưởng khi xóa nhầm đối tượng.
* Theo dõi các phiên bản nội dung frontend.
* Tăng khả năng khôi phục dữ liệu media.

### Bucket Policy và Origin Access Control

User Frontend Bucket, Admin Frontend Bucket và Media Bucket sử dụng Bucket Policy để cho phép CloudFront đọc đối tượng.

Quyền chính được sử dụng là:

```text
s3:GetObject
```

Bucket Policy giới hạn quyền truy cập theo đúng CloudFront Distribution, thay vì cho phép mọi người truy cập công khai.

## Kiểm tra S3 Bucket trên AWS Management Console

### Bước 1: Truy cập Amazon S3

Đăng nhập vào **AWS Management Console**.

Tại thanh tìm kiếm phía trên, nhập:

```text
S3
```

Chọn **S3 — Scalable Storage in the Cloud**.

### Bước 2: Mở danh sách bucket

Tại menu bên trái, chọn:

```text
General purpose buckets
```

Tìm các bucket có tiền tố:

```text
la-
```

Kiểm tra sự tồn tại của:

* User Frontend Bucket.
* Admin Frontend Bucket.
* Item Media Bucket.

Các nội dung cần kiểm tra gồm:

* Tên bucket.
* AWS Region.
* Ngày tạo.
* Bucket có tuân thủ quy ước đặt tên của dự án hay không.
* Ba bucket cần thiết đã được tạo đầy đủ hay chưa.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-bucket-list.png"
    title="Hình 5.4.2.1: Các S3 Bucket của hệ thống Live Auction"
    width="100%"
>}}

### Bước 3: Kiểm tra nội dung User Frontend Bucket

Chọn User Frontend Bucket.

Mở tab:

```text
Objects
```

Kiểm tra các tệp frontend đã được tải lên bucket, chẳng hạn:

```text
index.html
assets/
```

Sự xuất hiện của `index.html` và thư mục chứa tài nguyên build cho thấy giao diện người dùng đã được đưa lên S3.

Không mở nội dung tệp cấu hình nếu tệp có chứa API endpoint, Client ID hoặc thông tin không muốn công khai.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-user-frontend-objects.png"
    title="Hình 5.4.2.2: Nội dung được lưu trong User Frontend Bucket"
    width="100%"
>}}

### Bước 4: Kiểm tra nội dung Admin Frontend Bucket

Quay lại danh sách bucket và chọn Admin Frontend Bucket.

Mở tab:

```text
Objects
```

Kiểm tra các tệp đã build của Admin Frontend.

Nội dung của Admin Frontend được lưu độc lập với User Frontend, qua đó xác nhận hai giao diện được triển khai bằng hai bucket riêng biệt.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-admin-frontend-objects.png"
    title="Hình 5.4.2.3: Nội dung được lưu trong Admin Frontend Bucket"
    width="100%"
>}}

### Bước 5: Kiểm tra nội dung Item Media Bucket

Quay lại danh sách bucket và chọn Item Media Bucket.

Mở tab:

```text
Objects
```

Kiểm tra các thư mục hoặc đối tượng hình ảnh của vật phẩm đấu giá.

Nếu bucket chưa có dữ liệu, giao diện có thể hiển thị trạng thái trống. Trạng thái này không có nghĩa là bucket triển khai thất bại; dữ liệu chỉ xuất hiện sau khi hệ thống tải hình ảnh vật phẩm lên S3.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-item-media-objects.png"
    title="Hình 5.4.2.4: Dữ liệu trong Item Media Bucket"
    width="100%"
>}}

### Bước 6: Kiểm tra Block Public Access

Trong một bucket của dự án, mở tab:

```text
Permissions
```

Tìm phần:

```text
Block public access (bucket settings)
```

Kiểm tra trạng thái:

```text
Block all public access: On
```

Sau đó kiểm tra phần **Bucket policy** để xác nhận CloudFront được cấp quyền đọc đối tượng từ bucket.

Không chỉnh sửa Bucket Policy trực tiếp trên Console vì cấu hình đang được quản lý bằng Terraform.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-block-public-access.png"
    title="Hình 5.4.2.5: Cấu hình Block Public Access của S3 Bucket"
    width="100%"
>}}

### Bước 7: Kiểm tra Versioning và Encryption

Trong trang chi tiết bucket, mở tab:

```text
Properties
```

Kiểm tra các phần:

```text
Bucket Versioning
Default encryption
```

Trạng thái cần xác nhận:

* Bucket Versioning đã được bật.
* Default encryption đã được bật.
* Encryption type sử dụng Amazon S3 managed keys.
* Thuật toán mã hóa là `SSE-S3` hoặc `AES256`.



{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-bucket-versioning.png"
    title="Hình 5.4.2.6: Cấu hình Versioning của User Frontend Bucket"
    width="100%"
>}}
Kết quả kiểm tra cho thấy **Bucket Versioning** đang ở trạng thái `Enabled`. Nhờ đó, Amazon S3 có thể lưu giữ nhiều phiên bản của cùng một đối tượng và hỗ trợ khôi phục dữ liệu khi tệp bị ghi đè hoặc xóa nhầm.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-default-encryption.png"
    title="Hình 5.4.2.7: Cấu hình mã hóa mặc định của User Frontend Bucket"
    width="100%"
>}}

Kết quả kiểm tra cho thấy bucket sử dụng cơ chế **Server-side encryption with Amazon S3 managed keys — SSE-S3**. Các đối tượng mới được tải lên bucket sẽ tự động được Amazon S3 mã hóa trước khi lưu trữ.

### Bước 8: Kiểm tra CORS của Item Media Bucket

Mở Item Media Bucket và chọn:

```text
Permissions → Cross-origin resource sharing (CORS)
```

Kiểm tra các phương thức được cho phép:

```text
GET
POST
```

CORS cho phép frontend gửi các yêu cầu phù hợp đến Media Bucket từ những origin đã được cấu hình.

Không công khai toàn bộ danh sách origin nếu có địa chỉ môi trường nội bộ hoặc thông tin không muốn công khai.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.2-S3/s3-media-cors.png"
    title="Hình 5.4.2.8: Cấu hình CORS của Item Media Bucket"
    width="100%"
>}}

## Kết quả

Sau khi kiểm tra trực tiếp trên AWS Management Console, nhóm ghi nhận:

* User Frontend Bucket đã được tạo và chứa các tệp giao diện người dùng.
* Admin Frontend Bucket đã được tạo và chứa các tệp giao diện quản trị.
* Hai giao diện frontend được lưu trữ trong hai bucket độc lập.
* Item Media Bucket đã được tạo để lưu hình ảnh vật phẩm đấu giá.
* Các bucket được bật Versioning.
* Dữ liệu được mã hóa phía máy chủ bằng Amazon S3 managed keys.
* Block Public Access được bật để ngăn truy cập trực tiếp từ Internet.
* Frontend Bucket và Media Bucket được CloudFront truy cập thông qua Origin Access Control.
* Bucket Policy giới hạn quyền đọc đối tượng cho CloudFront Distribution phù hợp.
* Media Bucket được cấu hình CORS cho các phương thức cần thiết.
* Các S3 Bucket đã sẵn sàng để tích hợp với Amazon CloudFront và các thành phần còn lại của hệ thống.

Việc cấu hình và kiểm tra quá trình phân phối nội dung từ S3 thông qua Amazon CloudFront được trình bày tại mục **5.4.3 — Amazon CloudFront**.