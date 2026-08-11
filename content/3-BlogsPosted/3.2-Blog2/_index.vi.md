---
title: "Blog 2 - Lambda scale nhanh, nhưng Database không scale theo cùng cách"
date: 2026-08-09
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# Lambda scale nhanh, nhưng Database không scale theo cùng cách

## Tổng quan

Trong quá trình phát triển hệ thống **Live Auction**, nhóm từng nghiên cứu việc chuyển backend từ mô hình **FastAPI** sang kiến trúc **serverless** sử dụng **AWS Lambda**, trong khi cơ sở dữ liệu vẫn được lưu trữ trên **Amazon RDS for MySQL**.

Việc chuyển sang AWS Lambda giúp hệ thống có khả năng tự động mở rộng theo số lượng yêu cầu. Tuy nhiên, khả năng mở rộng nhanh của Lambda cũng đặt ra một vấn đề quan trọng: cơ sở dữ liệu phía sau không nhất thiết có thể mở rộng số lượng kết nối và tài nguyên với cùng tốc độ.

Bài viết phân tích vấn đề quản lý kết nối cơ sở dữ liệu khi AWS Lambda xử lý nhiều yêu cầu đồng thời, đồng thời giới thiệu **Reserved Concurrency** và **Amazon RDS Proxy** như những giải pháp có thể được cân nhắc để bảo vệ cơ sở dữ liệu.

## Mô hình kết nối cơ sở dữ liệu truyền thống

Với backend chạy lâu dài trên **Amazon EC2** hoặc container, ứng dụng thường duy trì một nhóm kết nối đến cơ sở dữ liệu.

Luồng kết nối thường có dạng:

```text
Người dùng
    ↓
Backend chạy trên EC2 hoặc Container
    ↓
Connection Pool
    ↓
Amazon RDS for MySQL
```

`Connection Pool` duy trì một số lượng kết nối giới hạn và tái sử dụng chúng giữa nhiều yêu cầu. Nhờ đó, ứng dụng không phải tạo một kết nối cơ sở dữ liệu mới cho từng request.

Mô hình này tương đối dễ kiểm soát vì:

* Số lượng phiên bản backend thường ổn định.
* Số lượng kết nối tối đa có thể được cấu hình trước.
* Các kết nối được duy trì và tái sử dụng trong thời gian dài.
* Database có thể dự đoán tương đối ổn định số lượng kết nối cần xử lý.

## Điều gì thay đổi khi sử dụng AWS Lambda?

AWS Lambda thực thi mã nguồn bên trong các **execution environment**. Khi số lượng yêu cầu đồng thời tăng, Lambda có thể tạo thêm nhiều execution environment để xử lý workload.

Ví dụ, tại thời điểm cuối phiên đấu giá, hệ thống có thể nhận:

```text
500 yêu cầu đồng thời
        ↓
Nhiều Lambda execution đồng thời
        ↓
Nhiều kết nối mới đến Database
```

Nếu mỗi execution tạo một kết nối trực tiếp đến Amazon RDS, số lượng kết nối có thể tăng rất nhanh trong thời gian ngắn.

Trong khi Lambda có khả năng mở rộng theo workload, Amazon RDS vẫn bị giới hạn bởi:

* Cấu hình của DB Instance.
* Số lượng kết nối tối đa.
* Tài nguyên CPU và bộ nhớ.
* Khả năng xử lý transaction.
* Khả năng đọc và ghi dữ liệu.
* Thời gian thiết lập kết nối mới.

Do đó, việc Lambda vẫn tiếp tục mở rộng không có nghĩa cơ sở dữ liệu phía sau cũng có thể xử lý cùng một mức tải.

## Nguy cơ khi số lượng kết nối tăng đột ngột

Giả sử hệ thống nhận 500 yêu cầu đặt giá đồng thời và mỗi Lambda execution mở một kết nối mới đến MySQL.

Luồng xử lý có thể trở thành:

```text
500 yêu cầu đồng thời
        ↓
500 Lambda execution
        ↓
Tối đa 500 yêu cầu kết nối mới
        ↓
Amazon RDS for MySQL
```

Tùy thuộc vào cách Lambda tái sử dụng execution environment và kết nối, số lượng kết nối thực tế có thể khác. Tuy nhiên, việc mở rộng đột ngột vẫn có khả năng gây áp lực lớn lên cơ sở dữ liệu.

Một số vấn đề có thể xảy ra gồm:

* Database hết kết nối khả dụng.
* Thời gian tạo kết nối tăng.
* Độ trễ của API tăng.
* Transaction được xử lý chậm.
* Yêu cầu bị timeout.
* Các workload khác sử dụng chung database bị ảnh hưởng.
* Hệ thống phát sinh lỗi trong thời điểm có lượng truy cập cao.

Đối với hệ thống đấu giá trực tuyến, rủi ro này đặc biệt quan trọng vì lưu lượng đặt giá thường có thể tăng mạnh trong thời điểm gần kết thúc phiên.

{{% notice note %}}
Khả năng mở rộng của hệ thống serverless không nên chỉ được đánh giá dựa trên AWS Lambda. Nhóm cần xem xét cả giới hạn và khả năng chịu tải của những dịch vụ phụ thuộc phía sau Lambda.
{{% /notice %}}

## Giới hạn Lambda bằng Reserved Concurrency

Một giải pháp có thể được sử dụng để bảo vệ dịch vụ phía sau là cấu hình **Reserved Concurrency** cho Lambda.

Reserved Concurrency giúp:

* Giới hạn số lượng execution đồng thời của một Lambda Function.
* Ngăn Lambda mở rộng vượt quá khả năng chịu tải của database.
* Dành trước một phần concurrency cho function quan trọng.
* Hạn chế một function sử dụng toàn bộ concurrency của tài khoản.
* Giúp hệ thống kiểm soát tải gửi đến downstream service.

Luồng xử lý sau khi giới hạn concurrency có thể được mô tả như sau:

```text
Lượng request tăng
        ↓
AWS Lambda
        ↓
Giới hạn bởi Reserved Concurrency
        ↓
Amazon RDS nhận số lượng kết nối có kiểm soát
```

Tuy nhiên, việc giới hạn concurrency cũng có thể làm một số yêu cầu bị trì hoãn hoặc throttling khi lượng truy cập vượt quá giới hạn. Vì vậy, giá trị concurrency cần được xác định dựa trên khả năng xử lý thực tế của database và yêu cầu của hệ thống.

## Amazon RDS Proxy

Một giải pháp khác được AWS cung cấp cho ứng dụng serverless kết nối đến Amazon RDS là **Amazon RDS Proxy**.

Thay vì để Lambda kết nối trực tiếp đến cơ sở dữ liệu:

```text
AWS Lambda
    ↓
Amazon RDS
```

Hệ thống có thể sử dụng kiến trúc:

```text
AWS Lambda
    ↓
Amazon RDS Proxy
    ↓
Amazon RDS for MySQL
```

RDS Proxy nằm giữa ứng dụng và database, duy trì một nhóm kết nối đến cơ sở dữ liệu và cho phép các Lambda execution sử dụng lại những kết nối này.

## Cách RDS Proxy hỗ trợ quản lý kết nối

Khi nhiều Lambda execution xuất hiện cùng lúc, mục tiêu không phải lúc nào cũng là tạo số lượng kết nối database tương ứng với số lượng execution.

RDS Proxy hỗ trợ:

* Duy trì connection pool đến database.
* Tái sử dụng kết nối giữa nhiều yêu cầu.
* Giảm số lần thiết lập kết nối mới.
* Hạn chế áp lực kết nối trực tiếp lên Amazon RDS.
* Hỗ trợ ứng dụng xử lý các đợt tăng traffic khó dự đoán.
* Cải thiện khả năng phục hồi khi database thực hiện failover.
* Quản lý thông tin xác thực database an toàn hơn khi kết hợp với AWS Secrets Manager và IAM.

Luồng xử lý tổng quát:

```text
Nhiều Lambda execution
          ↓
     Amazon RDS Proxy
          ↓
Pool kết nối được quản lý
          ↓
 Amazon RDS for MySQL
```

Ví dụ, khi có 300 Lambda execution hoạt động đồng thời, RDS Proxy có thể giúp quản lý và tái sử dụng các kết nối phù hợp thay vì yêu cầu database thiết lập ngay một kết nối hoàn toàn mới cho từng execution.

## RDS Proxy không giải quyết mọi giới hạn của Database

RDS Proxy chủ yếu giải quyết bài toán **quản lý kết nối**. Dịch vụ này không tự động làm tăng toàn bộ khả năng xử lý của cơ sở dữ liệu.

RDS Proxy không thể thay thế việc:

* Tối ưu câu truy vấn SQL.
* Thiết kế index phù hợp.
* Giảm thời gian thực hiện transaction.
* Chọn DB Instance có đủ CPU và bộ nhớ.
* Tối ưu cấu trúc dữ liệu.
* Thiết lập timeout hợp lý.
* Theo dõi hiệu năng và số lượng kết nối.
* Kiểm soát lượng request mà database phải xử lý.

Nếu truy vấn chậm hoặc database không đủ tài nguyên, việc thêm RDS Proxy sẽ không tự động giải quyết hoàn toàn vấn đề.


## Áp dụng vào hệ thống Live Auction

Trong hệ thống đấu giá, lưu lượng truy cập có thể không phân bố đều. Thời điểm gần kết thúc phiên thường có nhiều người dùng gửi yêu cầu đặt giá trong khoảng thời gian ngắn.

Nếu backend sử dụng Lambda và kết nối trực tiếp đến RDS, hệ thống cần xem xét:

* Số lượng người dùng có thể đặt giá đồng thời.
* Concurrency tối đa của Lambda Function.
* Giới hạn kết nối của DB Instance.
* Thời gian xử lý một transaction đặt giá.
* Cơ chế chống xử lý trùng yêu cầu.
* Khả năng duy trì đúng thứ tự đặt giá.
* Phương án xử lý throttling và timeout.
* Cách theo dõi số lượng kết nối đang sử dụng.

Kiến trúc có thể được cân nhắc:

```text
Người dùng
    ↓
Amazon API Gateway
    ↓
AWS Lambda
    ↓
Amazon RDS Proxy
    ↓
Amazon RDS for MySQL
```

Bên cạnh RDS Proxy, hệ thống có thể kết hợp thêm:

* **Reserved Concurrency** để kiểm soát số Lambda execution.
* **Amazon CloudWatch** để theo dõi lỗi, latency và concurrency.
* **Amazon SQS** để làm bộ đệm cho các workload phù hợp.
* Cơ chế retry có kiểm soát.
* Timeout và connection timeout hợp lý.
* Transaction và khóa dữ liệu phù hợp với nghiệp vụ đấu giá.

Bài viết này trình bày một hướng nghiên cứu kiến trúc trong quá trình phát triển dự án. Kiến trúc triển khai cuối cùng của hệ thống Live Auction có thể sử dụng các dịch vụ lưu trữ và xử lý khác tùy theo yêu cầu thực tế.

## So sánh hai phương án kết nối

| Tiêu chí | Lambda kết nối trực tiếp RDS | Lambda kết nối qua RDS Proxy |
| --- | --- | --- |
| Quản lý kết nối | Mỗi execution tự quản lý kết nối | Proxy quản lý và tái sử dụng kết nối |
| Khả năng chịu tăng tải | Dễ tạo nhiều kết nối trong thời gian ngắn | Giảm áp lực thiết lập kết nối trực tiếp |
| Connection Pool | Phụ thuộc vào execution environment | Được quản lý tập trung bởi Proxy |
| Khả năng failover | Ứng dụng tự xử lý việc kết nối lại | Proxy hỗ trợ cải thiện quá trình kết nối lại |
| Độ phức tạp | Kiến trúc đơn giản hơn | Cần cấu hình thêm Proxy, IAM và Secrets |
| Chi phí | Không có chi phí của Proxy | Phát sinh thêm chi phí sử dụng RDS Proxy |
| Trường hợp phù hợp | Workload nhỏ và concurrency ổn định | Workload serverless có concurrency biến động |

## Bài học rút ra

Qua quá trình nghiên cứu, nhóm rút ra một số bài học:

* Serverless scaling cần được đánh giá trên toàn bộ kiến trúc.
* Lambda scale nhanh không có nghĩa database cũng scale theo cùng cách.
* Cần kiểm soát số lượng kết nối tạo ra từ các Lambda execution.
* Reserved Concurrency có thể bảo vệ downstream service khỏi tải quá lớn.
* RDS Proxy phù hợp với workload có nhiều kết nối ngắn hoặc concurrency biến động.
* RDS Proxy giải quyết bài toán connection management, không thay thế việc tối ưu database.
* Hệ thống đấu giá cần đặc biệt chú ý đến những đợt tăng traffic gần thời điểm kết thúc phiên.
* Việc giám sát concurrency, connection count, latency và error rate là cần thiết trước khi đưa hệ thống vào vận hành.

## Bài viết đã đăng

Bài viết được nhóm chia sẻ trên cộng đồng **AWS Study Group – First Cloud Journey**.

**Tiêu đề:** Lambda scale nhanh, nhưng Database không scale theo cùng cách

**Liên kết bài viết:** [Xem bài viết trên AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/posts/2238842566880703/)

## Kết quả đạt được

Sau khi hoàn thành bài viết, nhóm đã:

* Hiểu rõ hơn sự khác biệt giữa khả năng mở rộng của Lambda và Amazon RDS.
* Nhận biết nguy cơ connection exhaustion trong kiến trúc serverless.
* Tìm hiểu cách sử dụng Reserved Concurrency để bảo vệ downstream service.
* Hiểu vai trò của Amazon RDS Proxy trong việc quản lý và tái sử dụng kết nối.
* Liên hệ vấn đề quản lý kết nối với tình huống tăng tải của hệ thống đấu giá.
* Chia sẻ kiến thức đã nghiên cứu với cộng đồng AWS Study Group.

**Hashtag:** `#AWS` `#AWSLambda` `#AmazonRDS` `#RDSProxy` `#Serverless` `#CloudEngineering` `#SystemDesign`