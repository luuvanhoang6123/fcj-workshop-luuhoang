---
title: "Worklog Week 3"
date: 2026-07-06
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Duration

**July 6, 2026 – July 10, 2026**

### Personal objectives

- Move from theory to hands-on AWS operations.
- Install AWS CLI and understand account authentication.
- Build cost-control habits from the start.

### Activities completed

| Day | Date | Work | Reference |
| --- | --- | --- | --- |
| Monday | 06/07/2026 | Explored Console: service search, Billing dashboard, enabled MFA on my account. | [Console Getting Started](https://docs.aws.amazon.com/awsconsolehelpdocs/latest/gsg/getting-started.html) |
| Tuesday | 07/07/2026 | Studied IAM users, groups, policies; created a read-only test user. | [IAM Introduction](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) |
| Wednesday | 08/07/2026 | Installed AWS CLI on Windows; ran `aws configure`, `aws sts get-caller-identity`, listed regions. | [AWS CLI Install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| Thursday | 09/07/2026 | Launched a t2.micro EC2 instance; configured Security Group and Key Pair; terminated when done. | [EC2 Get Started](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html) |
| Friday | 10/07/2026 | Set a $5 AWS Budget alert; reviewed Cost Explorer; wrote a cleanup checklist. | [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) |

### Results and notes

- I configured CLI and verified account identity from the command line.
- I understood minimum EC2 components: AMI, instance type, key pair, security group, EBS.
- I set budget alerts and avoided leaving test resources running overnight.
- **Lesson:** Forgetting to terminate an instance once made me watch Billing more closely — I now cleanup at end of day.
