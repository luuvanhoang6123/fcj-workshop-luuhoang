---
title: "AWS Lambda"
date: 2026-07-27
weight: 4
chapter: false
pre: " <b> 5.4.4. </b> "
---

## Overview

The **Live Auction** system uses **AWS Lambda** to execute backend operations without requiring the team to provision or manage servers.

The Lambda functions receive requests from Amazon API Gateway, process auction data, communicate with Amazon DynamoDB, process messages from Amazon SQS FIFO, and send real-time updates to users through API Gateway WebSocket.

The Lambda functions are created and configured using Terraform in the following modules:

```text
infra/03-identity
infra/06-compute
infra/06-compute/stage3-control-plane
```

The source code of each Lambda function is packaged as a `.zip` file before Terraform deploys it to AWS.


## Role of AWS Lambda

AWS Lambda is a serverless compute service that executes code when an event occurs.

In the Live Auction system, AWS Lambda is used to:

* Process accounts after users confirm their registration.
* Process auction session operations.
* Process auction item information.
* Query category, session, and item data.
* Process administrative commands.
* Validate tokens when users establish WebSocket connections.
* Manage the WebSocket connection lifecycle.
* Receive and process bid requests.
* Process messages sequentially from SQS FIFO.
* Update data in DynamoDB.
* Send auction results to WebSocket clients.
* Write execution logs to Amazon CloudWatch.

## Lambda functions of the system

The primary Lambda functions of the system include:

| Lambda Function             | Role                                                                   |
| --------------------------- | ---------------------------------------------------------------------- |
| **la-cognito-post-confirm** | Processes the event generated after a user confirms a Cognito account. |
| **la-session-service**      | Processes operations related to auction sessions.                      |
| **la-item-service**         | Processes auction item operations and media information.               |
| **la-query-service**        | Queries categories, auction sessions, items, and other display data.   |
| **la-admin-command**        | Processes administrative operations and auction lifecycle commands.    |
| **la-ws-authorizer**        | Validates Cognito JWTs when users establish WebSocket connections.     |
| **la-ws-handler**           | Manages WebSocket connections, auction rooms, and bid commands.        |
| **la-bid-processor**        | Processes bid messages from SQS FIFO and updates auction states.       |
| **la-broadcast**            | Sends auction results and state updates to WebSocket clients.          |

The actual names may contain additional prefixes or environment identifiers depending on the Terraform configuration.

## Authentication Lambda

### Cognito Post Confirmation

The `la-cognito-post-confirm` Lambda is invoked by Amazon Cognito after a user successfully confirms an account registration.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 128 MB
Timeout: 10 seconds
```

This Lambda performs the required operations after an account is confirmed.

The Lambda Trigger configuration of this function was presented in **Section 5.4.1 — AWS IAM and Amazon Cognito**.

## REST API Lambda functions

### Session Service

The `la-session-service` Lambda processes operations related to auction sessions.

Its responsibilities can include:

* Creating an auction session.
* Updating session information.
* Retrieving sessions created by the current user.
* Managing session states.
* Verifying session ownership.
* Connecting a session to one or more auction items.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### Item Service

The `la-item-service` Lambda processes auction item operations.

Its primary responsibilities include:

* Adding an item to an auction session.
* Updating item information.
* Managing image information.
* Creating the required information for uploading media to Amazon S3.
* Connecting items to an auction session.
* Validating item input data.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### Query Service

The `la-query-service` Lambda provides data query operations.

The function is used to:

* Retrieve a list of auction sessions.
* Retrieve auction session details.
* Retrieve a list of auction items.
* Retrieve auction item information.
* Query product categories.
* Retrieve data required by the User Frontend and Admin Frontend.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### Admin Command

The `la-admin-command` Lambda processes operations for administrators.

Related functions include:

* Managing user accounts.
* Managing product categories.
* Approving auction sessions.
* Creating additional Admin accounts.
* Processing commands that control auction session states.
* Working with EventBridge Scheduler to manage the auction lifecycle.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 60 seconds
```

## WebSocket Lambda functions

### WebSocket Authorizer

The `la-ws-authorizer` Lambda validates the Cognito JWT when a user requests a connection to the WebSocket API.

The authorization process includes:

1. Receiving the token from the connection request.
2. Validating the token signature and validity.
3. Verifying the Cognito issuer.
4. Retrieving account information from the token.
5. Allowing or rejecting the WebSocket connection.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### WebSocket Handler

The `la-ws-handler` Lambda manages the lifecycle and requests of WebSocket clients.

The function processes:

* The `$connect` event.
* The `$disconnect` event.
* Storing Connection IDs in DynamoDB.
* Removing Connection IDs when users disconnect.
* Managing auction room membership.
* Receiving bid requests.
* Sending valid bid requests to SQS FIFO.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

### Broadcast Lambda

The `la-broadcast` Lambda sends auction results to the WebSocket clients that are following an auction session.

The function performs the following operations:

* Receives results after a bid request has been processed.
* Retrieves the related Connection IDs.
* Sends the updated price through the API Gateway Management API.
* Removes inactive connections when required.
* Delivers real-time updates to participants.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

## Bid Processing Lambda

The `la-bid-processor` Lambda processes bid requests delivered through SQS FIFO.

The processing flow operates as follows:

1. The WebSocket Handler receives a bid request.
2. A valid request is sent to SQS FIFO.
3. SQS delivers the message to the Bid Processor Lambda.
4. The Lambda validates the request data.
5. The Lambda updates the auction state in DynamoDB.
6. The result is sent to the Broadcast Lambda or WebSocket API.
7. Users receive the updated price in real time.

The function is configured with:

```text
Runtime: Python 3.13
Architecture: x86_64
Handler: handler.handler
Memory: 512 MB
Timeout: 30 seconds
```

The Event Source Mapping between SQS FIFO and the Bid Processor is configured with:

```text
State: Enabled
Batch size: 10
Report batch item failures: Enabled
```

`ReportBatchItemFailures` allows the Lambda function to report individual messages that failed instead of processing the entire batch again.

## Lambda deployment packages

The source code of each Lambda function is packaged as a `.zip` file.

The deployment packages can include:

```text
cognito_post_confirm.zip
session_service.zip
item_service.zip
query_service.zip
admin_command.zip
ws_authorizer.zip
ws_handler.zip
bid_processor.zip
broadcast.zip
```

Terraform uses the paths of these packages to upload the function code to AWS Lambda.

When the source code changes, the packages must be rebuilt before running `terraform plan` or `terraform apply`. If a required package does not exist, Terraform may report an error from:

```text
filebase64sha256(...)
```

## Environment variables

The Lambda functions use environment variables to receive configuration values during execution.

Environment variables can include:

* DynamoDB Table names.
* SQS FIFO Queue names.
* Cognito User Pool ID.
* Cognito issuer.
* WebSocket API endpoint.
* Media Bucket name.
* Media CloudFront Domain.
* Deployment environment name.
* Application configuration values.

These values should not be hardcoded in the source code. Terraform provides the required values to the Lambda functions during deployment.

{{% notice warning %}}
Do not capture or publish all environment variable values if they contain internal endpoints, tokens, secrets, or resource identifiers. Only retain the variable names required to describe the configuration and mask their values when necessary.
{{% /notice %}}

## Lambda access permissions

Each Lambda function is assigned an IAM Execution Role appropriate to its responsibilities.

Depending on its function, a Lambda can be granted permission to:

* Write logs to Amazon CloudWatch.
* Read and write specified DynamoDB tables.
* Send messages to SQS FIFO.
* Receive and delete SQS messages through an Event Source Mapping.
* Access Cognito information.
* Send data through the API Gateway WebSocket Management API.
* Access objects in the Item Media Bucket.
* Create or manage EventBridge Schedules.
* Invoke another Lambda function when required.

The IAM Roles and Policies of the Lambda functions were presented in **Section 5.4.1 — AWS IAM and Amazon Cognito**.

## Logging with Amazon CloudWatch

Each Lambda function has a CloudWatch Log Group with the following structure:

```text
/aws/lambda/<function-name>
```

Examples include:

```text
/aws/lambda/la-bid-processor
/aws/lambda/la-ws-handler
/aws/lambda/la-session-service
```

Lambda logs help the team to:

* Confirm whether a function was invoked.
* View the start and end time of an invocation.
* Inspect errors generated during processing.
* Track requests and events.
* Detect IAM permission errors.
* Identify DynamoDB or SQS connection errors.
* Inspect execution duration and memory usage.

Complete logs should not be published if they contain tokens, email addresses, account data, or sensitive bid information.

## Verifying Lambda functions on the AWS Management Console

### Step 1: Access AWS Lambda

Sign in to the **AWS Management Console**.

Enter the following value in the search bar:

```text
Lambda
```

Select **Lambda**.

Ensure that the selected Region is:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

### Step 2: Verify the function list

From the navigation menu, select:

```text
Functions
```

Enter the following prefix in the search box:

```text
la-
```

Verify the Lambda functions of the system.

The following information should be checked:

* Function name.
* Description.
* Package type.
* Runtime.
* Last modified time.
* AWS Region.
* Whether all required functions were deployed.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-function-list.png"
    title="Figure 5.4.4.1: Lambda functions of the Live Auction system"
    width="100%"
>}}

### Step 3: Verify the function configuration

Select a Lambda function, such as:

```text
la-session-service
```

On the function overview page, verify:

* Function ARN.
* Runtime.
* Handler.
* Architecture.
* Last modified time.
* Code package.
* Function state.

Next, open:

```text
Configuration → General configuration
```

Verify:

* Memory.
* Ephemeral storage.
* Timeout.
* Execution Role.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-general-configuration.png"
    title="Figure 5.4.4.2: Runtime, memory, and timeout configuration of a Lambda function"
    width="100%"
>}}

### Step 4: Verify the Bid Processor trigger

Select:

```text
la-bid-processor
```

In the **Function overview** or **Configuration** tab, inspect the SQS Trigger.

The following information should be confirmed:

* The trigger is Amazon SQS.
* The Event Source Mapping is Enabled.
* The queue points to the Bid Command FIFO Queue.
* Batch size is `10`.
* Report batch item failures is enabled.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-bid-processor-sqs-trigger.png"
    title="Figure 5.4.4.3: SQS Trigger of the Bid Processor Lambda"
    width="100%"
>}}

### Step 5: Verify an API Gateway trigger

Select a Lambda function invoked by the REST API, such as:

```text
la-session-service
```

In the **Function overview** or:

```text
Configuration → Triggers
```

Inspect the API Gateway Trigger.

The following information should be verified:

* The connected API Gateway.
* API type.
* Stage.
* Statement ID.
* Permission allowing API Gateway to invoke the Lambda.
* Whether the Lambda is connected to the correct API.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-api-gateway-trigger.png"
    title="Figure 5.4.4.4: API Gateway Trigger of a Lambda function"
    width="100%"
>}}

### Step 6: Verify environment variables

On the Lambda function page, open:

```text
Configuration → Environment variables
```

Verify the names of the environment variables configured for the function.

Only confirm that:

* The required variables exist.
* DynamoDB Table names are correct.
* Queue names are correct.
* Cognito configuration is correct.
* The WebSocket endpoint or Media Bucket matches the environment.

Complete values should not be displayed if they contain information that should not be published.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-environment-variables.png"
    title="Figure 5.4.4.5: Environment variables of a Lambda function"
    width="100%"
>}}

### Step 7: Verify the Execution Role

On the Lambda function page, open:

```text
Configuration → Permissions
```

Inspect:

```text
Execution role
```

The following information should be confirmed:

* An IAM Role is assigned to the Lambda.
* The Role name follows the project naming convention.
* The Resource Summary matches the function's responsibilities.
* `AdministratorAccess` is not assigned.
* The function can access only the required resources.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-execution-role.png"
    title="Figure 5.4.4.6: IAM Execution Role assigned to a Lambda function"
    width="100%"
>}}

### Step 8: Verify CloudWatch Logs

Select a Lambda function that has already been invoked, such as:

```text
la-bid-processor
```

Open:

```text
Monitor → View CloudWatch logs
```

Select the most recent Log Stream and verify:

* Function invocation time.
* The `START` entry.
* The `END` entry.
* The `REPORT` entry.
* Execution duration.
* Memory usage.
* Errors, if present.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-cloudwatch-log.png"
    title="Figure 5.4.4.7: Lambda execution logs on Amazon CloudWatch"
    width="100%"
>}}

### Step 9: Verify Metrics

Open the following tab on the Lambda function page:

```text
Monitor
```

Verify the following Metrics:

* Invocations.
* Duration.
* Error count and success rate.
* Throttles.
* Concurrent executions.
* Async event age, if used.

These Metrics confirm that the function has been invoked and help identify operational issues.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.4-Lambda/lambda-monitoring-metrics.png"
    title="Figure 5.4.4.8: Monitoring Metrics of a Lambda function"
    width="100%"
>}}

## Results

After verifying the resources directly through the AWS Management Console, the team confirmed that:

* The required Lambda functions were deployed by Terraform.
* The functions used Python 3.13 and the `x86_64` architecture.
* Runtime, Handler, Memory, and Timeout were configured according to each function's responsibilities.
* The Cognito Post Confirmation Lambda was connected to the Cognito User Pool.
* The REST API Lambda functions were connected to Amazon API Gateway.
* The WebSocket Authorizer and WebSocket Handler were ready to process real-time connections.
* The Bid Processor Lambda was connected to SQS FIFO through an Event Source Mapping.
* The Event Source Mapping was Enabled.
* Appropriate IAM Execution Roles were assigned to the functions.
* The required environment variables were configured.
* The Lambda functions could write logs to Amazon CloudWatch.
* Metrics and logs could be used for monitoring and troubleshooting.
* The Lambda functions were ready for integration with API Gateway, DynamoDB, SQS FIFO, and the WebSocket API.

The configuration and verification of the REST API are presented in **Section 5.4.5 — Amazon API Gateway**.