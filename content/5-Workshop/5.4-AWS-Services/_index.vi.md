---
title: "Các dịch vụ AWS được triển khai"
date: 2026-07-27
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

## Tổng quan

Sau khi hạ tầng được khởi tạo và triển khai bằng Terraform, nhóm tiến hành tích hợp các dịch vụ AWS để xây dựng hệ thống **Live Auction** theo kiến trúc serverless.

Hệ thống gồm hai giao diện frontend được triển khai độc lập:

* **User Frontend** dành cho người dùng tham gia hệ thống.
* **Admin Frontend** dành cho quản trị viên thực hiện các nghiệp vụ quản lý.

Hai giao diện sử dụng chung các dịch vụ backend, nhưng quyền truy cập và chức năng được phân biệt dựa trên vai trò của tài khoản.

## Chức năng của người dùng

Thông qua User Frontend, người dùng có thể thực hiện các chức năng chính sau:

* Tạo tài khoản và đăng nhập vào hệ thống.
* Xem và cập nhật thông tin cá nhân.
* Xem danh sách và thông tin chi tiết của các phiên đấu giá.
* Xem danh sách các phiên đấu giá do mình tạo.
* Tạo phiên đấu giá mới.
* Thêm một hoặc nhiều vật phẩm vào một phiên đấu giá.
* Theo dõi trạng thái xét duyệt của phiên đấu giá.
* Tham gia các phiên đấu giá đã được phê duyệt.
* Đặt giá cho vật phẩm trong phiên đấu giá.
* Theo dõi giá hiện tại và lịch sử đặt giá.
* Nhận thông tin cập nhật theo thời gian thực trong quá trình đấu giá.

Phiên đấu giá do người dùng tạo phải được quản trị viên xét duyệt trước khi có thể hoạt động chính thức trên hệ thống.

## Chức năng của quản trị viên

Thông qua Admin Frontend, quản trị viên có thể thực hiện các chức năng sau:

* Quản lý tài khoản người dùng.
* Xem thông tin và trạng thái của các tài khoản.
* Quản lý danh mục sản phẩm.
* Xem và kiểm tra các phiên đấu giá đang chờ duyệt.
* Phê duyệt hoặc từ chối phiên đấu giá.
* Tạo thêm tài khoản quản trị viên.

Các chức năng quản trị được giới hạn theo quyền của tài khoản Admin nhằm ngăn người dùng thông thường truy cập vào những API quản lý của hệ thống.

## Luồng hoạt động của hệ thống

Luồng hoạt động tổng quát của hệ thống được triển khai như sau:

1. Người dùng hoặc quản trị viên truy cập giao diện tương ứng thông qua **Amazon CloudFront**.

2. CloudFront phân phối nội dung frontend được lưu trữ trong các bucket **Amazon S3** riêng biệt.

3. Người dùng đăng ký hoặc đăng nhập thông qua **Amazon Cognito**. Sau khi xác thực thành công, Cognito cung cấp token để frontend sử dụng khi gửi yêu cầu đến backend.

4. Frontend gửi yêu cầu nghiệp vụ kèm token đến **Amazon API Gateway REST API**.

5. API Gateway kiểm tra thông tin xác thực và chuyển tiếp yêu cầu hợp lệ đến các hàm **AWS Lambda**.

6. Lambda xử lý các nghiệp vụ của người dùng hoặc quản trị viên, sau đó đọc hoặc ghi dữ liệu trong **Amazon DynamoDB**.

7. Khi người dùng gửi yêu cầu đặt giá hợp lệ, yêu cầu được chuyển đến **Amazon SQS FIFO** để bảo đảm các lượt đặt giá của cùng một phiên được xử lý tuần tự.

8. Lambda xử lý đặt giá nhận thông điệp từ SQS FIFO, kiểm tra giá đặt và cập nhật thông tin mới vào DynamoDB.

9. Kết quả đặt giá được gửi đến **API Gateway WebSocket API** để cập nhật giá mới cho những người dùng đang theo dõi hoặc tham gia phiên đấu giá.

10. **AWS IAM** cung cấp Role và Policy để kiểm soát quyền truy cập giữa API Gateway, Lambda, SQS, DynamoDB, S3 và các dịch vụ AWS khác.

## Các dịch vụ được triển khai

Bảng dưới đây trình bày các dịch vụ AWS chính được sử dụng trong hệ thống.

| Dịch vụ AWS | Vai trò trong hệ thống |
| --- | --- |
| **AWS IAM** | Quản lý Role và Policy, đồng thời kiểm soát quyền truy cập giữa các dịch vụ AWS. |
| **Amazon Cognito** | Xác thực tài khoản và phân biệt quyền truy cập của User và Admin. |
| **Amazon S3** | Lưu trữ User Frontend, Admin Frontend và các tài nguyên tĩnh của hệ thống. |
| **Amazon CloudFront** | Phân phối hai giao diện frontend đến người dùng thông qua mạng phân phối nội dung CDN. |
| **AWS Lambda** | Xử lý các nghiệp vụ của người dùng, quản trị viên, phiên đấu giá và yêu cầu đặt giá. |
| **Amazon API Gateway REST API** | Cung cấp các API nghiệp vụ để frontend giao tiếp với các hàm Lambda. |
| **Amazon API Gateway WebSocket API** | Duy trì kết nối thời gian thực và gửi thông tin cập nhật của phiên đấu giá đến người dùng. |
| **Amazon DynamoDB** | Lưu trữ tài khoản, danh mục, phiên đấu giá, vật phẩm, lượt đặt giá, trạng thái phiên và kết nối WebSocket. |
| **Amazon SQS FIFO** | Tiếp nhận và xử lý tuần tự các yêu cầu đặt giá theo từng phiên đấu giá. |

## Kiểm tra tài nguyên trên AWS

Sau khi Terraform hoàn tất quá trình triển khai, nhóm đăng nhập vào **AWS Management Console** để kiểm tra các tài nguyên đã được tạo.

Các nội dung cần kiểm tra gồm:

* **Amazon Cognito User Pool** và **App Client**.
* **IAM Role** và **IAM Policy**.
* Các **Amazon S3 Bucket** dùng để lưu trữ User Frontend và Admin Frontend.
* Các **Amazon CloudFront Distribution** dùng để phân phối hai giao diện frontend.
* Danh sách **AWS Lambda Function**.
* **Amazon API Gateway REST API** và các Route.
* **Amazon API Gateway WebSocket API** và các Route.
* Các bảng **Amazon DynamoDB**.
* **Amazon SQS FIFO Queue**.
* AWS Region của từng tài nguyên.
* Tên tài nguyên có tuân thủ quy ước đặt tên của dự án hay không.
* Trạng thái hoạt động của các tài nguyên sau khi triển khai.

### Các bước kiểm tra tài nguyên bằng AWS Resource Explorer

Để kiểm tra tổng thể các tài nguyên đã được Terraform triển khai, nhóm thực hiện các bước sau:

**Bước 1:** Đăng nhập vào **AWS Management Console** bằng tài khoản được sử dụng để triển khai hệ thống.

**Bước 2:** Tại thanh tìm kiếm của AWS Management Console, nhập:

```text
AWS Resource Explorer
```

Sau đó chọn dịch vụ **AWS Resource Explorer** từ danh sách kết quả.

**Bước 3:** Trong giao diện AWS Resource Explorer, chọn **Resources** tại nhóm chức năng **Explore resources**.

**Bước 4:** Tại trường **View**, chọn Region được sử dụng để triển khai hệ thống:

```text
ap-southeast-1
```

Region này tương ứng với khu vực **Asia Pacific (Singapore)**.

**Bước 5:** Trong ô **Query keywords, filters and operators**, nhập tiền tố tên tài nguyên của dự án:

```text
la-
```

Tiền tố `la-` được sử dụng trong Terraform để đặt tên cho các tài nguyên thuộc hệ thống **Live Auction**.

**Bước 6:** Nhấn **Enter** và chờ AWS Resource Explorer trả về danh sách tài nguyên phù hợp.

**Bước 7:** Kiểm tra các thông tin được hiển thị trong kết quả, bao gồm:

* **Identifier:** Tên hoặc mã định danh của tài nguyên.
* **Service:** Dịch vụ AWS tương ứng.
* **Resource type:** Loại tài nguyên.
* **Region:** Khu vực triển khai tài nguyên.
* **Tags:** Các thẻ được gắn cho tài nguyên.

Kết quả tìm kiếm hiển thị **63 tài nguyên** có tiền tố `la-`. Danh sách này bao gồm các tài nguyên chính của hệ thống và một số tài nguyên hỗ trợ như Lambda Function, DynamoDB Table, SQS Queue, CloudWatch Alarm, CloudTrail và CodeBuild Project.

### Các tài nguyên được triển khai trên AWS

<!--
HƯỚNG DẪN CHỤP HÌNH:
1. Đăng nhập AWS Management Console.
2. Mở trang Resource Groups & Tag Editor hoặc từng trang dịch vụ AWS.
3. Chụp màn hình thể hiện các tài nguyên chính của dự án.
4. Không để lộ Account ID, Access Key, email hoặc thông tin nhạy cảm.
5. Lưu ảnh tại:
   static/images/5-Workshop/5.4-AWS-services/aws-deployed-resources.png
-->

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/aws-deployed-resources.png"
    title="Hình 5.4.2: Các tài nguyên của hệ thống Live Auction trên AWS Management Console"
    width="100%"
>}}

## Kết quả

Sau khi hoàn tất quá trình triển khai:

* Các dịch vụ AWS đã được tạo và quản lý bằng Terraform.
* Hai giao diện User Frontend và Admin Frontend được lưu trữ trên Amazon S3 và phân phối thông qua Amazon CloudFront.
* Người dùng và quản trị viên được xác thực bằng Amazon Cognito.
* Các yêu cầu nghiệp vụ được tiếp nhận thông qua Amazon API Gateway REST API.
* Logic nghiệp vụ của người dùng, quản trị viên và phiên đấu giá được xử lý bằng AWS Lambda.
* Dữ liệu hệ thống được lưu trữ trong Amazon DynamoDB.
* Yêu cầu đặt giá được xử lý tuần tự thông qua Amazon SQS FIFO.
* Trạng thái đấu giá được cập nhật theo thời gian thực bằng API Gateway WebSocket API.
* Quyền truy cập giữa các dịch vụ được kiểm soát bằng AWS IAM.
* Các thành phần đã sẵn sàng để được kiểm tra chi tiết trong những mục tiếp theo.

## Phân chia nội dung triển khai

Quá trình cấu hình và triển khai từng dịch vụ được trình bày trong các mục sau:

* **5.4.1. AWS IAM và Amazon Cognito**
* **5.4.2. Amazon S3**
* **5.4.3. Amazon CloudFront**
* **5.4.4. AWS Lambda**
* **5.4.5. Amazon API Gateway REST API**
* **5.4.6. Amazon API Gateway WebSocket API**
* **5.4.7. Amazon DynamoDB**
* **5.4.8. Amazon SQS FIFO**

Trong mỗi mục, Workshop sẽ trình bày vai trò của dịch vụ, tài nguyên được triển khai bằng Terraform, các thông số cấu hình quan trọng và kết quả kiểm tra sau khi triển khai.

