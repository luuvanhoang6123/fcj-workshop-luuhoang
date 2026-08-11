---
title: "Environment Preparation"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---

## Introduction

Before deploying the **Live Auction** system on **Amazon Web Services (AWS)**, the team needs to prepare the development environment, AWS accounts, and supporting tools. A consistent setup helps the application development, infrastructure deployment with **Terraform**, and system testing processes run smoothly.

## Project Source Code

The Live Auction source code is stored on GitHub. The team uses the `develop` branch to update and synchronize source code during development.

GitHub link: [GitHub Repository – Live Auction](https://github.com/CallmeSen/Live-Auction/tree/develop)

## Required Tools

- AWS account.
- Git.
- AWS CLI.
- Terraform.
- Docker Desktop.
- Node.js.
- Python 3.
- Visual Studio Code or an equivalent IDE.

## Preparing the Source Code

Clone the project source code from the `develop` branch:

```powershell
git clone -b develop https://github.com/CallmeSen/Live-Auction.git
cd Live-Auction
```

After cloning, the main project structure includes:

```text
backend/
frontend/
admin-frontend/
infra/
docker-compose.yml
```

The directories are described as follows:

- `backend/`: FastAPI backend source code.
- `frontend/`: Interface for users and members.
- `admin-frontend/`: Interface for administrators.
- `infra/`: Terraform source code for deploying AWS infrastructure.
- `docker-compose.yml`: Configuration for running the components in the local environment.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/project-structure.png"
    title="Figure 5.2.1: Main project directory structure"
    width="60%"
>}}

## Creating and Configuring the `la-admin` IAM User

To manage the AWS environment during the workshop, the team creates an IAM User named `la-admin`. This account is used to sign in to the AWS Management Console, create IAM Users for team members, and perform the management tasks required within the project scope.

The Root account is used only for high-level AWS Account configuration. For daily tasks, the team uses IAM Users to reduce security risks.

### Step 1: Open the IAM Service

Sign in to the AWS Management Console using an account with administrative permissions.

In the service search bar, enter:

```text
IAM
```

Then open:

```text
Identity and Access Management (IAM)
```

In the left navigation panel, select:

```text
Access management → Users
```

### Step 2: Create a New IAM User

On the IAM User list page, select:

```text
Create user
```

In the **User details** section, enter the user name:

```text
la-admin
```

Enable access to the AWS Management Console. The `la-admin` account requires a console password to manage AWS resources and create accounts for other team members.

When creating a password, the team can use an AWS-generated password or a custom password that meets the security policy. The user should change the password after the first sign-in if AWS requires it.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-create-user.jpg"
    title="Figure 5.2.2: Specifying the la-admin IAM User details"
    width="90%"
>}}

### Step 3: Assign Permissions to `la-admin`

After specifying the user name, continue to:

```text
Set permissions
```

During the workshop, the `la-admin` account is assigned the following policies:

- `AdministratorAccess`: Allows management of AWS resources required for the workshop environment.
- `IAMUserChangePassword`: Allows the user to change their own console password.

After checking the user name and assigned permissions, select:

```text
Create user
```

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-permissions.png"
    title="Figure 5.2.3: Policies assigned to the la-admin IAM User"
    width="90%"
>}}


Administrative access is suitable only for a management account in a learning and testing environment. In production, the **least-privilege** principle should be applied, meaning that each account receives only the permissions required for its assigned tasks.

### Step 4: Store Console Sign-In Information

After successful creation, AWS displays the AWS Management Console sign-in information for the `la-admin` IAM User.

This information includes:

- Console sign-in URL.
- User name.
- Initial password or instructions for downloading sign-in information.

The team stores this information securely for signing in with the `la-admin` account. Passwords must not be stored in GitHub repositories, source files, or shared publicly.

A screenshot showing the password is not required in the report. If a user-creation confirmation screenshot is used, the Console sign-in URL containing the Account ID and all password-related information must be hidden.

### Step 5: Configure MFA for `la-admin`

After creating the user, open:

```text
IAM → Users → la-admin → Security credentials
```

In the **Multi-factor authentication (MFA)** section, select:

```text
Assign MFA device
```

The team selects:

```text
Authenticator app
```

Then, an authenticator application on a mobile device is used to scan the QR code and enter the verification codes required by AWS.

MFA helps protect the account because users must provide an additional verification code from the MFA device when signing in.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-mfa-setup.jpg"
    title="Figure 5.2.4: Selecting an Authenticator app to configure MFA for la-admin"
    width="90%"
>}}

After completing the verification, check the MFA status in the **Security credentials** tab. The account should display an assigned MFA device.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-mfa-enabled.jpg"
    title="Figure 5.2.5: MFA configured for the la-admin IAM User"
    width="90%"
>}}


### Step 6: Create an Access Key for AWS CLI and Terraform

To use AWS CLI and Terraform on a local computer, the team creates an Access Key for the `la-admin` IAM User.

Open:

```text
IAM → Users → la-admin → Security credentials
```

In the **Access keys** section, select:

```text
Create access key
```

For the use case, select:

```text
Command Line Interface (CLI)
```

Confirm that the AWS security recommendation has been reviewed, then select:

```text
Next
```

An optional Description tag can be added to describe the Access Key purpose, for example:

```text
AWS CLI and Terraform for Live Auction workshop
```

Then select:

```text
Create access key
```

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-access-key-purpose.jpg"
    title="Figure 5.2.6: Selecting the Access Key use case for AWS CLI"
    width="90%"
>}}

After the Access Key is created, AWS displays the **Secret Access Key** only once. The team stores this value only in the personal development environment for AWS CLI configuration and never uploads it to GitHub.


### Step 7: Verify the IAM User

After completing the configuration, open:

```text
IAM → Users
```

Verify the IAM User:

```text
la-admin
```

The account should meet the following conditions:

- The user was created successfully.
- Appropriate permissions were assigned for managing the workshop environment.
- The account can sign in to the AWS Management Console.
- MFA has been configured.
- An Access Key is available for AWS CLI and Terraform if local deployment is required.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/iam-la-admin-user-list.jpg"
    title="Figure 5.2.7: The la-admin IAM User in the AWS user list"
    width="90%"
>}}

## Cost Estimate

The team uses the AWS Billing Dashboard and Cost Explorer to monitor costs generated during the deployment of the Live Auction system. The diagram below presents AWS costs grouped by service.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/aws-cost-breakdown.jpg"
    title="Figure 5.2.8: AWS cost breakdown by service"
    width="100%"
>}}

According to the chart, the main costs are generated by Amazon API Gateway and AWS Config. Amazon S3, Amazon DynamoDB, and AWS Secrets Manager generate only minor costs during the testing phase.

At the workshop scale and with a low number of users, the expected cost remains low. Services such as AWS Lambda, Amazon DynamoDB on-demand, Amazon S3, Amazon SQS FIFO, and Amazon Cognito may remain within the Free Tier or generate only small charges.

The actual cost depends on:

- Resource runtime.
- Number of requests to API Gateway and Lambda.
- Data storage size in Amazon S3 and DynamoDB.
- Data transfer through CloudFront.
- Number of messages processed by SQS FIFO.
- CloudWatch log retention time.
- Number of resources monitored by AWS Config.

### Scaling Scenario

If the system is used with a larger number of users and bidding requests, costs may increase due to the following factors:

- Amazon API Gateway costs increase with the number of API requests.
- AWS Lambda costs increase with invocation count and execution duration.
- Amazon DynamoDB costs increase with read/write operations and storage capacity.
- Amazon SQS FIFO costs increase with the number of processed messages.
- Amazon CloudFront costs increase with content delivery traffic.
- Amazon S3 costs increase with storage size and request volume.
- CloudWatch costs increase when logs and metrics are retained for a long time.
- AWS Config costs increase when more resources and configuration records are monitored.

To control costs as the system scales, the team can use billing alarms, limit Lambda concurrency, configure CloudWatch log retention, apply S3 Lifecycle Policies, and remove unused resources.

Therefore, actual costs depend not only on the number of deployed services but also on traffic volume, transaction volume, and AWS resource retention time.

## Installing and Configuring AWS CLI

After preparing the IAM account, the team checks the AWS CLI version:

```powershell
aws --version
```

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/aws-version.png"
    title="Figure 5.2.9: Checking the AWS CLI version"
    width="80%"
>}}

Next, the team configures AWS account credentials:

```powershell
aws configure
```

The command requires the following information:

- AWS Access Key ID.
- AWS Secret Access Key.
- Default Region, for example `ap-southeast-1`.
- Default Output Format.

Credentials are stored only in the personal development environment and must not be uploaded to GitHub, included in screenshots, or written in report source code.

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/aws-configure.png"
    title="Figure 5.2.10: Configuring AWS CLI with the aws configure command"
    width="80%"
>}}

## Installing Terraform

After installing Terraform, the team checks the version with the following command:

```powershell
terraform version
```

{{< figure
    src="/images/5-Workshop/5.2-Prerequisite/terraform-version.png"
    title="Figure 5.2.11: Checking the Terraform version"
    width="60%"
>}}

Terraform is used to initialize, validate, deploy, and manage AWS infrastructure according to the Infrastructure as Code model.

## Results

After completing the preparation steps, the team has:

- Cloned and managed the project source code on GitHub.
- Prepared the Backend, User Frontend, and Admin Frontend environments.
- Created the `la-admin` IAM User to manage the AWS environment.
- Configured permissions and MFA for the administrative account.
- Created an Access Key for AWS CLI and Terraform.
- Installed and configured AWS CLI.
- Installed Terraform for infrastructure deployment.
- Prepared Docker Desktop, Node.js, Python, and Git for system development.
- Monitored costs and active AWS resources.

In the next section, the team initializes the Terraform environment, validates the configuration, and creates an infrastructure deployment plan for the Live Auction system.