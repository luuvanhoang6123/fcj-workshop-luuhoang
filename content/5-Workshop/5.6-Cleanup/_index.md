---
title: "Resource Cleanup"
date: 2026-08-09
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

## Overview

After completing the Workshop, AWS resources that are no longer required should be removed to prevent additional costs.

Because the Live Auction system still needs to remain available for demonstration, testing, and project evaluation, the team **has not deleted the resources at the time of writing this report**. This section describes the cleanup process that will be performed after the project has been accepted and the website no longer needs to remain available.

{{% notice warning %}}
Do not run `terraform destroy` while the system is still required for demonstration or evaluation. This operation may delete the website, APIs, Lambda functions, DynamoDB data, Cognito accounts, and other related resources, making the system unavailable. The team will perform the cleanup only after the project has been accepted and maintaining the system is no longer required.
{{% /notice %}}

## Notes Before Cleanup

Before deleting resources, confirm that:

- The project evaluation has been completed.
- The website is no longer required for demonstration.
- The source code has been stored on GitHub.
- The report and images have been backed up.
- No team member is using the AWS environment.
- All required data has been backed up.
- AWS CLI is using the correct account and Region.
- Terraform State still exists and is accessible.
- No `terraform apply` or CI/CD process is running.

Verify the current AWS account:

```powershell
aws sts get-caller-identity
```

Verify the configured Region:

```powershell
aws configure get region
```

The project Region is:

```text
ap-southeast-1
```

{{% notice warning %}}
Do not include Access Keys, Secret Access Keys, Session Tokens, or other AWS credentials in the report. Carefully verify the Account ID before running any command to avoid deleting resources from the wrong account.
{{% /notice %}}

## Data Backup

Before cleanup, the team must preserve all important data.

### Source Code and Report

Verify that all changes have been committed and pushed to GitHub:

```powershell
git status
git add .
git commit -m "Complete Live Auction workshop"
git push
```

If `git status` displays:

```text
nothing to commit, working tree clean
```

the current changes have already been committed.

### Amazon S3 Data

Download the files that must be retained from:

- User Frontend Bucket.
- Admin Frontend Bucket.
- Item Media Bucket.
- Auction item images.
- Required configuration files or artifacts.

AWS CLI can be used as follows:

```powershell
aws s3 sync "s3://<bucket-name>" ".\backup\<bucket-name>"
```

Replace `<bucket-name>` with the Bucket that needs to be backed up.

### Amazon DynamoDB Data

The important tables that should be backed up include:

```text
la_item_auction_state
la_bid_events
la_auction_catalog
la_category_catalog
la_admin_audit_events
```

The following backup mechanisms can be used:

- Point-in-Time Recovery.
- AWS Backup.
- Export to Amazon S3.
- On-demand backup.

Before continuing with cleanup, verify that the backup status is:

```text
Available
```

### Terraform Configuration

The following files should be retained:

```text
*.tf
*.tfvars
.terraform.lock.hcl
```

Files containing sensitive information must not be uploaded to GitHub. If `terraform.tfvars` contains secrets, it should be stored securely and included in `.gitignore`.

## Reviewing the Destruction Plan

Before deleting resources from a module, open its Terraform directory and run:

```powershell
terraform init
```

Then generate a destruction plan:

```powershell
terraform plan -destroy
```

This command only displays the destruction plan and does not delete any resources.

Verify:

- The correct AWS Account is selected.
- The correct Region is selected.
- The correct Terraform Workspace is active.
- The correct Backend State is being used.
- The resources scheduled for deletion belong to the project.
- No resource outside the project is included.
- No unexpected change is present.

{{% notice info %}}
Run `terraform plan -destroy` before every `terraform destroy`. Continue only after confirming that all resources in the plan belong to the Live Auction system.
{{% /notice %}}

## Destroying Resources with Terraform

Because the infrastructure is divided into multiple Terraform modules, resources should be destroyed in the **reverse order of deployment**. This helps prevent dependency errors between modules.

The recommended order is:

```text
13-cicd
12-backup-dr
11-security
10-observability
09-edge
07-api
06-compute/stage3-control-plane
06-compute
05-messaging
04-data
03-identity
01-foundation
00-bootstrap
```

In each module, run:

```powershell
terraform init
terraform plan -destroy
terraform destroy
```

Terraform requests confirmation:

```text
Enter a value:
```

Enter:

```text
yes
```

only after carefully reviewing the destruction plan.

Example:

```powershell
cd "D:\ThucTap\Live-Auction\infra\09-edge"
terraform init
terraform plan -destroy
terraform destroy
```

After one module has been successfully removed, continue with the next dependent module.

{{% notice warning %}}
Do not copy and run the destruction commands for all modules simultaneously. Process one module at a time and verify the result before moving to the next module.
{{% /notice %}}

## Resources Requiring Additional Preparation

Some resources may prevent `terraform destroy` from completing if they still contain data or remain in use.

### Amazon S3

An S3 Bucket may not be deleted if it still contains:

- Objects.
- Object Versions.
- Delete Markers.
- Incomplete Multipart Uploads.

Back up all required data and empty the Bucket before destroying its Terraform module.

### Amazon CloudFront

A CloudFront Distribution must be disabled before it can be deleted. Updating and deleting a Distribution may take several minutes.

### Amazon DynamoDB

Before deleting a table:

- Verify its backup.
- Check Point-in-Time Recovery.
- Confirm that its data is no longer required.
- Do not delete the table while the system is still writing data.

### Amazon Cognito

Deleting a Cognito User Pool also deletes user accounts and related App Client configuration. Confirm that authentication and demonstration are no longer required.

### Lambda and API Gateway

After Lambda functions and API Gateway resources are deleted:

- The REST API stops working.
- WebSocket connections cannot be established.
- The frontend cannot communicate with the backend.
- Real-time auction functionality becomes unavailable.

### Terraform Backend

The `00-bootstrap` module may contain the S3 Bucket used to store Terraform State and the resource used for State locking. Therefore, this module must be processed **last**.

Before deleting the Backend:

- Confirm that all other modules have been destroyed.
- Back up the Terraform State.
- Verify that no Terraform process is running.
- Do not delete the Backend while other modules still depend on its State.

## Verification After Cleanup

After Terraform completes the cleanup, sign in to the AWS Management Console and inspect the services used by the project:

- Amazon Cognito.
- AWS IAM.
- Amazon S3.
- Amazon CloudFront.
- AWS Lambda.
- Amazon API Gateway.
- Amazon DynamoDB.
- Amazon SQS.
- Amazon CloudWatch.
- AWS Backup.
- AWS CodeBuild and other CI/CD resources.

Use AWS Resource Explorer to search for the project resource prefix:

```text
la-
```

Confirm that:

- No unnecessary project resources remain.
- No CloudFront Distribution remains active.
- No project Lambda Function remains.
- No REST API or WebSocket API remains.
- No SQS Queue remains.
- No DynamoDB Table is removed before its required data is backed up.
- No unnecessary S3 Bucket remains.
- No CloudWatch Log Group has unlimited retention unnecessarily.
- No Pipeline or Build Project continues to run.

Finally, open:

```text
AWS Billing and Cost Management
```

Review:

- Bills.
- Cost Explorer.
- Free Tier.
- Budgets.
- Services that continue to generate costs.

{{% notice info %}}
AWS cost information may be delayed. After cleanup, continue monitoring Billing for the following days to confirm that no unexpected resource continues to generate costs.
{{% /notice %}}

## Expected Result

After the cleanup process is performed:

- Live Auction AWS resources are deleted in the correct order.
- Important data has been backed up.
- Terraform State is processed last.
- Unnecessary services no longer generate costs.
- Source code, reports, and deployment documentation remain available on GitHub.
- The system can be redeployed using Terraform when required.

At the time of completing this report, the team has documented only the cleanup procedure and **has not executed `terraform destroy`**, ensuring that the website remains available for project demonstration and evaluation.