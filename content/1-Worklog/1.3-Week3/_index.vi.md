---
title: "Worklog Tuần 3"
date: 2026-07-06
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Thời gian

**06/07/2026 – 10/07/2026**

### Mục tiêu cá nhân

- Chuyển từ đọc lý thuyết sang thao tác thực tế trên AWS.
- Cài đặt AWS CLI và hiểu cách xác thực tài khoản.
- Hình thành thói quen kiểm soát chi phí ngay từ đầu.

### Công việc đã thực hiện

| Thứ | Ngày | Nội dung | Tài liệu |
| --- | --- | --- | --- |
| Thứ Hai | 06/07/2026 | Khám phá Console: tìm kiếm dịch vụ, xem dashboard Billing, bật MFA cho tài khoản cá nhân. | [Console Getting Started](https://docs.aws.amazon.com/awsconsolehelpdocs/latest/gsg/getting-started.html) |
| Thứ Ba | 07/07/2026 | Tìm hiểu IAM User, Group, Policy; tạo user phụ chỉ có quyền đọc để thử nghiệm. | [IAM Introduction](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) |
| Thứ Tư | 08/07/2026 | Cài AWS CLI trên Windows; chạy `aws configure`, `aws sts get-caller-identity`, liệt kê Region. | [AWS CLI Install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| Thứ Năm | 09/07/2026 | Tạo EC2 t2.micro thử nghiệm; cấu hình Security Group, Key Pair; terminate instance sau khi xong. | [EC2 Get Started](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html) |
| Thứ Sáu | 10/07/2026 | Thiết lập AWS Budget cảnh báo 5 USD; rà soát Cost Explorer, ghi checklist tắt/xóa tài nguyên. | [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) |

### Kết quả và ghi chú

- Em tự cấu hình được CLI và xác minh danh tính tài khoản bằng dòng lệnh.
- Hiểu các thành phần tối thiểu khi khởi tạo EC2: AMI, instance type, key pair, security group, EBS.
- Đã thiết lập ngân sách cảnh báo; không để tài nguyên thử nghiệm chạy qua đêm.
- **Bài học:** Lần đầu quên terminate instance khiến em phải theo dõi Billing sát hơn — từ đó lập thói quen cleanup cuối ngày.
