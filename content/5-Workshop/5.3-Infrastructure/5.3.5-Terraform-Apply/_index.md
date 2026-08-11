---
title: "Infrastructure Deployment"
date: 2026-07-13
weight: 5
chapter: false
pre: "<b>5.3.5. </b>"
---

## Introduction

After reviewing and approving the deployment plan, the team uses `terraform apply` to create and configure resources on Amazon Web Services.

The `terraform apply` command applies the changes identified in the Terraform plan, including:

* Creating new resources.
* Updating existing resources.
* Replacing resources when the configuration requires them to be recreated.
* Removing resources that are no longer declared.
* Updating Terraform State after a successful deployment.

In the Live Auction system, the infrastructure is divided into multiple independent modules. Each module manages a group of resources and uses a separate Terraform State in the Remote Backend.

This structure helps the team:

* Deploy the infrastructure by functional layer.
* Limit the scope of changes in each deployment.
* Reduce the impact between components.
* Simplify verification and troubleshooting.
* Allow later modules to use the Outputs of previously deployed modules.

{{% notice warning %}}
The Live Auction environment has already been deployed and is currently running on AWS. Do not run `terraform apply` again only to create a screenshot. Applying an unreviewed plan may update, replace, or remove resources currently used by the system.
{{% /notice %}}

---

## Preparations Before Deployment

Before executing `terraform apply`, the team must ensure that:

* AWS CLI is configured for the correct account.
* The correct AWS Region is being used.
* The module has been initialized using `terraform init`.
* The configuration has passed `terraform validate`.
* The plan has been reviewed using `terraform plan`.
* No resources are unexpectedly removed or replaced.
* The Lambda packages and Lambda Layers have been built.
* Terraform State is not locked by another process.
* Dependent modules have already been deployed.

Check the current AWS identity:

```powershell
aws sts get-caller-identity
```

Check the configured Region:

```powershell
aws configure get region
```

The primary Region used by the project is:

```text
ap-southeast-1
```

{{% notice warning %}}
The result of `aws sts get-caller-identity` may contain the AWS Account ID, User ID, and ARN. These identifiers should be partially hidden before a screenshot is included in a public report or repository.
{{% /notice %}}

---

## Preparing Lambda Deployment Packages

Some modules use ZIP archives of Lambda Functions or Lambda Layers to calculate `source_code_hash`.

The build artifacts are not stored directly in the Git repository. Therefore, they must be generated on the deployment computer before executing `terraform plan` or `terraform apply`.

For example, build the Post Confirmation Lambda used by the Identity module:

```powershell
cd "D:\ThucTap\Live-Auction"
.\backend\build.ps1 -Target function -FunctionName cognito_post_confirm
```

Verify that the file has been created:

```powershell
Test-Path .\backend\build\cognito_post_confirm.zip
```

If the build process succeeds, the command returns:

```text
True
```

The remaining Lambda Functions and Lambda Layers must also be built before deploying the Compute module.

{{% notice info %}}
The `backend/build.ps1` script uses Docker to generate Lambda packages for the required runtime environment. Docker Desktop must be running before the script is executed.
{{% /notice %}}

---

## Reviewing the Plan Before Apply

If a plan has been saved using:

```powershell
terraform plan -out="tfplan"
```

the team can review it using:

```powershell
terraform show tfplan
```

The following items must be reviewed:

* The number of resources to be created.
* Resources to be updated.
* Resources to be removed.
* Resources to be replaced.
* Resource names and AWS Regions.
* IAM Roles and IAM Policies.
* Expected Output values.
* Changes to Lambda Functions.
* Changes to DynamoDB Tables and S3 Buckets.

{{% notice warning %}}
Do not continue the deployment if the plan contains unexpected resource removals or replacements. Stop the process, review the configuration and Terraform State, and run `terraform plan` again.
{{% /notice %}}

---

## Executing Terraform Apply

If the plan has been saved to a `tfplan` file, apply that exact plan using:

```powershell
terraform apply "tfplan"
```

When a saved plan is used, Terraform does not ask for the `yes` confirmation again.

If a saved plan is not used, run:

```powershell
terraform apply
```

Terraform displays the plan and asks for confirmation:

```text
Do you want to perform these actions?

  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Enter:

```text
yes
```

to begin the deployment.

{{% notice info %}}
The `terraform plan -out="tfplan"` and `terraform apply "tfplan"` workflow ensures that Terraform applies only the plan reviewed by the team.
{{% /notice %}}

After a successful deployment, Terraform displays a message in the following format:

```text
Apply complete! Resources: ... added, ... changed, ... destroyed.
```

The number of resources depends on the module and the infrastructure state at the time of deployment.

A fixed number of resources should not be used as a general result for the entire system.

---

## Deployment Evidence

Because the infrastructure was deployed previously, evidence of the `terraform apply` execution should be collected from:

* The Terminal of the member who performed the deployment.
* GitHub Actions or another CI/CD system.
* AWS CodeBuild logs.
* Deployment records retained by the team.

Do not execute `terraform apply` again only to create a screenshot for the report.

Because the infrastructure was previously deployed by another team member and the original Terminal output was not retained, the team verifies the deployment through Terraform State, Terraform Output, and the actual resources on AWS Management Console.

These verification results confirm that the resources were created, are managed by Terraform, and are operating in the AWS environment.

---

## Module Deployment Order

The infrastructure is organized according to functional layers and dependencies.

The general deployment order is:

1. Bootstrap.
2. Foundation.
3. Identity.
4. Data.
5. Messaging.
6. Compute.
7. API.
8. Edge.
9. Observability.
10. Security.
11. Backup and Disaster Recovery.
12. CI/CD.

The corresponding source code modules are:

| Order | Module | Primary components |
| --- | --- | --- |
| 1 | `00-bootstrap` | S3 Remote Backend and Terraform State locking. |
| 2 | `01-foundation` | Shared foundational resources. |
| 3 | `03-identity` | Cognito, IAM, Post Confirmation Lambda, and CloudWatch Log Group. |
| 4 | `04-data` | DynamoDB Tables and the S3 Bucket used for media storage. |
| 5 | `05-messaging` | SQS FIFO, Dead-letter Queues, and EventBridge Scheduler. |
| 6 | `06-compute` | Lambda Functions, Lambda Layers, IAM Roles, and Event Source Mappings. |
| 7 | `06-compute/stage3-control-plane` | Lambda Functions for sessions, items, queries, and administration operations. |
| 8 | `07-api` | REST API, WebSocket API, Authorizers, Routes, Integrations, and Stages. |
| 9 | `09-edge` | S3 and CloudFront for the User Frontend, Admin Frontend, and media content. |
| 10 | `10-observability` | Logs, metrics, dashboards, and alarms. |
| 11 | `11-security` | Additional security configurations. |
| 12 | `12-backup-dr` | Backup and disaster recovery support. |
| 13 | `13-cicd` | Resources used by the CI/CD process. |

The actual order may be adjusted by the deployment pipeline. However, modules using `terraform_remote_state` can only be deployed after the State of their dependencies exists.

---

## Deploying the Identity Module

The Identity module deploys the authentication and authorization components.

Navigate to the module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\03-identity"
```

Deployment workflow:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

This module manages:

* Amazon Cognito User Pool.
* Cognito User Pool App Client.
* Cognito User Group for Users.
* Cognito User Group for Administrators.
* Post Confirmation Lambda.
* IAM Role and IAM Policy for the Lambda Function.
* Lambda Permission allowing Cognito to invoke the Lambda Function.
* CloudWatch Log Group.

---

## Deploying the Data Module

Navigate to the module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\04-data"
```

Deployment workflow:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

The Data module deploys DynamoDB Tables used for:

* Auction item state.
* Bid events.
* WebSocket connections.
* Bidder aliases.
* Idempotency control.
* Auction session catalog.
* Product category catalog.
* Administration audit events.

This module also deploys the S3 Bucket used to store media data for products and auction sessions.

---

## Deploying the Messaging Module

Navigate to the module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\05-messaging"
```

Deployment workflow:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

The Messaging module deploys:

* The SQS Queue that receives bid commands.
* A Dead-letter Queue for messages that cannot be processed.
* An EventBridge Scheduler Schedule Group.
* A Scheduler Dead-letter Queue.
* IAM Role and IAM Policy for the Scheduler.
* The mechanism used to process requests in order.

---

## Deploying the Compute Module

Before deploying the Compute module, the team builds the Lambda packages and Lambda Layer using the script in the `backend` directory.

For example, build all packages:

```powershell
cd "D:\ThucTap\Live-Auction"
.\backend\build.ps1 -Target all
```

After the build is completed, navigate to the Compute module:

```powershell
cd infra\06-compute
```

Deployment workflow:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

The Compute module deploys:

* Bid Processor Lambda.
* WebSocket Authorizer Lambda.
* WebSocket Handler Lambda.
* Broadcast Lambda.
* Shared Lambda Layer.
* IAM Role and IAM Policy for each Lambda Function.
* CloudWatch Log Groups.
* Event Source Mapping between SQS and Lambda.
* Environment Variables used to connect the services.

The `stage3-control-plane` module additionally deploys:

* Session Service Lambda.
* Item Service Lambda.
* Query Service Lambda.
* Admin Command Lambda.
* Shared Lambda Layer.
* EventBridge Scheduler for auction session lifecycle operations.

---

## Deploying the API Module

Navigate to the module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\07-api"
```

Deployment workflow:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

The API module deploys:

* REST API.
* REST API Resources and Methods.
* Lambda Integrations.
* API Gateway Stage.
* API Gateway Deployment.
* API Key and Usage Plan.
* WebSocket API.
* WebSocket Authorizer.
* `$connect` Route.
* `$disconnect` Route.
* `join_room` Route.
* `place_bid` Route.
* Lambda Permissions for API Gateway.
* CloudWatch Access Logs.

---

## Deploying the Edge Module

The Edge module is deployed after the backend and APIs are available.

Navigate to the module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\09-edge"
```

Deployment workflow:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

The Edge module deploys:

* S3 Bucket for the User Frontend.
* S3 Bucket for the Admin Frontend.
* CloudFront Distribution for the User Frontend.
* CloudFront Distribution for the Admin Frontend.
* CloudFront Distribution for media content.
* Origin Access Control.
* S3 Bucket Policies.
* Server-side Encryption.
* S3 Public Access Block.
* Default Root Object.
* Static content distribution rules.

---

## Deploying the Operational Modules

After the primary functional components are available, the team deploys the following modules.

### Observability

```text
infra/10-observability
```

This module provides logs, metrics, dashboards, and alarms for monitoring the system.

### Security

```text
infra/11-security
```

This module adds security configurations and controls to the infrastructure.

### Backup and Disaster Recovery

```text
infra/12-backup-dr
```

This module configures backup and recovery capabilities for system failures.

### CI/CD

```text
infra/13-cicd
```

This module deploys the components that support automated building, testing, and deployment.

Each module follows the basic workflow:

```powershell
terraform init
terraform validate
terraform plan -out="tfplan"
terraform apply "tfplan"
```

---

## Checking Terraform State After Deployment

After the infrastructure has been deployed, the resources managed by Terraform can be verified using:

```powershell
terraform state list
```

For example, in the Identity module:

```powershell
cd "D:\ThucTap\Live-Auction\infra\03-identity"
terraform state list
```

The result displays resources such as:

```text
aws_cloudwatch_log_group.cognito_post_confirm
aws_cognito_user_group.admin
aws_cognito_user_group.user
aws_cognito_user_pool.main
aws_cognito_user_pool_client.web
aws_iam_role.cognito_post_confirm
aws_iam_role_policy.cognito_post_confirm
aws_lambda_function.cognito_post_confirm
aws_lambda_permission.cognito_post_confirm
```

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-state-list.png" alt="Resources recorded in Terraform State" width="85%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.18.</b> Identity module resources managed through Terraform State.
    </figcaption>
</figure>

{{% notice warning %}}
Do not directly modify Terraform State. Incorrect State operations may prevent Terraform from managing the infrastructure correctly or cause unintended changes.
{{% /notice %}}

---

## Checking Terraform Output

After deployment, Terraform Output is used to retrieve information required to connect the modules, frontend applications, backend services, and AWS resources.

From the Identity module, run:

```powershell
cd "D:\ThucTap\Live-Auction\infra\03-identity"
terraform output
```

The command returns:

```text
PS D:\ThucTap\Live-Auction\infra\03-identity> terraform output

cognito_client_id = "2ttqnjt0nmttmi655dav*******"
cognito_issuer = "https://cognito-idp.ap-southeast-1.amazonaws.com/ap-southeast-1_1Ly4*****"
cognito_jwks_url = "https://cognito-idp.ap-southeast-1.amazonaws.com/ap-southeast-1_1Ly4*****/.well-known/jwks.json"
cognito_user_pool_arn = "arn:aws:cognito-idp:ap-southeast-1:************:userpool/ap-southeast-1_1Ly4*****"
cognito_user_pool_client_id = "2tt*********************703g"
cognito_user_pool_id = "ap-southeast-1_1Ly4*****"

Warning: Deprecated Parameter

The parameter "dynamodb_table" is deprecated.
Use parameter "use_lockfile" instead.
```

The Outputs have the following roles:

| Output | Role |
| --- | --- |
| `cognito_client_id` | ID of the Cognito App Client used by the frontend when sending authentication requests. |
| `cognito_issuer` | Address of the JWT issuer for the User Pool. |
| `cognito_jwks_url` | Address providing the public keys used by the backend or Authorizer to verify JWT signatures. |
| `cognito_user_pool_arn` | ARN identifying the Cognito User Pool on AWS. |
| `cognito_user_pool_client_id` | ID of the Cognito App Client exported for use by other components. |
| `cognito_user_pool_id` | ID of the Cognito User Pool deployed for the system. |

These Outputs are used to:

* Configure the User Frontend and Admin Frontend.
* Send registration and sign-in requests to Amazon Cognito.
* Verify JWTs in the backend or API Authorizer.
* Pass Identity information to dependent modules.
* Identify the User Pool and App Client currently in use.
* Connect backend services to the authentication system.

To retrieve an individual Output, run:

```powershell
terraform output cognito_user_pool_id
```

To retrieve the value without quotation marks, use `-raw`:

```powershell
terraform output -raw cognito_user_pool_id
```

To export all Outputs as JSON:

```powershell
terraform output -json
```

The JSON result can be consumed by deployment scripts or passed to subsequent configuration steps.

{{% notice info %}}
The `dynamodb_table is deprecated` warning appears because the Terraform Backend currently uses a DynamoDB table to lock Terraform State. The configuration was still operational at the time of verification, but newer Terraform versions recommend using the S3 Backend `use_lockfile` option. This is a future compatibility warning, not an error from `terraform output`.
{{% /notice %}}

{{% notice warning %}}
The Cognito User Pool ID, App Client ID, and ARN are not passwords or Secret Keys. However, these values still disclose information about the AWS resource identifiers and infrastructure structure. Because the report repository is Public, the team partially hides the Account ID, User Pool ID, App Client ID, and ARN before including the output in the report. Never disclose a Client Secret, Access Token, Refresh Token, password, or AWS credential.
{{% /notice %}}

The `terraform output` result confirms that the Identity module has been deployed, Terraform State contains the output values, and other components can use these values to connect to Amazon Cognito.

---

## Confirming Synchronization After Deployment

After deployment, the team runs:

```powershell
terraform plan -no-color
```

If the infrastructure matches the configuration, Terraform displays:

```text
No changes. Your infrastructure matches the configuration.
```

This result confirms that:

* AWS resources were deployed according to the configuration.
* Terraform State was updated.
* No unapplied changes remain.
* The Terraform source code and actual infrastructure are synchronized.

The `No changes` screenshot was already presented in **Section 5.3.4 – Reviewing the Deployment Plan**, so the same image is not repeated in this section.

---

## Checking Resources on AWS Management Console

In addition to Terraform State, the team signs in to AWS Management Console and verifies:

* Cognito User Pool and App Client.
* IAM Roles and IAM Policies.
* DynamoDB Tables.
* SQS FIFO Queues and Dead-letter Queues.
* Lambda Functions and Lambda Layers.
* REST API and WebSocket API.
* S3 Buckets.
* CloudFront Distributions.
* CloudWatch Log Groups and Alarms.
* CI/CD resources.

The AWS Management Console verification confirms that the resources were created and currently exist in the AWS account.

A consolidated resource list is presented in **Section 5.4 – Deployed AWS Services**.

---

## Common Errors

### Missing Lambda Package

Terraform may return:

```text
Call to function "filebase64sha256" failed
```

or:

```text
The system cannot find the file specified.
```

This error occurs when the Lambda ZIP archive has not been built on the current computer.

For example, build the Post Confirmation Lambda:

```powershell
cd "D:\ThucTap\Live-Auction"
.\backend\build.ps1 -Target function -FunctionName cognito_post_confirm
```

### Saved Plan Is Stale

Terraform may return:

```text
Saved plan is stale
```

This error occurs when the configuration or State changes after the `tfplan` file has been created.

Create a new plan:

```powershell
terraform plan -out="tfplan"
```

Review the new plan before applying it.

### Insufficient AWS Permissions

The following message may appear:

```text
AccessDenied
```

Check the AWS identity:

```powershell
aws sts get-caller-identity
```

Then review the IAM Policy of the account or Role performing the deployment.

### State Is Locked

Terraform may return an error when another process is operating on the same State.

Do not manually remove the lock or use `force-unlock` until the previous process has been confirmed as completed.

### Resource Exists Outside Terraform State

If a resource was created manually but is not managed through Terraform State, the apply operation may fail because of a duplicate resource name.

Review the existing resource and consider using `terraform import` instead of recreating it.

### CloudFront Is Not Updated Immediately

After deployment, CloudFront may require additional time to enter the `Deployed` state. Wait for the distribution to complete before testing the frontend.

---

## Resource Destruction Warning

The following command:

```powershell
terraform destroy
```

removes the resources managed by the current module.

{{% notice danger %}}
Do not run `terraform destroy` against the active Live Auction environment. This command may remove the Cognito User Pool, Lambda Functions, DynamoDB Tables, SQS Queues, API Gateway APIs, S3 Buckets, or other critical resources, causing system interruption and data loss.
{{% /notice %}}

Resources should only be destroyed in a test environment after:

* Identifying the exact module and resource scope.
* Backing up the required data.
* Reviewing the destruction plan.
* Receiving approval from the team.
* Confirming that users will not be affected.

---

## Results

After the infrastructure deployment was completed:

* The Remote Backend and Terraform State were configured.
* The modules were deployed according to their dependencies.
* Amazon Cognito and IAM resources were created.
* The Post Confirmation Lambda was integrated with Cognito.
* Amazon DynamoDB Tables were deployed.
* The media storage S3 Bucket was configured.
* Amazon SQS FIFO and Dead-letter Queues were created.
* EventBridge Scheduler was configured for time-based operations.
* AWS Lambda Functions and Lambda Layers were deployed.
* The REST API and WebSocket API were created through Amazon API Gateway.
* The User Frontend and Admin Frontend were stored in separate S3 Buckets.
* CloudFront Distributions were configured for the frontend applications and media content.
* Monitoring, security, backup, and CI/CD components were added.
* Terraform State recorded the managed resources.
* Terraform Output provided the values required by dependent components.
* Post-deployment verification confirmed that the Terraform source code and AWS infrastructure were synchronized.
* The infrastructure was ready for detailed service verification in the next section.