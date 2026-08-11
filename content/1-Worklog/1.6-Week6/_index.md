---
title: "Worklog Week 6"
date: 2026-07-27
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Duration

**July 27, 2026 – July 31, 2026**

### Personal objectives

- Deploy AWS infrastructure with Terraform (my primary responsibility).
- Provision serverless services and capture endpoint outputs for integration.
- Write Workshop section 5.3 (Infrastructure) with real commands and screenshots.

### Activities completed

| Day | Date | Work | Reference |
| --- | --- | --- | --- |
| Monday | 27/07/2026 | Installed Terraform 1.x; verified `terraform -version`; configured team AWS profile. | [Install Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) |
| Tuesday | 28/07/2026 | Created `Infrastructure/modules/` layout; declared provider, variables, locals; split IAM and Cognito modules. | [Terraform Language](https://developer.hashicorp.com/terraform/language) |
| Wednesday | 29/07/2026 | Ran `terraform init`, `validate`, `plan`; fixed duplicate resource names and Lambda–IAM dependency gaps. | [terraform init](https://developer.hashicorp.com/terraform/cli/commands/init) |
| Thursday | 30/07/2026 | `terraform apply` deployed S3, CloudFront, DynamoDB, SQS FIFO, API Gateway, Lambda; verified on Console. | [terraform apply](https://developer.hashicorp.com/terraform/cli/commands/apply) |
| Friday | 31/07/2026 | Set Lambda env vars; deployed frontend to S3; drafted Workshop 5.3.1–5.3.5 from deployment logs. | Source & deploy logs |

### Results and notes

- Core infrastructure provisioned via Terraform; outputs included REST URL, WebSocket URL, CloudFront domain.
- I drafted Infrastructure Workshop docs (init → plan → apply).
- Fixed missing `sqs:SendMessage` permission on bid producer Lambda IAM policy.
- **Challenge:** CloudFront cache invalidation after frontend upload — added this step to deploy guide.
- **Team coordination:** Teammate integrated Cognito UI; I provided endpoints and env var mapping table.
