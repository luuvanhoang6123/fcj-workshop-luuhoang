---
title: "AWS IAM and Amazon Cognito"
date: 2026-07-27
weight: 1
chapter: false
pre: " <b> 5.4.1. </b> "
---

## Overview

The **Live Auction** system uses **AWS Identity and Access Management (IAM)** and **Amazon Cognito** for two different purposes:

* **AWS IAM** manages access permissions between AWS services.
* **Amazon Cognito** manages accounts, authenticates users, and distinguishes User permissions from Admin permissions.

AWS IAM is not used to create application user accounts. Instead, User and Admin accounts are managed through an Amazon Cognito User Pool.

All IAM and Cognito resources are declared with Terraform in the following module:

```text
infra/03-identity
```

After the Terraform deployment was completed, the team verified the deployed resources directly through the AWS Management Console.


## AWS Identity and Access Management

### Role of AWS IAM

AWS IAM provides Roles, Policies, and permission mechanisms that allow AWS services in the system to communicate with each other.

Permissions are configured according to the **principle of least privilege**, meaning that each component is granted only the permissions required to perform its assigned tasks.

In the Live Auction system, AWS IAM is used to:

* Allow Lambda functions to write logs to Amazon CloudWatch.
* Allow the required Lambda functions to read and write data in Amazon DynamoDB.
* Allow Lambda functions to send messages to Amazon SQS FIFO.
* Allow Lambda functions to process messages delivered from SQS FIFO.
* Allow Lambda functions to send real-time updates through API Gateway WebSocket.
* Allow the Post Confirmation Lambda to perform operations after an account is confirmed.
* Limit the resources that each Lambda function can access.
* Establish trust relationships between IAM Roles and the services that use them.
* Control access between components in the system.

### Authorization mechanisms of system components

| Component | Authorization mechanism |
| --- | --- |
| **Business Logic Lambda** | Uses an IAM Role to write logs to CloudWatch and access the required DynamoDB tables. |
| **Bid Processing Lambda** | Uses an IAM Role to process messages from SQS FIFO, update DynamoDB, and send results through the WebSocket API. |
| **WebSocket Lambda** | Uses an IAM Role to store, retrieve, or remove WebSocket connection data in DynamoDB and write logs to CloudWatch. |
| **Cognito Post Confirmation Lambda** | Uses an IAM Role to write logs and perform operations after a user confirms an account. |
| **API Gateway** | Invokes Lambda through Lambda Permissions configured for the corresponding API, route, or stage. |
| **CloudFront and Amazon S3** | CloudFront accesses frontend content stored in S3 through Origin Access Control and an S3 Bucket Policy. |

IAM Roles, IAM Policies, and Lambda Permissions are declared using Terraform. Therefore, the authorization configuration can be managed consistently and reused in subsequent deployments.

## Verifying IAM Roles on the AWS Management Console

### Step 1: Access the IAM service

Sign in to the **AWS Management Console**.

Enter the following value in the search bar:

```text
IAM
```

Select **IAM — Identity and Access Management**.

### Step 2: Open the IAM Role list

From the navigation menu, select:

```text
Access management → Roles
```

The Roles page displays the IAM Roles available in the AWS account.

### Step 3: Find the project IAM Roles

Enter the project resource prefix in the search box:

```text
la-
```

Verify the IAM Roles created by Terraform for the Lambda functions and other related components.

The following information should be verified:

* Whether the Role names follow the project naming convention.
* The trusted entity of each Role.
* The creation time of each Role.
* Whether each Role is assigned to the correct service.
* Whether all required Lambda Roles have been created.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/iam-role-list.png"
    title="Figure 5.4.1.1: IAM Roles of the Live Auction system"
    width="100%"
>}}

### Step 4: Verify Permission Policies

Select an IAM Role assigned to a Lambda function.

In the **Permissions** tab, verify:

* The Permission Policies attached to the Role.
* Permissions to write logs to CloudWatch.
* Permissions to access DynamoDB.
* Permissions to access SQS FIFO if the Lambda function processes queue messages.
* Permissions to send data through the WebSocket API if the Lambda function performs real-time updates.
* The resource scope allowed by each Policy.

Broad permissions such as `AdministratorAccess` should not be assigned to Lambda functions. Each Policy should be limited to the resources and operations required by the corresponding function.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/iam-role-permissions.png"
    title="Figure 5.4.1.2: Permission Policies attached to a Lambda IAM Role"
    width="100%"
>}}

### Step 5: Verify the Trust Relationship

On the IAM Role details page, open:

```text
Trust relationships
```

For a Lambda execution role, the Trusted entities configuration must allow:

```text
lambda.amazonaws.com
```

The Trust Relationship determines which service is allowed to assume the IAM Role. This configuration prevents unrelated services from using the Lambda Role.

The **View policy document** option can be used to inspect the Trust Policy. Because the resource is managed by Terraform, its configuration should not be modified directly from the AWS Console.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/iam-role-trust-relationship.png"
    title="Figure 5.4.1.3: Trust Relationship of a Lambda execution role"
    width="100%"
>}}

## Amazon Cognito

### Role of Amazon Cognito

Amazon Cognito is used to manage and authenticate accounts for both the **User Frontend** and the **Admin Frontend**.

Amazon Cognito is responsible for:

* User registration.
* Sign-in and sign-out.
* Account verification.
* Password management.
* Issuing tokens after successful authentication.
* Storing basic account attributes.
* Managing the `user` and `admin` authorization groups.
* Providing authentication information for API authorization.
* Invoking the Post Confirmation Lambda after a user confirms an account.

### Cognito User Pool

The Cognito User Pool is the account directory of the system.

The User Pool stores and manages information such as:

* Username or email.
* Passwords securely managed by Cognito.
* Account confirmation status.
* Account activation status.
* User attributes.
* Password policy.
* Registration and sign-in processes.
* Account authorization groups.

The system uses a shared User Pool for both User and Admin accounts. Access permissions are distinguished through two Cognito Groups:

```text
user
admin
```

### Cognito App Client

The App Client allows the frontend applications to connect to the Cognito User Pool for registration and authentication.

After successful authentication, Cognito can return:

* **ID Token:** Contains identity information about the account.
* **Access Token:** Used to demonstrate access permission.
* **Refresh Token:** Used to request new tokens when the current tokens expire.

The frontend sends a token in the `Authorization` header when calling the REST API. An API Gateway Authorizer or Lambda Authorizer validates the token before forwarding the request to a Lambda function.

For administrative functions, the system also verifies that the account belongs to the `admin` group before allowing the operation.

### Post Confirmation Lambda

The system configures a Post Confirmation Lambda trigger for the Cognito User Pool.

After a user successfully confirms an account, Cognito invokes this Lambda function to perform the required post-confirmation operations, such as initializing related account data in the system.

This configuration includes:

* A Lambda function for processing the Post Confirmation event.
* An IAM Role assigned to the Lambda function.
* A Lambda Permission that allows Cognito to invoke the function.
* A Lambda Trigger connected to the Cognito User Pool.
* A CloudWatch Log Group for storing execution logs.

## Account authentication flow

The general authentication flow of the system is performed as follows:

1. A User or Admin enters sign-in information on the corresponding frontend.
2. The frontend sends an authentication request to Amazon Cognito.
3. Cognito validates the account information in the User Pool.
4. If the information is valid, Cognito returns tokens to the frontend.
5. The frontend includes the token in subsequent API requests.
6. An API Gateway Authorizer or Lambda Authorizer validates the token.
7. The backend verifies the `user` or `admin` group when processing operations that require authorization.
8. The request is processed only when the account is authenticated and has the required permission.
9. Administrative APIs reject requests from accounts that do not belong to the `admin` group.

## Verifying the Cognito User Pool on the AWS Management Console

### Step 1: Access Amazon Cognito

Enter the following value in the AWS Management Console search bar:

```text
Cognito
```

Select **Amazon Cognito**.

### Step 2: Open the User Pool list

In the Amazon Cognito interface, select:

```text
User pools
```

Verify the User Pool created by Terraform for the Live Auction system.

The following information should be verified:

* User Pool name.
* User Pool status.
* User Pool ID.
* AWS Region.
* Last updated time.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-user-pool.png"
    title="Figure 5.4.1.4: Cognito User Pool of the Live Auction system"
    width="100%"
>}}

### Step 3: Verify the User Pool configuration

Select the User Pool of the system.

On the overview and configuration pages, verify:

* User Pool name and ID.
* AWS Region.
* Sign-in method.
* Attributes used for authentication.
* Password policy.
* Self-registration status.
* Email verification mechanism.
* Multi-Factor Authentication configuration, if enabled.
* Security configuration and token validity periods.

The configuration should not be modified directly from the AWS Console because the resource is managed by Terraform. When a change is required, the team should update the Terraform configuration and redeploy the module.

### Step 4: Verify the App Client

On the User Pool details page, open:

```text
Applications → App clients
```

Verify the App Client used by the frontend applications for registration and sign-in.

The following information should be verified:

* App Client name.
* Client ID.
* Enabled authentication flows.
* Token validity periods.
* Callback URL and sign-out URL, if configured.
* Whether the App Client configuration is suitable for the frontend applications.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-app-client.png"
    title="Figure 5.4.1.5: App Client configured for the Live Auction system"
    width="100%"
>}}

### Step 5: Verify Cognito Groups

In the User Pool, open:

```text
User management → Groups
```

Verify the two authorization groups:

```text
user
admin
```

The `user` group is used for standard user accounts. The `admin` group is used for accounts allowed to access administrative functions.

The following information should be verified:

* Group name.
* Group description.
* Group precedence, if configured.
* Number of accounts in each Group.
* Whether Admin accounts are assigned to the `admin` group.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-groups.png"
    title="Figure 5.4.1.6: The user and admin groups in the Cognito User Pool"
    width="100%"
>}}

### Step 6: Verify the account list

In the User Pool, open:

```text
User management → Users
```

Verify the accounts created in the system.

The following information should be verified:

* Username.
* Email.
* Account confirmation status.
* Account activation status.
* Account creation date.
* Group assigned to the account.

An administrator account must be assigned to the `admin` group before it can use administrative functions.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-users.png"
    title="Figure 5.4.1.7: Accounts managed in Amazon Cognito"
    width="100%"
>}}

### Step 7: Verify the Lambda Trigger

On the User Pool details page, locate the Lambda Trigger configuration. Depending on the current version of the AWS Console, this configuration may be available under:

```text
User pool properties → Lambda triggers
```

or:

```text
Extensions → Lambda triggers
```

Verify that the **Post confirmation** event is connected to the correct Lambda function.

The following information should be verified:

* The trigger type is Post Confirmation.
* The name of the connected Lambda function.
* The AWS Region of the Lambda function.
* The User Pool using the trigger.
* Whether the Lambda function exists and is active.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.1-IAM-Cognito/cognito-post-confirmation-trigger.png"
    title="Figure 5.4.1.8: Post Confirmation Lambda connected to the Cognito User Pool"
    width="100%"
>}}

## Results

After verifying the resources directly through the AWS Management Console, the team confirmed that:

* The Cognito User Pool was successfully created by Terraform.
* The App Client was configured for the frontend applications.
* The `user` and `admin` Cognito Groups were created for authorization.
* User accounts were centrally managed in the Cognito User Pool.
* The Post Confirmation Lambda was connected to the User Pool.
* The required IAM Roles and IAM Policies were created.
* The Lambda execution roles contained a Trust Relationship with `lambda.amazonaws.com`.
* The Lambda functions were assigned permissions appropriate to their responsibilities.
* Access between AWS services was restricted according to the principle of least privilege.
* The Cognito and IAM configurations were ready for integration with the frontend applications, API Gateway, Lambda, and other services.

The testing of account registration, sign-in, User/Admin authorization, and API protection is presented in **Section 5.5 — System Testing**.