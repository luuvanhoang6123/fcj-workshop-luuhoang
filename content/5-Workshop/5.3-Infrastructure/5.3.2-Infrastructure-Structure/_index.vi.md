---
title: "Cấu trúc thư mục Infrastructure"
date: 2026-07-13
weight: 2
chapter: false
pre: "<b>5.3.2. </b>"
---

## Giới thiệu

Toàn bộ hạ tầng của hệ thống Live Auction được quản lý trong thư mục **infra/** bằng Terraform theo mô hình **Infrastructure as Code (IaC)**. Thay vì khai báo toàn bộ tài nguyên trong một tệp duy nhất, nhóm chia hạ tầng thành nhiều module độc lập tương ứng với từng lớp chức năng của hệ thống. Cách tổ chức này giúp quá trình phát triển, kiểm thử và bảo trì trở nên dễ dàng hơn, đồng thời hạn chế ảnh hưởng giữa các thành phần khi có sự thay đổi.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/infrastructure-structure.png" alt="Infrastructure Directory Structure" width="45%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.2.</b> Cấu trúc thư mục Infrastructure của dự án.
    </figcaption>
</figure>

---

## Thư mục 00-bootstrap

Thư mục **00-bootstrap/** được sử dụng trong giai đoạn khởi tạo ban đầu của dự án. Đây là nơi chứa các tập lệnh phục vụ việc chuẩn bị môi trường Terraform trước khi triển khai hạ tầng chính thức.

Trong thư mục này có tệp **bootstrap-remote-state.ps1**, được sử dụng để tạo các tài nguyên cần thiết cho Terraform Backend, giúp lưu trữ trạng thái (Terraform State) trên AWS thay vì lưu cục bộ. Ngoài ra còn có thư mục **tests/** phục vụ việc kiểm thử quá trình khởi tạo.

Việc tách riêng giai đoạn bootstrap giúp các bước triển khai tiếp theo không cần cấu hình lại Terraform Backend, đồng thời đảm bảo nhiều thành viên trong nhóm có thể sử dụng chung một trạng thái hạ tầng.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/bootstrap-structure.png" alt="Bootstrap Structure" width="55%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.3.</b> Cấu trúc thư mục <code>00-bootstrap</code>.
    </figcaption>
</figure>

---

## Thư mục 01-foundation

Sau khi hoàn thành bước bootstrap, Terraform tiếp tục sử dụng thư mục **01-foundation/** để triển khai các thành phần nền tảng của hệ thống.

Thư mục này chứa các cấu hình cơ bản như **backend.tf**, dùng để khai báo Terraform Backend và kết nối tới nơi lưu trữ Terraform State. Đây là nền tảng để các module phía sau có thể kế thừa và triển khai hạ tầng một cách thống nhất.

Việc tách riêng phần Foundation giúp các cấu hình dùng chung chỉ cần định nghĩa một lần và có thể tái sử dụng cho toàn bộ hệ thống.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/foundation-structure.png" alt="Foundation Structure" width="45%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.4.</b> Cấu trúc thư mục <code>01-foundation</code>.
    </figcaption>
</figure>

---

## Thư mục 03-identity

Thư mục **03-identity/** quản lý toàn bộ các tài nguyên liên quan đến định danh và xác thực của hệ thống.

Module này triển khai các dịch vụ như **AWS Identity and Access Management (IAM)** và **Amazon Cognito**, phục vụ việc quản lý người dùng, phân quyền truy cập và xác thực tài khoản khi sử dụng hệ thống Live Auction.

Ngoài các tệp cấu hình Terraform như **main.tf**, **variables.tf**, **outputs.tf**, **providers.tf**, **backend.tf** và **versions.tf**, thư mục còn chứa **terraform.lock.hcl** để khóa phiên bản Provider, thư mục **.terraform/** và tệp **tfplan** được sinh ra trong quá trình lập kế hoạch triển khai.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/identity-structure.png" alt="Identity Structure" width="35%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.5.</b> Cấu trúc module <code>03-identity</code>.
    </figcaption>
</figure>

---

## Thư mục 04-data

Thư mục **04-data/** chịu trách nhiệm triển khai lớp lưu trữ dữ liệu của hệ thống. Đây là nơi Terraform định nghĩa và quản lý các tài nguyên liên quan đến **Amazon DynamoDB**, nơi lưu trữ dữ liệu chính của hệ thống Live Auction.

Các bảng DynamoDB được tạo trong giai đoạn này phục vụ nhiều mục đích khác nhau như quản lý danh mục đấu giá, trạng thái phiên đấu giá, lịch sử đấu giá, kết nối WebSocket và các dữ liệu nghiệp vụ khác. Việc tách riêng lớp dữ liệu thành một module độc lập giúp việc mở rộng hoặc thay đổi cấu trúc cơ sở dữ liệu trở nên thuận tiện hơn mà không ảnh hưởng đến các thành phần còn lại của hệ thống.

Bên cạnh tệp **main.tf** dùng để khai báo tài nguyên DynamoDB, module còn bao gồm các tệp **variables.tf**, **outputs.tf**, **providers.tf**, **backend.tf** và **versions.tf** nhằm chuẩn hóa quá trình triển khai và quản lý hạ tầng.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/data-structure.png" alt="Data Structure" width="35%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.6.</b> Cấu trúc module <code>04-data</code>.
    </figcaption>
</figure>

---

## Thư mục 05-messaging

Thư mục **05-messaging/** quản lý các dịch vụ truyền thông điệp của hệ thống. Trong đồ án Live Auction, module này được sử dụng để triển khai **Amazon SQS FIFO**, đóng vai trò tiếp nhận và điều phối các yêu cầu đặt giá từ người dùng.

Trong quá trình đấu giá thời gian thực, nhiều người dùng có thể gửi yêu cầu đặt giá gần như đồng thời. Việc sử dụng hàng đợi FIFO giúp đảm bảo các yêu cầu được xử lý theo đúng thứ tự phát sinh, hạn chế xung đột dữ liệu và duy trì tính nhất quán của phiên đấu giá.

Ngoài các tệp cấu hình Terraform chuẩn như **main.tf**, **variables.tf**, **outputs.tf**, **providers.tf**, **backend.tf** và **versions.tf**, module còn lưu trữ kết quả của lệnh **terraform plan** để hỗ trợ quá trình kiểm tra trước khi triển khai.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/messaging-structure.png" alt="Messaging Structure" width="35%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.7.</b> Cấu trúc module <code>05-messaging</code>.
    </figcaption>
</figure>

---

## Thư mục 06-compute

Thư mục **06-compute/** chịu trách nhiệm triển khai lớp xử lý nghiệp vụ của hệ thống. Đây là module quản lý các tài nguyên tính toán trên AWS, chủ yếu là các **AWS Lambda Functions** thực hiện xử lý logic của hệ thống Live Auction.

Các hàm Lambda đảm nhiệm nhiều chức năng như xử lý yêu cầu từ REST API, xử lý sự kiện WebSocket, xác thực người dùng, quản lý phiên đấu giá, điều phối các tác vụ nền và xử lý dữ liệu nhận từ Amazon SQS.

Bên trong module còn có thư mục **stage3-control-plane/**, được sử dụng để quản lý các cấu hình triển khai thuộc giai đoạn Control Plane của hệ thống. Việc tách riêng các cấu hình theo từng giai đoạn giúp quá trình triển khai trở nên rõ ràng hơn và thuận tiện cho việc kiểm thử cũng như bảo trì sau này.

Tương tự các module khác, thư mục này cũng bao gồm các tệp **main.tf**, **variables.tf**, **outputs.tf**, **providers.tf**, **backend.tf**, **versions.tf** và **terraform.lock.hcl** nhằm đảm bảo quá trình triển khai được thực hiện nhất quán.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/compute-structure.png" alt="Compute Structure" width="40%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.8.</b> Cấu trúc module <code>06-compute</code>.
    </figcaption>
</figure>

---

## Thư mục 07-api

Thư mục **07-api/** chịu trách nhiệm triển khai lớp giao tiếp giữa hệ thống và người dùng thông qua **Amazon API Gateway**. Đây là nơi Terraform cấu hình các REST API phục vụ các chức năng như đăng nhập, quản lý người dùng, quản lý phiên đấu giá và các nghiệp vụ khác của hệ thống.

Bên cạnh REST API, module còn triển khai **API Gateway WebSocket**, cho phép thiết lập kết nối hai chiều giữa máy khách và máy chủ. Kết nối WebSocket được sử dụng để truyền dữ liệu theo thời gian thực trong quá trình đấu giá, giúp người dùng nhận được thông tin cập nhật ngay khi có sự thay đổi về mức giá hoặc trạng thái phiên đấu giá.

Tương tự các module khác, thư mục này bao gồm các tệp **main.tf**, **variables.tf**, **outputs.tf**, **providers.tf**, **backend.tf**, **versions.tf**, cùng các tệp hỗ trợ quá trình lập kế hoạch và triển khai bằng Terraform.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/api-structure.png" alt="API Structure" width="30%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.9.</b> Cấu trúc module <code>07-api</code>.
    </figcaption>
</figure>

---

## Thư mục 09-edge

Thư mục **09-edge/** quản lý các dịch vụ biên (Edge Services) của hệ thống. Module này triển khai **Amazon S3** dùng để lưu trữ giao diện người dùng, giao diện quản trị và các tệp tĩnh của ứng dụng, đồng thời cấu hình **Amazon CloudFront** để phân phối nội dung đến người dùng thông qua mạng CDN.

Việc sử dụng CloudFront giúp giảm độ trễ truy cập, tăng tốc độ tải trang và cải thiện trải nghiệm người dùng khi truy cập hệ thống từ nhiều vị trí địa lý khác nhau. Ngoài ra, module cũng chịu trách nhiệm cấu hình các tài nguyên liên quan đến việc phân phối nội dung trên AWS.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/edge-structure.png" alt="Edge Structure" width="30%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.10.</b> Cấu trúc module <code>09-edge</code>.
    </figcaption>
</figure>
---

## Thư mục tests

Bên cạnh các module triển khai hạ tầng, dự án còn xây dựng thư mục **tests/** nhằm phục vụ việc kiểm thử sau khi triển khai.

Thư mục này bao gồm nhiều tập lệnh PowerShell được tổ chức theo từng giai đoạn của hệ thống như **stage1**, **stage2**, **stage3** và **stage4**. Các tập lệnh được sử dụng để kiểm tra hoạt động của từng nhóm tài nguyên như Identity, Compute, API, Data, Messaging cũng như kiểm thử tích hợp (Integration Test) và kiểm thử toàn bộ hệ thống (End-to-End Test).

Việc xây dựng các tập lệnh kiểm thử riêng giúp nhóm dễ dàng xác minh kết quả sau mỗi lần triển khai, đồng thời hỗ trợ phát hiện sớm các lỗi phát sinh trong quá trình phát triển.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/tests-structure.png" alt="Tests Structure" width="30%">
    <figcaption style="text-align: center;">
        <b>Hình 5.3.11.</b> Cấu trúc thư mục <code>tests</code>.
    </figcaption>
</figure>

---

## Các tệp cấu hình Terraform

Hầu hết các module trong thư mục **Infrastructure** đều được tổ chức theo cùng một cấu trúc nhằm đảm bảo tính nhất quán trong quá trình phát triển. Mỗi tệp cấu hình đảm nhận một vai trò riêng trong quá trình triển khai hạ tầng.

| Tệp cấu hình           | Chức năng                                                                                      |
| ---------------------- | ---------------------------------------------------------------------------------------------- |
| **main.tf**            | Khai báo các tài nguyên AWS được triển khai trong module.                                      |
| **variables.tf**       | Định nghĩa các biến đầu vào sử dụng trong cấu hình Terraform.                                  |
| **outputs.tf**         | Khai báo các giá trị đầu ra để sử dụng cho các module khác.                                    |
| **providers.tf**       | Cấu hình Terraform Provider và thông tin kết nối tới AWS.                                      |
| **backend.tf**         | Cấu hình Terraform Backend để lưu trữ Terraform State từ xa.                                   |
| **versions.tf**        | Quy định phiên bản Terraform và các Provider được sử dụng.                                     |
| **terraform.lock.hcl** | Khóa phiên bản Provider nhằm đảm bảo tính nhất quán giữa các lần triển khai.                   |
| **tfplan**             | Kết quả sinh ra từ lệnh `terraform plan`, dùng để xem trước các thay đổi trước khi triển khai. |

Việc chuẩn hóa cấu trúc của các module giúp các thành viên trong nhóm dễ dàng theo dõi, bảo trì và mở rộng hạ tầng trong quá trình phát triển hệ thống.

---

## Kết quả

Sau khi hoàn thiện cấu trúc thư mục Infrastructure, toàn bộ mã nguồn Terraform của hệ thống đã được tổ chức theo từng lớp chức năng riêng biệt. Cách tổ chức này giúp quá trình triển khai hạ tầng trở nên rõ ràng, thuận tiện cho việc phát triển, kiểm thử và bảo trì, đồng thời tạo nền tảng để thực hiện các bước khởi tạo, lập kế hoạch và triển khai hạ tầng AWS trong các mục tiếp theo.