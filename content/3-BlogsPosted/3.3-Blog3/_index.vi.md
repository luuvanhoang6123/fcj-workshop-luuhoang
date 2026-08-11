---
title: "Blog 3 - Tối ưu chi phí lưu trữ trên Amazon S3"
date: 2026-08-09
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# Tối ưu chi phí lưu trữ trên Amazon S3: Tìm hiểu về Storage Classes và Lifecycle Policy

## Tổng quan

Khi mới tìm hiểu AWS, Amazon S3 thường được sử dụng theo cách đơn giản: tải tệp lên bucket, lưu trữ và tải xuống khi cần thiết.

Tuy nhiên, dữ liệu trong một hệ thống không phải lúc nào cũng có cùng tần suất truy cập và mức độ quan trọng. Một số tệp cần được sử dụng thường xuyên, trong khi những tệp khác chỉ cần lưu trữ để sao lưu, kiểm tra hoặc đáp ứng yêu cầu pháp lý.

Nếu tất cả dữ liệu đều được lưu trong **S3 Standard** trong suốt vòng đời, hệ thống có thể phải trả mức phí lưu trữ cao cho những dữ liệu hiếm khi được truy cập.

Amazon S3 cung cấp nhiều **Storage Class** và tính năng **Lifecycle Policy**, giúp lựa chọn tầng lưu trữ phù hợp và tự động chuyển dữ liệu sang lớp có chi phí thấp hơn theo thời gian.

{{% notice note %}}
Mức chi phí tiết kiệm được phụ thuộc vào Region, dung lượng, kích thước object, tần suất truy cập, phí truy xuất và thời gian lưu trữ. Vì vậy, không có một tỷ lệ tiết kiệm cố định phù hợp với mọi hệ thống.
{{% /notice %}}

## Vấn đề: Không phải dữ liệu nào cũng được truy cập giống nhau

Giả sử một S3 Bucket đang lưu trữ:

* Log của ứng dụng.
* Hình ảnh do người dùng tải lên.
* Tệp backup cơ sở dữ liệu.
* Báo cáo kiểm tra hệ thống.
* Tài liệu cần lưu giữ lâu dài.

Tần suất truy cập của các dữ liệu này có thể khác nhau:

```text
Log trong tuần hiện tại
→ Được truy cập thường xuyên để theo dõi và xử lý lỗi.

Log của tháng trước
→ Chỉ thỉnh thoảng được truy cập khi cần kiểm tra.

Log của sáu tháng trước
→ Gần như không được sử dụng nhưng vẫn phải lưu để audit.

Tệp backup cũ
→ Chỉ được truy xuất khi cần khôi phục hệ thống.
```

Nếu tất cả những dữ liệu trên đều nằm trong S3 Standard, hệ thống vẫn phải trả phí cho lớp lưu trữ dành cho dữ liệu thường xuyên được truy cập, kể cả khi phần lớn object gần như không còn được sử dụng.

Do đó, việc lựa chọn Storage Class cần dựa trên:

* Tần suất truy cập dữ liệu.
* Thời gian cần lưu giữ.
* Thời gian có thể chờ khi khôi phục.
* Khả năng tái tạo dữ liệu.
* Yêu cầu về độ bền và tính sẵn sàng.
* Chi phí lưu trữ và truy xuất.
* Yêu cầu kiểm toán hoặc pháp lý.

## Tổng quan các S3 Storage Class

Amazon S3 cung cấp nhiều lớp lưu trữ để đáp ứng các loại workload khác nhau.

| Storage Class | Tần suất truy cập | Thời gian truy xuất | Trường hợp sử dụng |
| --- | --- | --- | --- |
| **S3 Standard** | Thường xuyên | Tức thì | Website, ứng dụng và dữ liệu đang hoạt động |
| **S3 Intelligent-Tiering** | Không xác định hoặc thay đổi | Tức thì đối với các tầng truy cập trực tuyến | Dữ liệu có mô hình truy cập khó dự đoán |
| **S3 Standard-IA** | Không thường xuyên | Tức thì | Backup và dữ liệu cần truy xuất nhanh |
| **S3 One Zone-IA** | Không thường xuyên | Tức thì | Dữ liệu có thể tái tạo và chỉ cần lưu trong một AZ |
| **S3 Glacier Instant Retrieval** | Hiếm khi | Tức thì | Dữ liệu lưu trữ ít dùng nhưng phải truy xuất ngay |
| **S3 Glacier Flexible Retrieval** | Rất hiếm khi | Từ vài phút đến nhiều giờ | Backup và dữ liệu lưu trữ dài hạn |
| **S3 Glacier Deep Archive** | Gần như không truy cập | Có thể mất nhiều giờ | Dữ liệu lưu giữ lâu dài và tuân thủ pháp lý |

## S3 Standard

**S3 Standard** được thiết kế cho dữ liệu được truy cập thường xuyên và yêu cầu độ trễ thấp.

Storage Class này phù hợp với:

* Tài nguyên tĩnh của website.
* Hình ảnh được truy cập thường xuyên.
* Dữ liệu của ứng dụng đang hoạt động.
* Tệp cần đọc và ghi thường xuyên.
* Nội dung được phân phối qua Amazon CloudFront.

Ưu điểm:

* Độ trễ truy cập thấp.
* Thông lượng cao.
* Không có thời gian chờ khôi phục dữ liệu.
* Phù hợp với nhiều loại ứng dụng.

Nhược điểm chính là chi phí lưu trữ thường cao hơn các lớp dành cho dữ liệu ít được truy cập.

## S3 Intelligent-Tiering

**S3 Intelligent-Tiering** phù hợp với dữ liệu có mô hình truy cập thay đổi hoặc khó dự đoán.

Dịch vụ theo dõi hoạt động truy cập và tự động di chuyển object giữa các tầng truy cập thích hợp. Nhờ đó, người quản trị không phải tự xác định chính xác thời điểm chuyển từng object.

Lớp lưu trữ này phù hợp khi:

* Không biết dữ liệu sẽ được truy cập thường xuyên hay không.
* Tần suất truy cập thay đổi theo thời gian.
* Muốn tự động tối ưu chi phí dựa trên hoạt động truy cập.
* Không muốn xây dựng nhiều Lifecycle Rule phức tạp.

S3 Intelligent-Tiering có phí giám sát và tự động hóa cho từng object. Vì vậy, cần xem xét số lượng và kích thước object trước khi lựa chọn.

## S3 Standard-IA

**S3 Standard-Infrequent Access (Standard-IA)** dành cho dữ liệu ít được truy cập nhưng vẫn cần lấy lại nhanh khi cần.

Storage Class này phù hợp với:

* Tệp backup.
* Dữ liệu disaster recovery.
* Tài liệu cũ thỉnh thoảng cần truy xuất.
* Dữ liệu không còn được sử dụng thường xuyên.

S3 Standard-IA có chi phí lưu trữ thấp hơn S3 Standard nhưng phát sinh phí khi truy xuất dữ liệu.

Vì vậy, nếu một object được truy cập thường xuyên, tổng chi phí có thể cao hơn so với việc tiếp tục lưu trong S3 Standard.

## S3 One Zone-IA

**S3 One Zone-IA** có đặc điểm tương tự Standard-IA nhưng dữ liệu chỉ được lưu trong một Availability Zone.

Ưu điểm:

* Chi phí lưu trữ thấp hơn Standard-IA.
* Dữ liệu vẫn được truy xuất với độ trễ thấp.

Hạn chế:

* Không có khả năng phục hồi qua nhiều Availability Zone như Standard-IA.
* Không phù hợp với bản dữ liệu duy nhất có mức độ quan trọng cao.

Storage Class này nên được sử dụng cho:

* Dữ liệu có thể tái tạo.
* Bản sao thứ hai của dữ liệu.
* Thumbnail hoặc tài nguyên có thể tạo lại.
* Dữ liệu không yêu cầu khả năng phục hồi qua nhiều AZ.


## S3 Glacier Instant Retrieval

**S3 Glacier Instant Retrieval** dành cho dữ liệu hiếm khi được truy cập nhưng phải được cung cấp ngay khi có yêu cầu.

Một số trường hợp sử dụng:

* Hình ảnh y tế.
* Dữ liệu media được lưu trữ lâu dài.
* Tài liệu cần truy xuất ngay nhưng ít được sử dụng.
* Dữ liệu archive có yêu cầu độ trễ thấp.

Lớp này có chi phí lưu trữ thấp hơn các lớp truy cập thường xuyên nhưng có phí truy xuất và yêu cầu thời gian lưu trữ tối thiểu.

## S3 Glacier Flexible Retrieval

**S3 Glacier Flexible Retrieval** phù hợp với dữ liệu archive không cần truy xuất ngay lập tức.

Tùy theo phương án truy xuất, thời gian lấy dữ liệu có thể từ vài phút đến nhiều giờ.

Các trường hợp phù hợp gồm:

* Backup định kỳ.
* Dữ liệu lưu trữ dài hạn.
* Log cũ.
* Tệp phục vụ kiểm toán.
* Dữ liệu chỉ được khôi phục trong trường hợp đặc biệt.

Đây là lựa chọn phù hợp khi ưu tiên giảm chi phí lưu trữ và có thể chấp nhận thời gian chờ để khôi phục dữ liệu.

## S3 Glacier Deep Archive

**S3 Glacier Deep Archive** là lớp lưu trữ dành cho dữ liệu gần như không được truy cập và được giữ lại trong thời gian dài.

Lớp này phù hợp với:

* Dữ liệu phải lưu giữ theo quy định pháp lý.
* Hồ sơ lưu trữ nhiều năm.
* Bản backup dài hạn.
* Dữ liệu phục vụ kiểm toán.
* Dữ liệu thay thế cho hệ thống lưu trữ băng từ.

Glacier Deep Archive có chi phí lưu trữ thấp, nhưng thời gian khôi phục có thể kéo dài nhiều giờ. Vì vậy, lớp này không phù hợp với dữ liệu phải được truy xuất ngay.

## S3 Lifecycle Policy

Việc chuyển từng object sang Storage Class khác theo cách thủ công sẽ tốn thời gian và dễ xảy ra sai sót.

**S3 Lifecycle Policy** cho phép tạo các quy tắc để Amazon S3 tự động quản lý object trong suốt vòng đời của chúng.

Lifecycle Policy có thể thực hiện:

* Chuyển object sang Storage Class khác.
* Quản lý các phiên bản cũ của object.
* Xóa object khi hết thời hạn lưu giữ.
* Xóa phiên bản cũ không còn cần thiết.
* Xóa incomplete multipart upload.
* Áp dụng quy tắc dựa trên prefix.
* Áp dụng quy tắc dựa trên object tag.

Việc chuyển đổi và xóa dữ liệu được Amazon S3 thực hiện tự động, không cần xây dựng cron job hoặc Lambda Function riêng.

## Ví dụ về một Lifecycle Policy

Một chính sách lưu trữ log có thể được thiết kế như sau:

```text
Ngày 0–30
→ Lưu trong S3 Standard để thường xuyên kiểm tra và xử lý lỗi.

Sau 30 ngày
→ Chuyển sang S3 Standard-IA.

Sau 90 ngày
→ Chuyển sang S3 Glacier Flexible Retrieval.

Sau 365 ngày
→ Chuyển sang S3 Glacier Deep Archive.

Sau 7 năm
→ Tự động xóa object nếu đã hết thời hạn lưu giữ.
```

Luồng chuyển đổi:

```text
S3 Standard
     ↓ Sau 30 ngày
S3 Standard-IA
     ↓ Sau 90 ngày
S3 Glacier Flexible Retrieval
     ↓ Sau 365 ngày
S3 Glacier Deep Archive
     ↓ Sau 7 năm
Expiration
```

Các mốc thời gian trên chỉ là ví dụ. Khi triển khai thực tế, cần lựa chọn thời gian dựa trên yêu cầu truy cập, quy định lưu giữ dữ liệu và chi phí của hệ thống.

## Ví dụ cấu hình Lifecycle bằng Terraform

Lifecycle Policy có thể được quản lý bằng Terraform để bảo đảm cấu hình nhất quán giữa các lần triển khai.

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "log_lifecycle" {
  bucket = aws_s3_bucket.application_logs.id

  rule {
    id     = "archive-application-logs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }

    expiration {
      days = 2555
    }
  }
}
```

Trong ví dụ trên:

* Quy tắc chỉ áp dụng cho object có prefix `logs/`.
* Object được chuyển sang Standard-IA sau 30 ngày.
* Object được chuyển sang Glacier Flexible Retrieval sau 90 ngày.
* Object được chuyển sang Deep Archive sau 365 ngày.
* Object được xóa sau 2.555 ngày, tương đương khoảng 7 năm.
 trước khi áp dụng vào hệ thống thực tế.


## Áp dụng Lifecycle theo Prefix hoặc Tag

Không nhất thiết phải áp dụng cùng một Lifecycle Policy cho toàn bộ bucket.

Giả sử một bucket có cấu trúc:

```text
application-data/
├── logs/
├── backups/
├── user-images/
└── reports/
```

Có thể xây dựng các quy tắc riêng:

```text
logs/
→ Chuyển sang Glacier sau một khoảng thời gian và xóa khi hết hạn.

backups/
→ Chuyển sang Deep Archive để lưu giữ lâu dài.

user-images/
→ Tiếp tục lưu trong Standard hoặc Intelligent-Tiering.

reports/
→ Chuyển sang Standard-IA khi ít được sử dụng.
```

Ngoài prefix, Lifecycle Rule có thể sử dụng object tag:

```text
DataType = ApplicationLog
Retention = 365Days
Environment = Production
```

Việc phân loại bằng prefix hoặc tag giúp quản lý nhiều nhóm dữ liệu trong cùng một bucket mà vẫn áp dụng được chính sách riêng.

## Những lưu ý khi tối ưu chi phí

### Chi phí chuyển đổi và truy xuất

Chuyển object sang Storage Class khác có thể phát sinh phí request. Các lớp IA và Glacier cũng có thể phát sinh phí truy xuất dữ liệu.

Vì vậy, không nên tạo quá nhiều lần chuyển đổi trong khoảng thời gian quá ngắn nếu mức tiết kiệm lưu trữ không đủ bù chi phí request và retrieval.

### Kích thước object

Các object nhỏ có thể không mang lại hiệu quả tiết kiệm như dự kiến do:

* Chi phí request khi chuyển đổi.
* Kích thước tính phí tối thiểu của một số Storage Class.
* Metadata bổ sung đối với các lớp archive.
* Số lượng object quá lớn làm tăng chi phí quản lý và request.

Theo hành vi mặc định hiện nay của S3 Lifecycle, object nhỏ hơn 128 KB thường không được tự động chuyển đổi bằng Lifecycle Rule, trừ khi cấu hình chuyển đổi kích thước object phù hợp được chỉ định.

### Thời gian lưu trữ tối thiểu

Một số Storage Class có thời gian lưu trữ tối thiểu:

| Storage Class | Thời gian lưu trữ tối thiểu |
| --- | --- |
| S3 Standard-IA | 30 ngày |
| S3 One Zone-IA | 30 ngày |
| S3 Glacier Instant Retrieval | 90 ngày |
| S3 Glacier Flexible Retrieval | 90 ngày |
| S3 Glacier Deep Archive | 180 ngày |

Nếu object bị xóa hoặc chuyển sang lớp khác trước thời gian tối thiểu, hệ thống vẫn có thể phải thanh toán phần thời gian lưu trữ còn lại.

### Thời gian khôi phục

Trước khi chuyển dữ liệu sang Glacier, cần xác định thời gian tối đa mà hệ thống có thể chờ để khôi phục.

Dữ liệu cần được lấy ngay không nên chuyển sang lớp có thời gian khôi phục kéo dài nhiều giờ.

### Phiên bản cũ của Object

Nếu S3 Versioning được bật, việc xóa hoặc ghi đè object không nhất thiết làm phiên bản cũ biến mất.

Các phiên bản không hiện hành vẫn tiếp tục chiếm dung lượng và phát sinh chi phí. Vì vậy, Lifecycle Policy nên xem xét cả:

* Current version.
* Noncurrent version.
* Delete marker.
* Incomplete multipart upload.

### Không xóa dữ liệu trái với yêu cầu lưu giữ

Trước khi cấu hình Expiration, cần kiểm tra:

* Chính sách lưu giữ của doanh nghiệp.
* Yêu cầu audit.
* Quy định pháp lý.
* Yêu cầu khôi phục dữ liệu.
* Chính sách backup.

Không nên tự động xóa dữ liệu chỉ nhằm giảm chi phí nếu chưa xác định rõ thời hạn lưu giữ.

## Kiểm tra chi phí trước khi triển khai

Trước khi áp dụng Lifecycle Policy, nhóm nên:

1. Phân loại dữ liệu theo tần suất truy cập.
2. Xác định dung lượng và số lượng object.
3. Kiểm tra kích thước trung bình của object.
4. Xác định thời gian cần lưu giữ.
5. Xác định thời gian khôi phục có thể chấp nhận.
6. Kiểm tra giá theo AWS Region đang sử dụng.
7. Tính cả phí lưu trữ, request, transition và retrieval.
8. Thử nghiệm chính sách với một prefix nhỏ trước.
9. Theo dõi chi phí sau khi áp dụng.
10. Điều chỉnh Lifecycle Rule khi mô hình truy cập thay đổi.

Có thể sử dụng **AWS Pricing Calculator**, **AWS Cost Explorer** và các chỉ số lưu trữ của Amazon S3 để hỗ trợ quá trình đánh giá.

## Liên hệ với hệ thống Live Auction

Trong hệ thống Live Auction, Amazon S3 có thể được sử dụng để lưu:

* Giao diện frontend đã build.
* Hình ảnh của vật phẩm đấu giá.
* Tài nguyên tĩnh của website.
* Log hoặc báo cáo được xuất ra.
* Tệp backup và dữ liệu lưu trữ dài hạn.

Không phải tất cả dữ liệu đều nên áp dụng cùng một chính sách.

Ví dụ:

| Loại dữ liệu | Phương án đề xuất |
| --- | --- |
| Frontend đang hoạt động | S3 Standard |
| Hình ảnh vật phẩm đang đấu giá | S3 Standard hoặc Intelligent-Tiering |
| Hình ảnh của phiên đã kết thúc | Standard-IA hoặc Intelligent-Tiering |
| Log cũ | Glacier Flexible Retrieval |
| Backup dài hạn | Glacier Deep Archive |
| Tệp tạm | Tự động xóa bằng Expiration Rule |

Đây là phương án nghiên cứu nhằm tối ưu chi phí. Storage Class thực tế cần được lựa chọn dựa trên yêu cầu truy cập, độ quan trọng của dữ liệu và chi phí tại Region triển khai.

## Bài học rút ra

Qua quá trình tìm hiểu, nhóm rút ra các bài học:

* Amazon S3 không chỉ là nơi tải lên và tải xuống tệp.
* Dữ liệu cần được phân loại theo tần suất truy cập và thời gian lưu giữ.
* Không phải object nào cũng nên nằm trong S3 Standard suốt vòng đời.
* Lifecycle Policy giúp tự động chuyển và xóa dữ liệu mà không cần cron job hoặc Lambda.
* Storage Class có chi phí lưu trữ, truy xuất và thời gian lưu tối thiểu khác nhau.
* Object nhỏ cần được đánh giá kỹ trước khi chuyển sang IA hoặc Glacier.
* Prefix và tag giúp áp dụng chính sách riêng cho từng nhóm dữ liệu.
* RDS, Lambda hoặc mã nguồn ứng dụng không phải lúc nào cũng là nơi cần tối ưu đầu tiên; cách lưu trữ dữ liệu cũng có thể tạo ra khoản chi phí đáng kể.

## Bài viết đã đăng

Bài viết được nhóm chia sẻ trên cộng đồng **AWS Study Group – First Cloud Journey**.

**Tiêu đề:** Tối ưu chi phí lưu trữ trên Amazon S3: Tìm hiểu về Storage Classes và Lifecycle Policy

**Liên kết bài viết:** [Xem bài viết trên AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/posts/2239357320162561/)

## Tài liệu tham khảo

* [Tổng quan Amazon S3 Storage Classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)
* [Amazon S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
* [Thiết lập Storage Class cho S3 Object](https://docs.aws.amazon.com/AmazonS3/latest/userguide/sc-howtoset.html)
* [Quản lý vòng đời Object bằng S3 Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
* [Các thành phần của Lifecycle Configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)
* [Thiết lập Lifecycle Configuration cho S3 Bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html)
* [Bảng giá chính thức của Amazon S3](https://aws.amazon.com/s3/pricing/)

## Kết quả đạt được

Sau khi hoàn thành bài viết, nhóm đã:

* Phân biệt được các Storage Class phổ biến của Amazon S3.
* Hiểu cách lựa chọn Storage Class dựa trên tần suất truy cập.
* Hiểu vai trò của Lifecycle Transition và Expiration.
* Biết cách áp dụng Lifecycle Rule theo prefix hoặc object tag.
* Nhận biết các loại chi phí ngoài phí lưu trữ.
* Liên hệ việc tối ưu S3 với dữ liệu của hệ thống Live Auction.
* Chia sẻ kiến thức đã nghiên cứu với cộng đồng AWS Study Group.