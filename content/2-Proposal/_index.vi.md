---
title: "Bản đề xuất"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# LIVE AUCTION PLATFORM ON AWS

## Nền tảng đấu giá trực tuyến triển khai trên điện toán đám mây AWS

### 1. Tóm tắt điều hành

**Live Auction** là nền tảng đấu giá trực tuyến được đề xuất nhằm xây dựng một hệ thống hỗ trợ người dùng đăng ký tài khoản, quản lý sản phẩm, tạo và tham gia các phiên đấu giá cũng như cập nhật trạng thái đấu giá theo thời gian thực. Đồ án hướng đến việc nghiên cứu và ứng dụng các dịch vụ của **Amazon Web Services (AWS)** để xây dựng một hệ thống có khả năng mở rộng, đảm bảo tính sẵn sàng và đáp ứng nhu cầu triển khai trên nền tảng điện toán đám mây.

Hệ thống được phát triển với **React/Vite** cho frontend, **FastAPI (Python)** cho backend và **MySQL** trong quá trình phát triển cục bộ. Trong giai đoạn đề xuất, nhóm định hướng triển khai frontend trên **Amazon S3**, backend trên các dịch vụ điện toán của AWS, cơ sở dữ liệu trên **Amazon RDS for MySQL**, đồng thời nghiên cứu ứng dụng các dịch vụ như **Amazon API Gateway**, **AWS Lambda**, **Amazon Cognito**, **Amazon DynamoDB**, **Amazon SQS** và **Amazon CloudFront** nhằm đáp ứng yêu cầu mở rộng của hệ thống.

Bản đề xuất này trình bày kiến trúc, các dịch vụ AWS dự kiến sử dụng và định hướng triển khai của nhóm trong quá trình phát triển đồ án. Trong quá trình thực hiện, một số thành phần của kiến trúc có thể được điều chỉnh để phù hợp với phạm vi triển khai và điều kiện thực tế.

---

### 2. Tuyên bố vấn đề

#### Vấn đề hiện tại

Các hệ thống đấu giá trực tuyến cần xử lý đồng thời nhiều yêu cầu từ người dùng trong cùng một thời điểm. Khi số lượng người tham gia tăng lên, hệ thống có thể gặp phải nhiều khó khăn nếu không được thiết kế theo kiến trúc phù hợp.

Một số vấn đề thường gặp gồm:

- Dữ liệu giá đấu không được cập nhật kịp thời.
- Nhiều yêu cầu đặt giá được xử lý sai thứ tự.
- Người dùng không nhận được trạng thái mới của phiên đấu giá theo thời gian thực.
- Hiệu năng giảm khi số lượng người truy cập tăng cao.
- Hệ thống khó mở rộng khi bổ sung thêm chức năng.
- Khó quản lý hạ tầng và theo dõi hoạt động của hệ thống.
- Dữ liệu và thông tin người dùng cần được bảo vệ tốt hơn trong môi trường Internet.

Ngoài ra, việc triển khai toàn bộ hệ thống trên một máy chủ duy nhất sẽ tạo ra điểm lỗi đơn (Single Point of Failure), gây khó khăn cho việc mở rộng và bảo trì trong tương lai.

#### Giải pháp đề xuất

Để giải quyết các vấn đề trên, nhóm đề xuất triển khai hệ thống **Live Auction** trên nền tảng **Amazon Web Services (AWS)** theo từng giai đoạn.

Ở giai đoạn đầu, nhóm tập trung xây dựng các chức năng cốt lõi của hệ thống, bao gồm giao diện người dùng, backend xử lý nghiệp vụ, cơ sở dữ liệu và lưu trữ hình ảnh sản phẩm. Các thành phần này được đề xuất triển khai bằng những dịch vụ AWS phù hợp nhằm giúp hệ thống dễ dàng mở rộng và quản lý.

Trong các giai đoạn tiếp theo, kiến trúc sẽ được mở rộng bằng cách tích hợp thêm các dịch vụ AWS phục vụ xác thực người dùng, xử lý API, truyền dữ liệu thời gian thực, hàng đợi thông điệp, lưu trữ dữ liệu hiệu năng cao và các dịch vụ giám sát nhằm nâng cao khả năng chịu tải, tính sẵn sàng và bảo mật của hệ thống.

#### Lợi ích của giải pháp

Giải pháp được đề xuất mang lại các lợi ích sau:

- Cho phép người dùng truy cập hệ thống đấu giá thông qua trình duyệt web.
- Hỗ trợ cập nhật trạng thái đấu giá theo thời gian thực.
- Phân tách các thành phần của hệ thống để thuận tiện cho việc mở rộng.
- Tận dụng các dịch vụ được quản lý sẵn của AWS nhằm giảm chi phí quản trị hạ tầng.
- Nâng cao khả năng bảo mật, giám sát và quản lý tài nguyên.
- Làm nền tảng để nghiên cứu và triển khai kiến trúc serverless trong các giai đoạn tiếp theo.

---

### 3. Kiến trúc giải pháp

> **Lưu ý:** Chương này trình bày **kiến trúc và phương án triển khai được đề xuất** trong giai đoạn thiết kế của đồ án. Trong quá trình phát triển, nhóm có thể điều chỉnh một số thành phần để phù hợp với điều kiện triển khai thực tế. Kiến trúc triển khai cuối cùng sẽ được trình bày chi tiết trong **Chương 5 - Workshop**.

#### 3.1. Kiến trúc triển khai ban đầu

Kiến trúc triển khai ban đầu được xây dựng nhằm đáp ứng các chức năng cốt lõi của hệ thống với mức độ phức tạp vừa phải, đồng thời tạo tiền đề cho việc mở rộng trong các giai đoạn tiếp theo.

Các thành phần chính của kiến trúc bao gồm:

1. Người dùng truy cập hệ thống thông qua trình duyệt web.
2. Frontend được phát triển bằng React/Vite và triển khai trên Amazon S3.
3. Frontend gửi yêu cầu đến các API của backend.
4. Backend được phát triển bằng FastAPI và được đề xuất triển khai trên hạ tầng AWS.
5. Dữ liệu được lưu trữ trên hệ quản trị cơ sở dữ liệu phù hợp với nhu cầu của hệ thống.
6. Hình ảnh sản phẩm được lưu trữ trên Amazon S3.
7. AWS Lambda được nghiên cứu sử dụng cho các tác vụ nền hoặc xử lý theo sự kiện.
8. Amazon CloudWatch hỗ trợ giám sát hoạt động và thu thập nhật ký hệ thống.

Kiến trúc này tập trung vào việc xây dựng các chức năng chính của hệ thống, đồng thời tạo nền tảng để mở rộng sang mô hình serverless và tích hợp thêm các dịch vụ AWS trong các giai đoạn tiếp theo.

#### 3.2. Kiến trúc AWS mở rộng được đề xuất

Sơ đồ dưới đây mô tả kiến trúc AWS mở rộng được nhóm đề xuất cho hệ thống Live Auction. Kiến trúc tập trung vào khả năng mở rộng, xử lý đấu giá theo thời gian thực, tăng cường bảo mật, giám sát và nâng cao tính sẵn sàng của hệ thống.

Nhấn vào sơ đồ để xem ở kích thước đầy đủ.

[![Sơ đồ kiến trúc AWS đề xuất cho hệ thống Live Auction](/images/2-Proposal/live-auction-proposed-architecture.svg)](/images/2-Proposal/live-auction-proposed-architecture.svg)

> **Lưu ý:** Sơ đồ trên thể hiện **kiến trúc mục tiêu** của hệ thống. Một số thành phần nâng cao sẽ được triển khai trong các giai đoạn tiếp theo tùy theo phạm vi và tiến độ của đồ án.

#### 3.3. Luồng hoạt động tổng quát

Luồng hoạt động tổng quát của kiến trúc được đề xuất như sau:

1. Người dùng truy cập hệ thống thông qua trình duyệt web.
2. Yêu cầu được định tuyến và phân phối đến giao diện người dùng.
3. Người dùng thực hiện đăng nhập và được xác thực trước khi sử dụng các chức năng yêu cầu quyền truy cập.
4. Frontend gửi các yêu cầu nghiệp vụ đến backend thông qua API.
5. Backend tiếp nhận và xử lý các yêu cầu của người dùng.
6. Các dữ liệu cần lưu trữ được ghi vào dịch vụ cơ sở dữ liệu phù hợp.
7. Hình ảnh và các tài nguyên tĩnh được lưu trữ trên Amazon S3.
8. Các yêu cầu đặt giá có thể được xử lý tuần tự thông qua hàng đợi thông điệp trong kiến trúc mở rộng.
9. Trạng thái đấu giá được cập nhật đến các người dùng đang kết nối theo thời gian thực.
10. Hệ thống được giám sát thông qua các dịch vụ theo dõi và ghi nhật ký của AWS.


### 4. Các dịch vụ AWS đề xuất sử dụng

#### Amazon Route 53

Amazon Route 53 được đề xuất sử dụng để quản lý tên miền và định tuyến người dùng đến hệ thống. Trong kiến trúc mở rộng, Route 53 có thể kết hợp với các cơ chế kiểm tra tình trạng dịch vụ (Health Check) và Failover Routing nhằm hỗ trợ chuyển hướng truy cập sang môi trường dự phòng khi xảy ra sự cố.

#### Amazon CloudFront

Amazon CloudFront được đề xuất làm mạng phân phối nội dung (Content Delivery Network - CDN) cho hệ thống. Dịch vụ này giúp phân phối các tệp HTML, CSS, JavaScript và hình ảnh từ Amazon S3 đến người dùng thông qua các Edge Location, từ đó giảm độ trễ và cải thiện tốc độ truy cập.

Ngoài việc tăng hiệu năng, CloudFront còn hỗ trợ HTTPS, bộ nhớ đệm (Caching) và có thể tích hợp với AWS WAF nhằm tăng cường khả năng bảo mật cho hệ thống.

#### AWS WAF

AWS WAF được đề xuất để bảo vệ ứng dụng web trước các hình thức tấn công phổ biến như SQL Injection, Cross-site Scripting (XSS) và các yêu cầu truy cập bất thường.

Khi kết hợp với Amazon CloudFront hoặc Amazon API Gateway, AWS WAF giúp giới hạn tần suất truy cập, lọc các yêu cầu không hợp lệ và tăng cường mức độ an toàn cho hệ thống.

#### Amazon S3

Amazon S3 được đề xuất sử dụng cho nhiều mục đích khác nhau trong hệ thống, bao gồm:

- Lưu trữ frontend sau khi build.
- Lưu trữ giao diện quản trị.
- Lưu trữ hình ảnh sản phẩm.
- Lưu trữ các tài nguyên tĩnh và dữ liệu phục vụ kiểm tra khi cần thiết.

Việc sử dụng Amazon S3 giúp tách biệt dữ liệu tĩnh khỏi backend, giảm tải cho hệ thống xử lý nghiệp vụ và tạo điều kiện thuận lợi cho việc mở rộng dung lượng lưu trữ trong tương lai.

#### Amazon EC2

Trong phương án triển khai ban đầu, Amazon EC2 được đề xuất là môi trường vận hành backend FastAPI. Backend được đóng gói bằng Docker nhằm đảm bảo tính nhất quán giữa môi trường phát triển và môi trường triển khai.

Việc sử dụng EC2 giúp nhóm chủ động quản lý môi trường chạy, đồng thời thuận tiện trong quá trình nghiên cứu, kiểm thử và triển khai các chức năng cốt lõi của hệ thống.

#### Amazon ECS và AWS Fargate

Trong giai đoạn mở rộng, nhóm định hướng chuyển backend từ mô hình triển khai trên máy chủ sang nền tảng quản lý container bằng Amazon ECS kết hợp AWS Fargate.

Giải pháp này giúp giảm công việc quản trị hạ tầng, hỗ trợ mở rộng số lượng container theo nhu cầu và tăng khả năng sẵn sàng của hệ thống.

#### Amazon ECR

Amazon Elastic Container Registry (Amazon ECR) được đề xuất sử dụng để lưu trữ Docker Image của backend.

Khi áp dụng quy trình CI/CD, các phiên bản Docker Image mới có thể được build, lưu trữ trên Amazon ECR và triển khai tự động đến các môi trường vận hành.

#### Elastic Load Balancing

Application Load Balancer (ALB) được đề xuất nhằm phân phối lưu lượng truy cập đến nhiều backend service hoặc container trong trường hợp hệ thống được mở rộng.

ALB còn hỗ trợ kiểm tra tình trạng dịch vụ (Health Check), SSL Termination và cân bằng tải giữa nhiều phiên bản của ứng dụng.

#### Amazon API Gateway

Amazon API Gateway được đề xuất là điểm truy cập thống nhất cho toàn bộ API của hệ thống.

Dịch vụ này hỗ trợ triển khai REST API và WebSocket API, đồng thời tích hợp với nhiều dịch vụ AWS khác như AWS Lambda và Amazon Cognito.

Trong kiến trúc mở rộng, REST API được sử dụng để xử lý các nghiệp vụ của hệ thống như quản lý tài khoản, sản phẩm và phiên đấu giá, trong khi WebSocket API hỗ trợ truyền dữ liệu đấu giá theo thời gian thực đến các người dùng đang kết nối.

#### AWS Lambda

AWS Lambda được đề xuất sử dụng để thực thi các nghiệp vụ của hệ thống theo mô hình serverless. Thay vì duy trì máy chủ hoạt động liên tục, các hàm Lambda chỉ được kích hoạt khi có yêu cầu từ người dùng hoặc từ các dịch vụ AWS khác.

Trong kiến trúc đề xuất, Lambda có thể đảm nhiệm các chức năng như:

- Xử lý các yêu cầu từ REST API.
- Tiếp nhận và xử lý dữ liệu từ WebSocket.
- Thực hiện các tác vụ nền theo sự kiện.
- Gửi thông báo hoặc đồng bộ dữ liệu giữa các thành phần của hệ thống.

Việc sử dụng Lambda giúp giảm chi phí vận hành, đồng thời tăng khả năng mở rộng theo lưu lượng truy cập thực tế.

---

#### Amazon RDS và Amazon Aurora

Trong phương án triển khai ban đầu, nhóm đề xuất sử dụng **Amazon RDS for MySQL** để lưu trữ dữ liệu quan hệ của hệ thống như thông tin người dùng, sản phẩm, phiên đấu giá và lịch sử giao dịch.

Ở giai đoạn mở rộng, **Amazon Aurora Serverless** được xem là một lựa chọn nhằm tăng hiệu năng truy vấn, khả năng mở rộng và tính sẵn sàng của cơ sở dữ liệu mà vẫn đảm bảo khả năng tương thích với MySQL.

---

#### Amazon DynamoDB

Amazon DynamoDB được đề xuất sử dụng cho các dữ liệu cần truy cập với độ trễ thấp hoặc phục vụ các chức năng theo thời gian thực.

Một số dữ liệu phù hợp để lưu trữ trên DynamoDB gồm:

- Trạng thái phiên đấu giá.
- Danh sách kết nối WebSocket.
- Lịch sử đặt giá.
- Các dữ liệu tạm thời phục vụ xử lý theo sự kiện.

Việc kết hợp cơ sở dữ liệu quan hệ và cơ sở dữ liệu NoSQL giúp tận dụng ưu điểm của từng mô hình lưu trữ.

---

#### Amazon SQS

Amazon Simple Queue Service (Amazon SQS) được đề xuất nhằm xử lý các yêu cầu đặt giá theo cơ chế hàng đợi.

Đối với hệ thống đấu giá, việc tiếp nhận nhiều yêu cầu đặt giá trong cùng một thời điểm có thể dẫn đến xung đột nếu không được xử lý đúng thứ tự. Amazon SQS giúp các yêu cầu được lưu vào hàng đợi trước khi chuyển đến thành phần xử lý, từ đó giảm nguy cơ mất dữ liệu và đảm bảo tính tuần tự trong quá trình xử lý.

---

#### Amazon EventBridge

Amazon EventBridge được đề xuất để xây dựng kiến trúc hướng sự kiện (Event-driven Architecture).

Thông qua EventBridge, các sự kiện phát sinh từ hệ thống như tạo phiên đấu giá, kết thúc phiên đấu giá hoặc thay đổi trạng thái có thể được chuyển đến các dịch vụ xử lý tương ứng mà không làm tăng mức độ phụ thuộc giữa các thành phần của hệ thống.

---

#### Amazon Kinesis

Amazon Kinesis được xem là giải pháp mở rộng cho các tình huống cần thu thập và xử lý luồng dữ liệu lớn theo thời gian thực.

Trong phạm vi đồ án, Kinesis chưa phải là thành phần bắt buộc nhưng có thể được nghiên cứu nhằm phục vụ việc phân tích dữ liệu hoặc thống kê hoạt động của hệ thống trong tương lai.

---

#### Amazon Cognito

Amazon Cognito được đề xuất để quản lý xác thực và phân quyền người dùng.

Dịch vụ hỗ trợ các chức năng như:

- Đăng ký tài khoản.
- Đăng nhập.
- Xác thực bằng JWT.
- Quản lý phiên làm việc.
- Khôi phục mật khẩu.
- Quản lý thông tin người dùng.

Việc sử dụng Cognito giúp giảm khối lượng xử lý xác thực của backend và tận dụng cơ chế bảo mật do AWS cung cấp.

---

#### Amazon CloudWatch

Amazon CloudWatch được đề xuất để theo dõi hoạt động của hệ thống, thu thập nhật ký và giám sát hiệu năng của các dịch vụ AWS.

Thông qua CloudWatch, nhóm có thể theo dõi trạng thái hoạt động của ứng dụng, phát hiện lỗi và hỗ trợ quá trình phân tích khi xảy ra sự cố.

---

#### AWS IAM

AWS Identity and Access Management (IAM) được sử dụng để quản lý người dùng, nhóm người dùng, vai trò (Role) và quyền truy cập đối với các tài nguyên AWS.

Việc phân quyền theo nguyên tắc **Least Privilege** giúp hạn chế các quyền không cần thiết, đồng thời tăng cường tính bảo mật cho toàn bộ hệ thống.

---

### 5. Thiết kế các thành phần của hệ thống

Hệ thống Live Auction được đề xuất xây dựng theo kiến trúc phân tầng, trong đó mỗi thành phần đảm nhận một vai trò riêng nhằm tăng tính độc lập, khả năng bảo trì và mở rộng.

Các thành phần chính bao gồm:

#### Frontend

Frontend được phát triển bằng **React** kết hợp với **Vite**, cung cấp giao diện cho người dùng và quản trị viên.

Frontend chịu trách nhiệm:

- Hiển thị dữ liệu.
- Gửi yêu cầu đến backend thông qua API.
- Tiếp nhận dữ liệu thời gian thực.
- Quản lý trạng thái giao diện.
- Điều hướng giữa các chức năng của hệ thống.

---

#### Backend

Backend được phát triển bằng **FastAPI (Python)** theo mô hình RESTful API.

Các chức năng chính gồm:

- Xử lý nghiệp vụ.
- Quản lý phiên đấu giá.
- Quản lý người dùng.
- Quản lý sản phẩm.
- Kiểm tra dữ liệu đầu vào.
- Xử lý xác thực và phân quyền.
- Kết nối với cơ sở dữ liệu và các dịch vụ AWS.

---

#### Cơ sở dữ liệu

Hệ thống sử dụng cơ sở dữ liệu để lưu trữ toàn bộ thông tin của ứng dụng.

Trong phương án đề xuất, dữ liệu quan hệ được lưu trên **Amazon RDS**, đồng thời các dữ liệu phục vụ xử lý thời gian thực có thể được lưu trên **Amazon DynamoDB** nhằm đáp ứng yêu cầu về hiệu năng và khả năng mở rộng.

---

#### Lưu trữ tài nguyên

Các hình ảnh sản phẩm, tệp tĩnh và giao diện sau khi xây dựng được đề xuất lưu trữ trên **Amazon S3**.

Việc tách dữ liệu tĩnh khỏi backend giúp giảm tải cho máy chủ xử lý nghiệp vụ và tăng hiệu quả phân phối nội dung khi kết hợp với Amazon CloudFront.

---

### 6. Kế hoạch triển khai

Nhằm đảm bảo việc phát triển hệ thống được thực hiện có lộ trình, nhóm đề xuất chia quá trình triển khai thành nhiều giai đoạn. Mỗi giai đoạn tập trung vào một nhóm chức năng và các dịch vụ AWS tương ứng, giúp giảm rủi ro trong quá trình phát triển và thuận tiện cho việc kiểm thử.

#### Giai đoạn 1: Xây dựng hệ thống cốt lõi

Trong giai đoạn đầu, nhóm tập trung phát triển các chức năng cơ bản của hệ thống, bao gồm:

- Xây dựng giao diện người dùng bằng React/Vite.
- Phát triển backend bằng FastAPI.
- Thiết kế cơ sở dữ liệu.
- Xây dựng các API phục vụ người dùng và quản trị viên.
- Hoàn thiện các chức năng đăng nhập, quản lý sản phẩm và quản lý phiên đấu giá.

Đồng thời, nhóm tiến hành chuẩn bị môi trường phát triển và nghiên cứu các dịch vụ AWS sẽ được sử dụng trong các giai đoạn tiếp theo.

---

#### Giai đoạn 2: Triển khai trên AWS

Sau khi hoàn thiện các chức năng cốt lõi, nhóm đề xuất triển khai hệ thống lên nền tảng Amazon Web Services.

Các công việc chính bao gồm:

- Triển khai frontend lên Amazon S3.
- Cấu hình Amazon CloudFront để phân phối nội dung.
- Thiết lập xác thực người dùng bằng Amazon Cognito.
- Triển khai các API trên hạ tầng AWS.
- Thiết lập cơ sở dữ liệu.
- Cấu hình lưu trữ hình ảnh và các tài nguyên tĩnh.
- Thiết lập các dịch vụ giám sát và ghi nhật ký.

---

#### Giai đoạn 3: Mở rộng hệ thống

Sau khi hệ thống hoạt động ổn định, nhóm định hướng mở rộng kiến trúc nhằm tăng khả năng mở rộng và đáp ứng lượng người dùng lớn hơn.

Một số nội dung được đề xuất nghiên cứu gồm:

- Triển khai kiến trúc serverless.
- Ứng dụng WebSocket cho đấu giá theo thời gian thực.
- Tích hợp hàng đợi thông điệp để xử lý các yêu cầu đặt giá.
- Bổ sung các dịch vụ bảo mật và giám sát.
- Xây dựng quy trình CI/CD cho việc triển khai hệ thống.

---

### 7. Đánh giá và kết luận

Kiến trúc được đề xuất hướng tới việc xây dựng một nền tảng đấu giá trực tuyến có khả năng mở rộng, dễ bảo trì và tận dụng hiệu quả các dịch vụ do Amazon Web Services cung cấp.

Việc phân tách hệ thống thành nhiều thành phần độc lập giúp thuận tiện trong quá trình phát triển, kiểm thử và mở rộng về sau. Đồng thời, việc nghiên cứu các dịch vụ AWS ngay từ giai đoạn thiết kế tạo tiền đề để nhóm từng bước hiện thực hóa hệ thống trên nền tảng điện toán đám mây.

Bản đề xuất này đóng vai trò là cơ sở cho quá trình triển khai của đồ án. Trong quá trình thực hiện, nhóm có thể điều chỉnh một số thành phần của kiến trúc để phù hợp với điều kiện triển khai thực tế, yêu cầu nghiệp vụ và phạm vi của đồ án.

Kiến trúc triển khai và các bước cấu hình thực tế sẽ được trình bày chi tiết trong **Chương 5 – Workshop**, bao gồm quá trình triển khai hạ tầng, cấu hình các dịch vụ AWS và kiểm thử hệ thống sau khi hoàn thành.
