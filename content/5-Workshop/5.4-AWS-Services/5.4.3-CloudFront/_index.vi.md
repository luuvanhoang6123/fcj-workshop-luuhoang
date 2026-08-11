---
title: "Amazon CloudFront"
date: 2026-07-27
weight: 3
chapter: false
pre: " <b> 5.4.3. </b> "
---

## Tổng quan

Hệ thống **Live Auction** sử dụng **Amazon CloudFront** để phân phối hai giao diện frontend và các tệp phương tiện của vật phẩm đấu giá.

CloudFront hoạt động như một mạng phân phối nội dung — Content Delivery Network (CDN), tiếp nhận yêu cầu từ người dùng, lấy nội dung từ Amazon S3 và phân phối nội dung thông qua các Edge Location của AWS.

Các CloudFront Distribution của hệ thống được tạo và cấu hình bằng Terraform trong module:

```text
infra/09-edge
```

Hệ thống sử dụng ba CloudFront Distribution:

* Distribution dành cho User Frontend.
* Distribution dành cho Admin Frontend.
* Distribution dành cho hình ảnh và nội dung Item Media.


## Vai trò của Amazon CloudFront

Trong hệ thống Live Auction, Amazon CloudFront được sử dụng để:

* Phân phối User Frontend đến người dùng.
* Phân phối Admin Frontend đến quản trị viên.
* Phân phối hình ảnh của các vật phẩm đấu giá.
* Giảm thời gian tải nội dung thông qua hệ thống Edge Location.
* Lưu nội dung vào bộ nhớ đệm — cache.
* Chuyển hướng các yêu cầu HTTP sang HTTPS.
* Giới hạn truy cập trực tiếp vào S3 Bucket.
* Truy cập S3 thông qua Origin Access Control.
* Hỗ trợ định tuyến cho ứng dụng frontend dạng Single Page Application.
* Tách biệt quá trình phân phối User Frontend, Admin Frontend và Item Media.

## Các CloudFront Distribution của hệ thống

| CloudFront Distribution         | Origin                   | Vai trò                                             |
| ------------------------------- | ------------------------ | --------------------------------------------------- |
| **User Frontend Distribution**  | User Frontend S3 Bucket  | Phân phối giao diện dành cho người dùng.            |
| **Admin Frontend Distribution** | Admin Frontend S3 Bucket | Phân phối giao diện dành cho quản trị viên.         |
| **Item Media Distribution**     | Item Media S3 Bucket     | Phân phối hình ảnh và tệp phương tiện của vật phẩm. |

Mỗi Distribution có một Domain Name mặc định do CloudFront cung cấp, có cấu trúc:

```text
xxxxxxxxxxxxxx.cloudfront.net
```

User Frontend và Admin Frontend sử dụng hai Distribution riêng biệt. Nhờ đó, nhóm có thể triển khai, cập nhật và quản lý hai giao diện độc lập.

## Luồng phân phối nội dung

Luồng phân phối nội dung frontend được thực hiện như sau:

1. Người dùng truy cập CloudFront Domain Name.
2. CloudFront kiểm tra nội dung được yêu cầu có tồn tại trong cache hay không.
3. Nếu nội dung đã có trong cache, CloudFront trả nội dung trực tiếp cho người dùng.
4. Nếu nội dung chưa có trong cache, CloudFront gửi yêu cầu đến S3 Origin.
5. CloudFront sử dụng Origin Access Control để ký yêu cầu đến Amazon S3.
6. S3 Bucket Policy kiểm tra CloudFront Distribution gửi yêu cầu.
7. Amazon S3 trả đối tượng về CloudFront.
8. CloudFront lưu nội dung phù hợp vào cache và trả kết quả về trình duyệt.

Luồng phân phối Item Media cũng hoạt động tương tự, nhưng origin là Item Media Bucket thay vì frontend bucket.

## Origin Access Control

Mỗi CloudFront Distribution được cấu hình một **Origin Access Control (OAC)** tương ứng:

```text
User Frontend OAC
Admin Frontend OAC
Item Media OAC
```

Origin Access Control được cấu hình với:

```text
Origin type: S3
Signing behavior: Always
Signing protocol: SigV4
```

OAC cho phép CloudFront truy cập bucket private bằng yêu cầu đã được ký. Người dùng không được phép truy cập trực tiếp các đối tượng thông qua URL của S3.

Cấu hình này giúp:

* Giữ S3 Bucket ở trạng thái private.
* Ngăn truy cập trái phép vào bucket.
* Chỉ cho phép CloudFront Distribution phù hợp đọc dữ liệu.
* Tách quyền truy cập giữa ba S3 Origin.
* Hạn chế việc công khai trực tiếp nội dung trên Amazon S3.

## Cấu hình User Frontend Distribution

User Frontend Distribution được cấu hình:

```text
Default root object: index.html
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Cached methods: GET, HEAD
Compress objects automatically: Enabled
Price class: PriceClass_100
```

Origin của Distribution là User Frontend S3 Bucket.

CloudFront chỉ cho phép các phương thức `GET` và `HEAD` vì frontend chủ yếu được sử dụng để đọc và tải các tài nguyên tĩnh.

Tính năng nén được bật để giảm kích thước nội dung truyền đến trình duyệt và cải thiện tốc độ tải trang.

## Cấu hình Admin Frontend Distribution

Admin Frontend Distribution được triển khai riêng và có cấu hình tương tự User Frontend Distribution:

```text
Default root object: index.html
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Cached methods: GET, HEAD
Compress objects automatically: Enabled
Price class: PriceClass_100
```

Origin của Distribution là Admin Frontend S3 Bucket.

Việc tách Distribution giúp giao diện quản trị không sử dụng chung origin và nội dung cache với giao diện người dùng.

## Cấu hình Item Media Distribution

Item Media Distribution sử dụng Item Media Bucket làm origin.

Distribution được cấu hình:

```text
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Cached methods: GET, HEAD
Compress objects automatically: Enabled
Price class: PriceClass_100
IPv6: Enabled
```

Distribution này dùng để phân phối hình ảnh và nội dung media của các vật phẩm đấu giá mà không cần công khai trực tiếp Item Media Bucket.

## Cấu hình Cache Behavior

Default Cache Behavior của các Distribution cho phép:

```text
GET
HEAD
```

Các phương thức được lưu vào cache gồm:

```text
GET
HEAD
```

Cấu hình hiện tại không chuyển tiếp query string và cookie đến S3 Origin:

```text
Query strings: None
Cookies: None
```

Cấu hình này phù hợp với nội dung tĩnh vì S3 trả cùng một đối tượng dựa trên đường dẫn yêu cầu.

Việc sử dụng cache giúp:

* Giảm số lượng yêu cầu gửi trực tiếp đến S3.
* Giảm độ trễ khi tải giao diện.
* Tăng tốc độ phân phối hình ảnh vật phẩm.
* Giảm tải cho origin.
* Cải thiện trải nghiệm truy cập của người dùng.

## Chuyển hướng HTTPS

Các Distribution được cấu hình:

```text
Viewer protocol policy: Redirect HTTP to HTTPS
```

Khi người dùng gửi yêu cầu bằng HTTP, CloudFront tự động chuyển hướng sang HTTPS.

Việc sử dụng HTTPS giúp:

* Mã hóa dữ liệu giữa trình duyệt và CloudFront.
* Hạn chế nghe lén dữ liệu trên đường truyền.
* Bảo đảm nội dung không bị thay đổi trong quá trình truyền tải.
* Tăng mức độ an toàn khi truy cập hệ thống.

Hệ thống hiện sử dụng chứng chỉ mặc định của CloudFront cho các domain có dạng:

```text
xxxxxxxxxxxxxx.cloudfront.net
```

## Hỗ trợ Single Page Application

User Frontend và Admin Frontend là các ứng dụng frontend dạng Single Page Application.

Khi người dùng truy cập trực tiếp một đường dẫn, chẳng hạn:

```text
/profile
/my-auctions
/admin/users
```

S3 có thể không tìm thấy đối tượng tương ứng và trả về lỗi `403` hoặc `404`.

Để frontend router tiếp tục xử lý đường dẫn, hai frontend Distribution được cấu hình Custom Error Response:

| Error Code | Response Code | Response Page |
| ---------- | ------------- | ------------- |
| `403`      | `200`         | `/index.html` |
| `404`      | `200`         | `/index.html` |

CloudFront trả tệp `index.html` cho trình duyệt, sau đó frontend router hiển thị trang phù hợp.

Item Media Distribution không sử dụng cơ chế này vì yêu cầu media phải trỏ đến đối tượng thực tế trong S3.

## Kiểm tra CloudFront trên AWS Management Console

### Bước 1: Truy cập Amazon CloudFront

Đăng nhập vào **AWS Management Console**.

Tại thanh tìm kiếm phía trên, nhập:

```text
CloudFront
```

Chọn **CloudFront — Content Delivery Network**.

CloudFront là dịch vụ toàn cầu nên danh sách Distribution không phụ thuộc hoàn toàn vào Region đang được chọn trên thanh điều hướng AWS.

### Bước 2: Kiểm tra danh sách Distribution

Tại menu bên trái, chọn:

```text
Distributions
```

Kiểm tra ba Distribution của hệ thống:

* User Frontend Distribution.
* Admin Frontend Distribution.
* Item Media Distribution.

Các nội dung cần kiểm tra gồm:

* Distribution ID.
* Description.
* Status.
* Last modified.
* Distribution domain name.
* Origin tương ứng.
* Trạng thái Enabled.

Trạng thái triển khai cần hiển thị:

```text
Enabled
Deployed
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-distribution-list.png"
    title="Hình 5.4.3.1: Các CloudFront Distribution của hệ thống Live Auction"
    width="100%"
>}}

### Bước 3: Kiểm tra User Frontend Distribution

Chọn Distribution có Description tương ứng với User Frontend.

Trong tab:

```text
General
```

Kiểm tra:

* Distribution status.
* Distribution domain name.
* Distribution ID.
* Price class.
* Supported HTTP versions.
* IPv6 configuration.
* Default root object.

Default root object phải là:

```text
index.html
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-user-frontend-general.png"
    title="Hình 5.4.3.2: Thông tin User Frontend CloudFront Distribution"
    width="100%"
>}}

### Bước 4: Kiểm tra Origin của User Frontend

Trong User Frontend Distribution, mở tab:

```text
Origins
```

Kiểm tra:

* Origin domain trỏ đến User Frontend S3 Bucket.
* Origin type là Amazon S3.
* Origin Access Control đã được cấu hình.
* Origin path nếu được sử dụng.
* Origin không trỏ nhầm sang Admin Frontend Bucket hoặc Media Bucket.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-user-frontend-origin.png"
    title="Hình 5.4.3.3: S3 Origin của User Frontend Distribution"
    width="100%"
>}}

### Bước 5: Kiểm tra Cache Behavior

Trong Distribution, mở tab:

```text
Behaviors
```

Chọn Default Behavior và kiểm tra:

* Path pattern.
* Origin.
* Viewer protocol policy.
* Allowed HTTP methods.
* Cache HTTP methods.
* Compress objects automatically.
* Query string forwarding.
* Cookie forwarding.

Các giá trị cần xác nhận gồm:

```text
Path pattern: Default (*)
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Cached methods: GET, HEAD
Compress objects automatically: Yes
```


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-cache-behavior.png"
    title="Hình 5.4.3.4: Default Cache Behavior của CloudFront Distribution"
    width="100%"
>}}

### Bước 6: Kiểm tra Custom Error Response

Trong User Frontend Distribution, mở tab:

```text
Error pages
```

Kiểm tra hai cấu hình:

```text
403 → 200 → /index.html
404 → 200 → /index.html
```

Hai cấu hình này cho phép frontend router xử lý các đường dẫn của Single Page Application.

Admin Frontend Distribution cũng cần có hai cấu hình tương tự.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-custom-error-responses.png"
    title="Hình 5.4.3.5: Custom Error Response hỗ trợ Single Page Application"
    width="100%"
>}}

### Bước 7: Kiểm tra Admin Frontend Distribution

Quay lại danh sách Distribution và chọn Admin Frontend Distribution.

Kiểm tra:

* Distribution đang ở trạng thái Enabled và Deployed.
* Origin trỏ đến Admin Frontend S3 Bucket.
* Default root object là `index.html`.
* Viewer protocol policy chuyển hướng HTTP sang HTTPS.
* Allowed methods là `GET` và `HEAD`.
* Custom Error Response đã được cấu hình cho `403` và `404`.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-admin-frontend.png"
    title="Hình 5.4.3.6: CloudFront Distribution dành cho Admin Frontend"
    width="100%"
>}}

### Bước 8: Kiểm tra Item Media Distribution

Chọn Item Media Distribution và mở tab:

```text
Origins
```

Kiểm tra:

* Origin trỏ đến Item Media Bucket.
* Origin Access Control đã được gán.
* Distribution đang ở trạng thái Enabled.
* Viewer protocol policy chuyển hướng HTTP sang HTTPS.
* Allowed methods là `GET` và `HEAD`.
* IPv6 đã được bật.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-item-media.png"
    title="Hình 5.4.3.7: CloudFront Distribution dành cho Item Media"
    width="100%"
>}}

### Bước 9: Kiểm tra Origin Access Control

Tại menu CloudFront bên trái, chọn:

```text
Origin access
```

Kiểm tra các OAC của hệ thống.

Các nội dung cần xác nhận:

* OAC dành cho User Frontend.
* OAC dành cho Admin Frontend.
* OAC dành cho Item Media.
* Origin type là S3.
* Signing behavior là Always.
* Signing protocol là SigV4.


{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.3-CloudFront/cloudfront-origin-access-control.png"
    title="Hình 5.4.3.8: Origin Access Control của các S3 Origin"
    width="100%"
>}}

## Kiểm tra truy cập qua CloudFront

Sau khi kiểm tra cấu hình trên AWS Management Console, có thể mở CloudFront Domain Name của User Frontend và Admin Frontend trên trình duyệt.

Ví dụ:

```text
https://xxxxxxxxxxxxxx.cloudfront.net
```

Kết quả mong đợi:

* User Frontend hiển thị giao diện dành cho người dùng.
* Admin Frontend hiển thị giao diện đăng nhập quản trị.
* Trình duyệt sử dụng kết nối HTTPS.
* Tải lại một đường dẫn con của frontend không trả lỗi XML từ S3.
* Hình ảnh vật phẩm được tải thông qua Media CloudFront Domain.

Phần kiểm thử đầy đủ các chức năng frontend được trình bày tại mục **5.5 — Kiểm thử hệ thống**.

## Kết quả

Sau khi kiểm tra trực tiếp trên AWS Management Console, nhóm ghi nhận:

* Ba CloudFront Distribution đã được Terraform tạo thành công.
* User Frontend và Admin Frontend được phân phối thông qua hai Distribution độc lập.
* Item Media được phân phối thông qua một Distribution riêng.
* Các Distribution đang ở trạng thái Enabled và Deployed.
* Mỗi Distribution trỏ đến đúng S3 Origin.
* Origin Access Control đã được cấu hình cho các S3 Bucket private.
* Các yêu cầu HTTP được chuyển hướng sang HTTPS.
* Các phương thức `GET` và `HEAD` được cho phép và lưu vào cache.
* Tính năng nén nội dung đã được bật.
* User Frontend và Admin Frontend sử dụng `index.html` làm Default Root Object.
* Custom Error Response đã được cấu hình để hỗ trợ frontend dạng Single Page Application.
* S3 Bucket vẫn được giữ private trong khi nội dung được phân phối thông qua CloudFront.
* Hệ thống đã sẵn sàng cung cấp hai giao diện frontend và nội dung media đến người dùng.

Việc triển khai và kiểm tra các Lambda Function xử lý nghiệp vụ được trình bày tại mục **5.4.4 — AWS Lambda**.