---
title: "Reviewing the Deployment Plan"
date: 2026-07-13
weight: 4
chapter: false
pre: "<b>5.3.4. </b>"
---

## Introduction

After initializing the Terraform environment, the team checks the configuration and reviews the deployment plan using `terraform fmt -check`, `terraform validate`, and `terraform plan`.

The `terraform plan` command reads the Terraform configuration, the infrastructure state stored in the Remote Backend, and the actual state of the AWS resources. Terraform then compares the current state with the desired configuration to determine which resources need to be created, updated, replaced, or removed.

Reviewing the plan before deployment helps the team:

* Detect syntax errors and invalid configurations.
* Review expected changes before they affect AWS resources.
* Detect resources that may be removed or replaced.
* Prevent unintended changes to the running system.
* Confirm that Terraform State is synchronized with the actual infrastructure.
* Verify the infrastructure after the deployment has been completed.

{{% notice info %}}
The `terraform validate` and `terraform plan` commands do not directly create, update, or remove AWS resources. Resources are only modified when `terraform apply` is executed and the deployment plan is approved.
{{% /notice %}}

---

## Navigating to the Module

In this section, the `03-identity` module is used as a representative example for validating the configuration and reviewing the deployment plan.

Open PowerShell from the project root directory:

```powershell
cd "D:\ThucTap\Live-Auction"
```

Navigate to the Identity module:

```powershell
cd infra\03-identity
```

Check the current location:

```powershell
Get-Location
```

The result should show that the Terminal is working in:

```text
D:\ThucTap\Live-Auction\infra\03-identity
```

If the module has not been initialized on the current computer, run:

```powershell
terraform init
```

---

## Checking Terraform Formatting

Before validating the configuration and creating a plan, the team checks the formatting of the Terraform files:

```powershell
terraform fmt -check
```

The `terraform fmt -check` command only checks the formatting and does not automatically modify any `.tf` files.

If all files are formatted correctly, the command completes without displaying an error. If the Terminal returns the name of one or more files, those files do not follow the standard Terraform formatting rules.

During development, the following command can be used to automatically format the files:

```powershell
terraform fmt
```

However, `terraform fmt` should not be executed only for the purpose of taking screenshots if there is no intention to update the source code, because the command may modify files managed by Git.

The Git working tree can be checked afterward using:

```powershell
git status
```

---

## Validating the Terraform Configuration

After checking the formatting, run:

```powershell
terraform validate
```

The `terraform validate` command checks:

* The syntax of the `.tf` files.
* Variable names and data types.
* Required resource attributes.
* References between resources.
* Provider and module configurations.
* The overall consistency of the Terraform configuration.

When the configuration is valid, Terraform displays:

```text
Success! The configuration is valid.
```

{{% notice warning %}}
The `terraform validate` command only checks the syntax and internal consistency of the configuration. A successful result does not confirm that the current AWS account has sufficient permissions to access the Backend or manage the declared resources.
{{% /notice %}}

---

## Creating the Deployment Plan

After the configuration has been validated, run:

```powershell
terraform plan -no-color
```

The `-no-color` option removes color codes from the output, making the Terminal content easier to read and capture for the report.

During execution, Terraform performs the following operations:

1. Reads the configuration files in the module.
2. Connects to the Remote Backend.
3. Reads the current Terraform State.
4. Queries the state of the AWS resources.
5. Compares the actual infrastructure with the desired configuration.
6. Identifies resources that need to be created, updated, replaced, or removed.
7. Displays the deployment plan for review.

Terraform uses the following symbols in the plan output:

| Symbol | Meaning                                     |
| ------ | ------------------------------------------- |
| `+`    | The resource will be created.               |
| `~`    | The resource will be updated in place.      |
| `-`    | The resource will be removed.               |
| `-/+`  | The resource will be removed and recreated. |
| `<=`   | Data will be read from a Data Source.       |

For example, before the initial deployment, Terraform may display:

```text
Plan: 3 to add, 0 to change, 0 to destroy.
```

This result means:

* `3 to add`: three resources will be created.
* `0 to change`: no resources will be updated.
* `0 to destroy`: no resources will be removed.

The actual number of resources depends on the source code and infrastructure state at the time the command is executed. Therefore, the values in the example above should not be treated as the official result of the system.

---

## Reviewing the Plan After Deployment

The Live Auction system has already been deployed on AWS. Therefore, when `terraform plan` is executed again using the latest source code and Terraform State, the expected result is:

```text
No changes. Your infrastructure matches the configuration.
```

This message indicates that:

* Terraform State matches the current configuration.
* The AWS resources match the desired state.
* No resources need to be created.
* No resources need to be updated.
* No resources need to be removed or replaced.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-plan-no-changes.png" alt="Terraform Plan detected no changes" width="90%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.16.</b> Terraform confirms that the current infrastructure matches the Identity module configuration.
    </figcaption>
</figure>

{{% notice warning %}}
If the result contains `to add`, `to change`, `to destroy`, or the `-/+` symbol, do not immediately execute `terraform apply`. The source code, Terraform State, and actual resources must be reviewed to determine the cause of the changes.
{{% /notice %}}

---

## Saving a Deployment Plan

During the initial deployment, the team can save the deployment plan to a `tfplan` file using:

```powershell
terraform plan -out="tfplan"
```

The `-out` option saves the exact plan created at the time of the review. This file can then be used during deployment:

```powershell
terraform apply "tfplan"
```

To review a saved plan, run:

```powershell
terraform show tfplan
```

The `tfplan` file currently present in the `03-identity` module was created during a previous planning process. It was not generated by `terraform init`.

{{% notice warning %}}
The `tfplan` file is a binary file and may contain resource names, ARNs, infrastructure configuration, or sensitive values. It should not be edited manually or pushed to a public repository. A saved plan may also become outdated if the source code or infrastructure state changes.
{{% /notice %}}

Because the system has already been deployed, the team does not need to recreate the `tfplan` file only to capture a screenshot. The `terraform plan -no-color` result showing no changes is sufficient to demonstrate that the current source code and infrastructure are synchronized.

---

## Terraform Modules of the System

The current infrastructure is divided into the following modules:

| Module             | Role                                                                                                          |
| ------------------ | ------------------------------------------------------------------------------------------------------------- |
| `00-bootstrap`     | Initializes the S3 Backend and Terraform State management mechanism.                                          |
| `01-foundation`    | Prepares the shared foundational components of the system.                                                    |
| `03-identity`      | Deploys the Cognito User Pool, App Client, User Groups, IAM resources, and Post Confirmation Lambda.          |
| `04-data`          | Deploys the DynamoDB tables and the S3 Bucket used to store media data.                                       |
| `05-messaging`     | Deploys SQS FIFO, Dead-letter Queues, and EventBridge Scheduler components.                                   |
| `06-compute`       | Deploys Lambda Functions, Lambda Layers, IAM Roles, and Event Source Mappings.                                |
| `07-api`           | Deploys the REST API, WebSocket API, Authorizers, Routes, Integrations, and Stages.                           |
| `09-edge`          | Deploys the S3 Buckets and CloudFront Distributions for the User Frontend, Admin Frontend, and media content. |
| `10-observability` | Configures monitoring, logging, alarms, and system observability.                                             |
| `11-security`      | Adds security configurations and protective measures to the infrastructure.                                   |
| `12-backup-dr`     | Configures backup and disaster recovery capabilities.                                                         |
| `13-cicd`          | Deploys resources used by the continuous integration and continuous deployment process.                       |

Each module has its own Remote Backend and Terraform State. Separating the State by module helps the team:

* Limit the scope of infrastructure changes.
* Reduce the impact between infrastructure layers.
* Review the plan of each component independently.
* Deploy modules according to their dependencies.
* Reduce the risk of multiple team members modifying the same State.

---

## Items to Review in the Plan

Before approving a deployment plan, the team reviews the following items.

### Resource Names

The names of S3 Buckets, DynamoDB Tables, Lambda Functions, API Gateway APIs, SQS Queues, and other resources must follow the naming convention of the project.

The primary resources of the system use the following prefix:

```text
la-
```

### AWS Region

Regional resources are deployed in:

```text
ap-southeast-1
```

This Region corresponds to **Asia Pacific (Singapore)**.

Some global services, such as AWS IAM and Amazon CloudFront, are not managed within a single Region in the same way as services such as Lambda or DynamoDB.

### IAM Permissions

IAM Policies must follow the principle of least privilege. Each service should only be granted the permissions required to perform its responsibilities.

### Resources Scheduled for Removal or Replacement

If the plan displays:

```text
Plan: 0 to add, 0 to change, 1 to destroy.
```

or the following symbol:

```text
-/+
```

the team must carefully review the plan before continuing. Removing or replacing a resource may:

* Interrupt the system.
* Change an Endpoint or ARN.
* Cause data loss if the resource is not protected.
* Affect dependent modules.
* Cause the frontend to lose its connection to the backend.

### Output Values

The expected Output values include:

* Cognito User Pool ID.
* Cognito App Client ID.
* DynamoDB table names.
* SQS Queue URL.
* REST API Endpoint.
* WebSocket Endpoint.
* S3 Bucket name.
* CloudFront domain name.
* Lambda Function names.
* ARNs consumed by other modules.

Full Account IDs, ARNs, or sensitive authentication values should not be included in screenshots used in a public report.

---

## Common Errors

### Terraform Has Not Been Initialized

The following message may be displayed:

```text
Backend initialization required
```

Initialize the module using:

```powershell
terraform init
```

If the Backend configuration has recently been changed and reviewed by the team, run:

```powershell
terraform init -reconfigure
```

### Insufficient AWS Permissions

The following message may appear:

```text
AccessDenied
```

Check the current AWS CLI identity:

```powershell
aws sts get-caller-identity
```

Then review the IAM Policy assigned to the current account or Role.

### Unable to Access the Remote Backend

Check the following:

* Whether the S3 Bucket used to store Terraform State still exists.
* Whether the Bucket name in `backend.tf` is correct.
* Whether the configured AWS Region is correct.
* Whether the current account can read the State stored in S3.
* Whether the DynamoDB table used for State locking is available.

### Terraform State Is Locked

Terraform may return a locking error when another process is currently using the State.

Do not manually remove or force-unlock the State before confirming that the previous process has finished. Multiple team members operating on the same module at the same time may cause conflicts and inconsistent infrastructure state.

### The Plan Contains Unexpected Changes

If the system has already been deployed but `terraform plan` still detects changes, check:

* Whether the local source code is at the latest commit.
* Whether the Terminal is currently in the correct module.
* Whether the AWS credentials point to the correct account.
* Whether the Backend points to the correct State.
* Whether resources have been manually modified through AWS Management Console.
* Whether a Lambda deployment package or build artifact has changed.
* Whether the environment variables and Provider configuration are correct.

Do not execute `terraform apply` until the cause of the unexpected changes has been identified.

---

## Results

After reviewing the deployment plan:

* The formatting of the Terraform files was checked using `terraform fmt -check`.
* The Identity module configuration was successfully validated using `terraform validate`.
* Terraform successfully connected to the Remote Backend.
* Terraform read the current infrastructure state from Terraform State and AWS.
* The `terraform plan` command compared the source code with the deployed resources.
* No resources needed to be created, updated, replaced, or removed.
* Terraform State and the current infrastructure were synchronized.
* The verification process did not modify the deployed AWS resources.
* The system was ready for the deployment workflow verification in the next section.