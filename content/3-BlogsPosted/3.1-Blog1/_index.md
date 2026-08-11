---
title: "Blog 1 - Live Auction on AWS Serverless"
date: 2026-08-09
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# LIVE AUCTION: BUILDING A REAL-TIME AUCTION PLATFORM ON AWS SERVERLESS

During the project, our team researched and shared the article **“Live Auction: Building a Real-Time Auction Platform on AWS Serverless”** with the AWS Study Group community.

The article presents the development of an online auction platform capable of processing concurrent bid requests, delivering real-time price updates, authenticating users, and deploying infrastructure using Terraform.

## System Problem

One of the most important requirements of an online auction platform is maintaining an accurate latest price when multiple users submit bids within a very short period.

Initially, the system was developed using the following architecture:

```text
React/Vite Frontend
        ↓
FastAPI Backend
        ↓
MySQL Database
```

This architecture was retained for local development and testing. However, after further analysis, the team identified several additional requirements:

* Delivering real-time auction updates.
* Processing multiple concurrent bid requests.
* Preserving the correct bid-processing order.
* Authenticating accounts and enforcing permissions.
* Storing auction item images.
* Automatically updating auction states according to a schedule.
* Monitoring failures and supporting data recovery.
* Deploying infrastructure consistently and repeatedly.

Therefore, the team researched and implemented a serverless AWS architecture for the core auction workflows.

## Architecture Overview

The following diagram illustrates the AWS architecture used for the Live Auction platform.

![AWS Serverless architecture of the Live Auction platform](/images/blog1/live-auction-serverless-architecture.jpg)

The architecture integrates the following technologies and AWS services:

* Amazon S3.
* Amazon CloudFront.
* Amazon Cognito.
* Amazon API Gateway.
* API Gateway WebSocket.
* AWS Lambda.
* Amazon DynamoDB.
* Amazon SQS FIFO.
* Amazon EventBridge.
* Amazon CloudWatch.
* AWS CloudTrail.
* AWS Backup.
* AWS Identity and Access Management.
* AWS CodeBuild and other CI/CD components.
* Terraform.

## Frontend Distribution with Amazon S3 and CloudFront

The User Frontend and Admin Frontend are built from React/Vite source code into static files.

The frontend delivery workflow is as follows:

```text
React/Vite source code
          ↓
Build static assets
          ↓
Amazon S3
          ↓
Amazon CloudFront
          ↓
Users
```

Amazon S3 buckets store HTML, CSS, JavaScript, images, fonts, and other static resources. Amazon CloudFront uses these buckets as origins and distributes the content to users through HTTPS.

The User Frontend and Admin Frontend have separate deployment artifacts, allowing the member and administrator interfaces to be managed independently.

Auction item images are also stored and distributed through separate resources so that media traffic does not compete directly with API requests.

## Authentication with Amazon Cognito

Amazon Cognito User Pool is used to manage:

* Account registration.
* Account confirmation.
* Sign-in.
* Password recovery.
* Token refresh.
* Separation of member and administrator permissions.

After successful authentication, Amazon Cognito issues JWT tokens. The frontend includes these tokens when sending requests to the REST API or establishing a WebSocket connection.

The authentication flow is as follows:

```text
User/Admin Frontend
        ↓
Amazon Cognito
        ↓
Receive JWT Token
        ↓
Send request to API Gateway
        ↓
Validate the token
        ↓
Allow or reject the request
```

A Post Confirmation Lambda trigger completes the initialization of user data after a new account has been confirmed.

## Business Logic with AWS Lambda

Instead of placing all business logic in a single large backend process, the serverless architecture separates responsibilities across specialized Lambda functions.

The main responsibilities include:

* Managing auction sessions and auction rules.
* Managing auction items and image uploads.
* Querying the catalog and bid history.
* Processing administrative commands.
* Managing WebSocket connections.
* Processing bid requests.
* Broadcasting updated results to participants.

Representative Lambda functions include:

```text
session-service
item-service
query-service
admin-command
ws-authorizer
ws-handler
bid-processor
broadcast
```

This structure gives each component a clear responsibility, makes the system easier to test, and allows individual functions to be updated independently.

## Real-Time Auctions with WebSocket

The system uses API Gateway WebSocket to maintain connections between participants and auction rooms.

The connection flow is as follows:

```text
User
  ↓
API Gateway WebSocket
  ↓
WebSocket Authorizer
  ↓
Lambda WebSocket Handler
  ↓
Store Connection ID and room membership
  ↓
Receive real-time auction updates
```

The WebSocket Authorizer validates the user token when a connection is established. The WebSocket Handler then manages the connection ID and the auction room joined by the user.

Using WebSocket allows the system to push new auction prices to connected browsers without requiring the frontend to continuously poll the backend.

## Ordered Bid Processing with Amazon SQS FIFO

A bid request is not written directly from the browser to the current-price record. Instead, the request is submitted to an Amazon SQS FIFO queue.

The bid-processing workflow is as follows:

```text
User submits a bid
        ↓
API Gateway WebSocket
        ↓
Lambda WebSocket Handler
        ↓
Amazon SQS FIFO
        ↓
Lambda Bid Processor
        ↓
Amazon DynamoDB
        ↓
Broadcast result through WebSocket
```

Messages are grouped by auction item to preserve the processing order of bid requests for the same item.

The Bid Processor validates:

* The current auction state.
* The auction start and end times.
* The current price.
* The minimum bid increment.
* The user’s permission to participate.
* Whether the request has already been processed.

DynamoDB conditional updates reduce the risk of two concurrent requests overwriting the same current-price record.

Idempotency records ensure that repeated requests do not change an already processed result, while bid events preserve both accepted and rejected outcomes.

## Data Storage with Amazon DynamoDB

Amazon DynamoDB stores data such as:

* Accounts and user profiles.
* Categories.
* Auction sessions.
* Auction items.
* Auction states.
* Bid history.
* Idempotency records.
* WebSocket connection IDs.
* Auction room memberships.
* Audit events.

DynamoDB works well with AWS Lambda because it does not require a long-running database connection pool and can scale according to the incoming workload.

## Broadcasting Results to Participants

After a bid has been processed, the Broadcast Lambda reads the active connections in the auction room and sends the result through the API Gateway Management API.

Expired or stale connections are removed during this process so that disconnected browsers do not affect the remaining participants.

The updated information may include:

* The latest auction price.
* Whether a bid was accepted or rejected.
* The current leading bidder.
* The remaining auction time.
* The current state of the item and auction session.

The result-broadcasting flow is as follows:

```text
Lambda Bid Processor
        ↓
Update Amazon DynamoDB
        ↓
Lambda Broadcast
        ↓
API Gateway Management API
        ↓
Connected browsers
```

## Auction Lifecycle Automation

Amazon EventBridge supports scheduled auction operations such as:

* Starting an auction session.
* Closing an auction item.
* Moving to the next item.
* Ending an auction session.

The scheduled workflow is as follows:

```text
Amazon EventBridge
        ↓
Lambda Admin Command
        ↓
Update auction state
        ↓
Amazon DynamoDB
        ↓
Broadcast the updated state
```

If a scheduled invocation cannot be processed successfully, a dead-letter queue can preserve the failed event for later inspection instead of silently losing it.

## Auction Item Image Storage

Auction item images are stored in Amazon S3 separately from the frontend assets.

When a user uploads an image, the system can use the following workflow:

```text
Frontend requests an upload
          ↓
Amazon API Gateway
          ↓
AWS Lambda
          ↓
Generate a presigned URL
          ↓
Frontend uploads directly to Amazon S3
```

This approach prevents image data from passing through Lambda, reducing unnecessary API processing and transfer overhead.

## Infrastructure Deployment with Terraform

AWS resources are defined and managed with Terraform according to the Infrastructure as Code model.

The infrastructure is divided into areas such as:

* Identity.
* Data.
* Messaging.
* Compute.
* REST API.
* WebSocket API.
* Edge.
* Security.
* Monitoring.
* Backup.
* CI/CD.

The basic deployment process is:

```text
Terraform configuration
          ↓
terraform init
          ↓
terraform plan
          ↓
Review the deployment plan
          ↓
terraform apply
          ↓
Create resources on AWS
```

Terraform allows the infrastructure configuration to be reviewed, version-controlled, and deployed repeatedly in a consistent manner.

It also helps manage dependencies between resources. For example, Lambda functions require IAM roles, API Gateway requires Lambda integrations, and CloudFront requires Amazon S3 origins.

## CI/CD

AWS CodeBuild supports the build and packaging of Lambda artifacts and frontend assets.

The general workflow is:

```text
Team member updates the source code
              ↓
Repository receives the changes
              ↓
AWS CodeBuild
              ↓
Build and package
              ↓
Create deployment artifacts
              ↓
Deploy the new version
```

AWS CodePipeline and AWS CodeDeploy can support controlled promotion and deployment of new versions.

Applying CI/CD reduces manual deployment work and provides execution history for each build and system update.

## Monitoring and Operational Evidence

Amazon CloudWatch collects:

* Lambda logs.
* API activity metrics.
* SQS queue metrics.
* Processing errors.
* Request latency.
* Bid-related metrics.
* Dead-letter queue activity.

CloudWatch alarms can notify the team when:

* Lambda functions produce multiple errors.
* Request latency increases.
* Messages accumulate in an SQS queue.
* A dead-letter queue receives messages.
* The rejected-bid rate increases unexpectedly.

The architecture also uses:

* AWS CloudTrail to record API activity in the AWS account.
* AWS Config to track resource configurations.
* IAM Access Analyzer to help identify unintended external access.
* AWS Backup to support data backup and recovery.
* Amazon SNS to deliver monitoring notifications.

## System Security

AWS Identity and Access Management controls permissions between AWS services. Each Lambda function receives only the permissions required for its responsibilities.

Examples include:

* Data-management Lambda functions can access only the required DynamoDB tables.
* WebSocket Lambda functions can manage the required WebSocket connections.
* The Bid Processor can receive and delete messages from the SQS FIFO queue.
* The WebSocket Handler can send valid bid requests to the queue.
* CloudFront can retrieve content from its Amazon S3 origin.
* CodeBuild can access only the resources required for the build and deployment process.

The team follows the principle of least privilege and avoids assigning one broadly privileged role to multiple unrelated components.

## Lessons Learned

During the development process, the team learned that serverless architecture does not remove the need for careful system design.

A serverless system still requires:

* Clear responsibility boundaries.
* Appropriate access patterns.
* Controlled retry behavior.
* Idempotent processing.
* Properly scoped IAM permissions.
* Handling of stale connections.
* Management of failed messages.
* Log and metric monitoring.
* Backup and recovery planning.

AWS provides the building blocks for the platform, but system reliability depends on how these services are designed and connected.

## Achieved Results

Through the article and project implementation, the team achieved the following results:

* Designed a serverless architecture for an online auction platform.
* Separated the User Frontend and Admin Frontend.
* Distributed frontend content through Amazon S3 and CloudFront.
* Authenticated users with Amazon Cognito.
* Processed business logic with AWS Lambda.
* Provided REST and WebSocket APIs through Amazon API Gateway.
* Processed bid requests in order with Amazon SQS FIFO.
* Stored auction data and states in Amazon DynamoDB.
* Automated infrastructure deployment with Terraform.
* Monitored system activity with Amazon CloudWatch.
* Improved the team’s understanding of event-driven architecture and real-time data processing.

## Future Improvements

The platform can be further improved by:

* Performing larger-scale load tests.
* Completing the notification system.
* Adding auction analytics.
* Testing multi-Region recovery procedures.
* Optimizing operational costs.
* Improving monitoring alarms and dashboards.
* Evaluating the reliability of the bidding workflow under higher concurrency.

## Article Link

The article was published and approved in the AWS Study Group community:

[**View “Live Auction: Building a Real-Time Auction Platform on AWS Serverless”**](https://www.facebook.com/groups/awsstudygroupfcj/posts/2239889186776041/)