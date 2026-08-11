---
title: "Test Methodology and Test Case Format"
date: 2026-08-03
weight: 2
chapter: false
pre: " <b> 5.5.2. </b> "
---
### Test Methodology and Test Case Format

#### Test Methodology

The team performs system testing using the **black-box testing** approach, combined with verification of data and logs across AWS services.

For each test case, the team provides input data and performs actions through:

- User Frontend.
- Admin Frontend.
- REST API.
- WebSocket connection.
- AWS Management Console.
- API testing tools such as Postman or `curl`.
- Load testing tools, if applicable.

The actual results are then compared with the expected results to determine the status of the test case.

The testing process is performed in the following sequence:

```text
Prepare the test environment and test data
1. Confirm prerequisites
2. Perform the test steps
3. Observe the result on the frontend or API
4. Verify data and logs on AWS
5. Compare with the expected result
6. Record the test status
7. Save test evidence
````

The evaluation is not based only on information displayed on the frontend. Depending on the test case, the team also verifies:

* HTTP status codes and response bodies from the REST API.
* Messages sent and received through WebSocket.
* Records created or updated in Amazon DynamoDB.
* Message status in Amazon SQS FIFO.
* CloudWatch Logs for AWS Lambda.
* CloudWatch Metrics for Lambda, API Gateway, DynamoDB, and SQS.
* Objects or object versions in Amazon S3.
* Content distributed through Amazon CloudFront.
* User status and permission groups in Amazon Cognito.

#### Test Case Classification

Test cases are divided into groups corresponding to the functions and components of the system:

| Prefix     | Test Group                                     | Example       |
| ---------- | ---------------------------------------------- | ------------- |
| `AUTH`     | Authentication and authorization               | `AUTH-01`     |
| `API`      | REST API and auction management business logic | `API-01`      |
| `WS`       | WebSocket and real-time updates                | `WS-01`       |
| `BID`      | End-to-end bidding flow                        | `BID-01`      |
| `FIFO`     | Message ordering and processing with SQS FIFO  | `FIFO-01`     |
| `DB`       | DynamoDB and data integrity                    | `DB-01`       |
| `STORAGE`  | Amazon S3 and CloudFront                       | `STORAGE-01`  |
| `RECOVERY` | Error handling and recovery                    | `RECOVERY-01` |
| `PERF`     | Performance and concurrent load                | `PERF-01`     |
| `SEC`      | System security                                | `SEC-01`      |

When a test case has the status `FAIL`, the team must additionally record:

* The step where the error occurred.
* The time when the error occurred.
* The observed error message.
* HTTP status code if related to the REST API.
* Request code or request ID, if available.
* The related Lambda Function.
* The related CloudWatch Log Group or Log Stream.
* The impact of the error on the system.
* The proposed solution or bug-fix task.

When a test case has the status `BLOCKED`, the team must clearly specify:

* The missing component.
* The dependent function that has not been completed.
* The configuration that has not yet been deployed.
* The access permission that has not been granted.
* The condition that must be completed before retesting.

#### Evidence Collection Guidelines

| Evidence Type             | Usage                                                                         |
| ------------------------- | ----------------------------------------------------------------------------- |
| User Frontend screenshot  | Demonstrates that user-facing functionality works correctly.                  |
| Admin Frontend screenshot | Demonstrates administrative functionality and authorization.                  |
| HTTP request and response | Demonstrates that the REST API returns the correct status and data.           |
| WebSocket message         | Demonstrates that data is sent and received in real time.                     |
| CloudWatch Logs           | Demonstrates that Lambda is invoked and processes business logic.             |
| CloudWatch Metrics        | Demonstrates request counts, errors, latency, or processed message volume.    |
| DynamoDB item             | Demonstrates that data is created or updated correctly.                       |
| SQS Metrics               | Demonstrates that messages are sent, received, and deleted from the Queue.    |
| DLQ content               | Demonstrates that failed messages are moved to the Dead-letter Queue.         |
| S3 object                 | Demonstrates that a file is uploaded and stored in the correct bucket.        |
| CloudFront response       | Demonstrates that the frontend or static content is distributed successfully. |
| Cognito User Pool         | Demonstrates account status or permission group membership.                   |

Each image should have a clear title and description corresponding to the related test case, for example:

```text
Figure 5.5.2.1: Result of test case AUTH-01
```

A single shared image should not be used for multiple test cases if it does not clearly demonstrate the result of each individual case.

#### Test Data Management

To ensure that results can be verified again, the team must prepare test data before performing the tests:

* A confirmed User account.
* An Admin account assigned to the correct permission group.
* A User account without Admin privileges.
* An auction session with the status `SCHEDULED`.
* An auction session with the status `ACTIVE`.
* An auction session with the status `ENDED`.
* An item with a starting price and minimum bid increment.
* A valid auction session ID and item ID.
* A non-existent resource ID.
* Valid and invalid bid amounts.
* Images with valid and invalid formats.
* Files within and exceeding the allowed size limit.
* An active WebSocket connection.
* Duplicate messages for idempotency testing, if this functionality has been implemented.

Test data must be clearly distinguished from real data. After testing is completed, the team should delete or mark the test data if it is no longer required.

#### Test Group Execution Order

The test case groups should be executed in the following dependency order:

1. Verify the environment and endpoints.
2. Test authentication and authorization.
3. Test the REST API.
4. Test data in DynamoDB.
5. Test WebSocket connectivity.
6. Test the end-to-end bidding flow.
7. Test SQS FIFO and concurrent processing.
8. Test S3 and CloudFront.
9. Test error handling and recovery.
10. Test security.
11. Test performance.
12. Consolidate the results.

#### Test Information Security

The team must not expose:

* Test account passwords.
* Access Tokens, ID Tokens, or Refresh Tokens.
* The `Authorization` header.
* AWS Access Key ID.
* AWS Secret Access Key.
* AWS Session Token.
* Cognito Client Secret.
* Cookies or login session information.
* Contents of `.env` files.
* Valid presigned URLs.
* Unnecessary personal data.

If a request or log contains sensitive information, the team must mask or remove that data before including it in the report. Only information necessary to demonstrate that the test case was performed and produced the corresponding result should be retained.