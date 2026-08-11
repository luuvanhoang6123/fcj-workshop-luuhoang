---
title: "Overview and Test Environment"
date: 2026-08-03
weight: 1
chapter: false
pre: " <b> 5.5.1. </b> "
---
### Overview and Test Environment

#### Overview

After completing the deployment of AWS services in section **5.4**, the team proceeded with system testing for the Live Auction platform to verify the operation of the entire system in the AWS environment.

The testing process does not only verify the existence or individual configuration of each resource, but also focuses on validating end-to-end business flows between the frontend, API Gateway, Lambda, Amazon SQS FIFO, DynamoDB, S3, and CloudFront.

Test results are recorded through:

- The User Frontend and Admin Frontend interfaces.
- HTTP responses from the REST API.
- Messages received through WebSocket.
- Data stored in Amazon DynamoDB.
- Message status in Amazon SQS FIFO.
- Files and object versions in Amazon S3.
- Lambda execution logs in Amazon CloudWatch Logs.
- Monitoring metrics in Amazon CloudWatch Metrics.

The objectives of the system testing process include:

- Verify that the main functions of the Live Auction system operate according to requirements.
- Test authentication and authorization between regular users and administrators.
- Verify that the REST API returns the correct data, HTTP status codes, and error messages.
- Test WebSocket connectivity and the ability to update auction data in real time.
- Verify that bid requests are sent to Amazon SQS FIFO and processed in the correct order within the same message group.
- Verify that auction session data, item data, and bid history are stored correctly in Amazon DynamoDB.
- Verify that the User Frontend and Admin Frontend can be accessed through Amazon CloudFront.
- Test the ability to upload, store, and display item images from Amazon S3.
- Test how the system handles invalid input, service errors, and duplicate requests.
- Verify that logs and monitoring metrics provide sufficient information for system monitoring and troubleshooting.
- Evaluate whether the system is ready for use after deployment.



#### Testing Scope

The testing scope includes the following functional areas:

1. User authentication and authorization using Amazon Cognito.
2. Business functions provided through Amazon API Gateway REST.
3. Real-time connection and data exchange through API Gateway WebSocket.
4. Business logic execution by AWS Lambda Functions.
5. The end-to-end bidding flow of the Live Auction system.
6. Ordering and asynchronous processing behavior of Amazon SQS FIFO.
7. Data integrity and consistency in Amazon DynamoDB.
8. Content storage and distribution using Amazon S3 and Amazon CloudFront.
9. Monitoring, logging, and error tracing using Amazon CloudWatch.
10. Error cases, invalid requests, and unauthorized access behavior.
11. Operation of the User Frontend and Admin Frontend in the AWS environment.

Resource configuration checks through the AWS Management Console or Terraform are used only as supporting evidence. System test results must be based on the actual behavior of business flows and the data generated after testing is performed.

#### Tested Architecture

The testing architecture of the Live Auction system consists of two main flow groups.

##### REST API Flow

```text
User Frontend or Admin Frontend
        ↓
Amazon CloudFront
        ↓
Amazon API Gateway REST
        ↓
Amazon Cognito Authorizer
        ↓
AWS Lambda
        ↓
Amazon DynamoDB or Amazon S3
        ↓
Response returned to the Frontend
```

This flow is used for functions such as:

- User registration and login.
- Retrieving the list of auction sessions.
- Viewing auction session and item information.
- Managing auction sessions.
- Managing auction items.
- Uploading and retrieving images.
- Performing administrative functions.



##### WebSocket and Bidding Flow

```text
User Frontend
        ↓
API Gateway WebSocket
        ↓
Lambda la-ws-handler
        ↓
Amazon SQS FIFO
        ↓
Lambda la-bid-processor
        ↓
Amazon DynamoDB
        ↓
Lambda la-broadcast
        ↓
API Gateway WebSocket
        ↓
Data update on the User Frontend
```

This flow is used to:

- Establish a WebSocket connection.
- Join an auction room.
- Send bid requests.
- Process bid requests in order.
- Update the current price and highest bidder.
- Store bid history.
- Send results to users who are following the auction session.



#### Test Environment

The system is tested on AWS infrastructure in the following Region:

```text
Asia Pacific (Singapore) – ap-southeast-1
```

The test environment information is summarized as follows:


| Component                  | Test Environment                              |
| -------------------------- | --------------------------------------------- |
| AWS Region                 | `ap-southeast-1`                              |
| User interface             | User Frontend                                 |
| Administration interface   | Admin Frontend                                |
| Frontend distribution      | Amazon CloudFront                             |
| Frontend and image storage | Amazon S3                                     |
| User authentication        | Amazon Cognito                                |
| REST API                   | Amazon API Gateway REST                       |
| Real-time communication    | API Gateway WebSocket                         |
| Business logic processing  | AWS Lambda                                    |
| Bid queue processing       | Amazon SQS FIFO                               |
| Data storage               | Amazon DynamoDB                               |
| Logging and monitoring     | Amazon CloudWatch                             |
| Test browser               | Browser with JavaScript and WebSocket support |
| Network connection         | Stable Internet connection                    |


CloudFront, API Gateway, and WebSocket API endpoints must be obtained from the actual deployment environment. When included in the report, the team may partially mask the addresses if they do not need to be publicly disclosed.

#### AWS Services Involved in Testing


| Service                     | Role in the Testing Process                                                                |
| --------------------------- | ------------------------------------------------------------------------------------------ |
| **Amazon Cognito**          | Authenticates users, issues tokens, and manages User/Admin permission groups.              |
| **Amazon API Gateway REST** | Receives HTTP requests from the User Frontend and Admin Frontend.                          |
| **API Gateway WebSocket**   | Maintains connections and transmits auction data in real time.                             |
| **AWS Lambda**              | Handles additional authentication, auction business logic, bidding, and data broadcasting. |
| **Amazon DynamoDB**         | Stores user data, auction sessions, items, WebSocket connections, and bid history.         |
| **Amazon SQS FIFO**         | Receives bid requests and preserves their order within the same Message Group.             |
| **Amazon S3**               | Stores the User Frontend, Admin Frontend, and auction item images.                         |
| **Amazon CloudFront**       | Distributes frontend content and static assets from private S3 buckets.                    |
| **Amazon CloudWatch**       | Stores logs, monitors metrics, and supports system error tracing.                          |




#### Interfaces Under Test

The system has two main interfaces.

##### User Frontend

The User Frontend is used to test functions available to regular users, including:

- User registration and login.
- Viewing the list of auction sessions.
- Viewing detailed item information.
- Joining an auction session.
- Establishing a WebSocket connection.
- Sending bid requests.
- Receiving updated prices in real time.
- Viewing item images and status.



##### Admin Frontend

The Admin Frontend is used to test administrative functions, including:

- Logging in with an Admin account.
- Creating and updating auction sessions.
- Managing auction items.
- Managing item images.
- Starting or ending auction sessions according to assigned permissions.
- Accessing REST APIs reserved for administrators.
- Verifying that User accounts cannot access Admin functions.



#### Test Accounts

The testing process uses at least two types of accounts:


| Account Type | Purpose                                                                                 |
| ------------ | --------------------------------------------------------------------------------------- |
| **User**     | Tests login, viewing auction sessions, joining WebSocket connections, and placing bids. |
| **Admin**    | Tests functions for creating, updating, and managing auction sessions or auction items. |


In addition to the two valid accounts, some test cases may use:

- An unconfirmed account.
- An account with an incorrect password.
- An account that does not belong to the Admin group.
- An invalid token.
- An expired token.
- A request without a token.

Test accounts must be created specifically for testing purposes. Personal accounts or accounts containing important data must not be used.