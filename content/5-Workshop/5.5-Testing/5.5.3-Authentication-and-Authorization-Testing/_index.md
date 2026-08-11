---
title: "Authentication and Authorization Testing"
date: 2026-08-03
weight: 3
chapter: false
pre: " <b> 5.5.3. </b> "
---

### Authentication and Authorization Testing

#### Test Objectives

This section verifies the authentication and authorization capabilities of the Live Auction system at the system level, including:

- User registration, confirmation, and login using Amazon Cognito.
- Synchronization of user information from Cognito to Amazon DynamoDB.
- Operation of the Lambda Post Confirmation Trigger.
- JWT validation using API Gateway Authorizer.
- Authorization between `User` and `Admin` accounts.
- Use of Cognito-verified claims such as `sub` and `cognito:groups`.
- Prevention of identity or privilege spoofing through the request body.
- Verification that Lambda only accesses AWS resources within the permissions granted by its IAM Policy.

The test cases are numbered from `AUTH-01` to `AUTH-13`. Post Confirmation and retry handling are combined in `AUTH-03`; the use of verified claims is tested in `AUTH-10`; and the behavior of not trusting `userId` and `role` values from the request body is tested in `AUTH-13`.

#### General Test Prerequisites

Before performing the test cases, the system must meet the following conditions:

- The Amazon Cognito User Pool has been deployed.
- The User Pool App Client has been configured for the frontend.
- The Post Confirmation Lambda has been linked to the Cognito User Pool.
- The REST API is protected by API Gateway Authorizer.
- The Lambda Functions have appropriate IAM Execution Roles assigned.
- DynamoDB contains a table for storing user information.
- The User Frontend and Admin Frontend are operational.
- At least one standard API is available for Users.
- At least one administrative API is available only for Admins.
- CloudWatch Logs are enabled for the related Lambda Functions.
- Separate User and Admin accounts are available for testing.

#### Test Data

| Data | Description |
| --- | --- |
| New User | Email address that does not yet exist in the Cognito User Pool |
| Confirmed User | Account with status `CONFIRMED` |
| Unconfirmed User | Account that has not completed the confirmation step |
| Admin | Account belonging to the Cognito Group `Admin` |
| Correct password | Valid password for the test account |
| Incorrect password | Password that does not match the account |
| Valid token | Unexpired JWT issued by the correct Cognito User Pool |
| Invalid token | Token with modified content, invalid signature, or incorrect format |
| Expired token | JWT with an `exp` value earlier than the current time |
| Spoofed User ID | ID of another user sent in the request body |
| Spoofed role | Value `Admin` sent by a User in the request body |

Do not include real email addresses, passwords, Access Tokens, ID Tokens, or Refresh Tokens in the test documentation.

---

#### AUTH-01 — Successful User Registration

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-01` |
| **Test Name** | Successful User account registration |
| **Objective** | Verify that a new user can register an account through the User Frontend and Amazon Cognito. |
| **Prerequisites** | The test email does not exist in the Cognito User Pool; the User Frontend can connect to Cognito. |
| **Test Steps** | 1. Open the registration page. 2. Enter a valid email and password. 3. Enter the required information. 4. Click the registration button. 5. Check the account status in the Cognito User Pool. |
| **Input Data** | A new test email, a password that satisfies the Password Policy, and valid profile information. |
| **Expected Result** | The registration request succeeds; the account is created in Cognito; the interface asks the user to enter a confirmation code; the account cannot log in before confirmation. |
| **Actual Result** | To be completed after testing. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | Screenshot of the registration interface and the account in the Cognito User Pool with sensitive information masked. |

---

#### AUTH-02 — Successful Account Confirmation

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-02` |
| **Test Name** | Confirm account using a valid confirmation code |
| **Objective** | Verify that a newly registered account can be confirmed using a valid confirmation code. |
| **Prerequisites** | The account has been registered but not yet confirmed; the user has received the confirmation code. |
| **Test Steps** | 1. Open the account confirmation page. 2. Enter the test email. 3. Enter a valid confirmation code. 4. Submit the confirmation request. 5. Check the account status in the Cognito User Pool. |
| **Input Data** | Test email and a valid, unexpired confirmation code. |
| **Expected Result** | Cognito confirms the account successfully; the account status changes to `CONFIRMED`; the user can proceed to the login step. |
| **Actual Result** | To be completed after testing. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | Screenshot of the confirmation interface and the `CONFIRMED` status in the Cognito User Pool. |

---

#### AUTH-03 — Post Confirmation Lambda Works and Does Not Create Duplicate Data

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-03` |
| **Test Name** | Trigger Post Confirmation and verify idempotency |
| **Objective** | Verify that Cognito invokes the Post Confirmation Lambda after account confirmation and that repeated processing does not create multiple records for the same user. |
| **Prerequisites** | The Post Confirmation Lambda is configured for the Cognito User Pool; the Lambda has permission to write to the Users table in DynamoDB. |
| **Test Steps** | 1. Register and confirm a new User. 2. Check the CloudWatch Logs of the Post Confirmation Lambda. 3. Check the user record in DynamoDB. 4. Reprocess the same event in the test environment or rerun the processing mechanism using the same `sub`. 5. Check the number of records after the second execution. |
| **Input Data** | Post Confirmation event containing the same Cognito `sub`. |
| **Expected Result** | The Lambda is invoked after confirmation; DynamoDB contains exactly one record corresponding to the Cognito `sub`; repeated processing does not create a second record or corrupt existing data. |
| **Actual Result** | Record the number of Lambda invocations and the number of observed records. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | Lambda CloudWatch Logs and the DynamoDB record with personal information masked. |

> Post Confirmation may be retried by AWS when an error occurs. Therefore, the Lambda should perform idempotent operations based on the Cognito `sub`, not on the email address.

---

#### AUTH-04 — Login with Valid Credentials

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-04` |
| **Test Name** | Login using a confirmed account |
| **Objective** | Verify that a User can log in using valid credentials. |
| **Prerequisites** | The account has status `CONFIRMED` and has not been disabled. |
| **Test Steps** | 1. Open the login page. 2. Enter the correct email and password. 3. Click the login button. 4. Observe the result. 5. Call a standard API after login. |
| **Input Data** | Valid email and password. |
| **Expected Result** | Cognito authenticates successfully; the frontend redirects the user into the system; a valid token is used to call the API; the API returns HTTP `200`. |
| **Actual Result** | To be completed after testing. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | Screenshot of the interface after login and an HTTP `200` response without exposing the token. |

---

#### AUTH-05 — Login with an Incorrect Password

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-05` |
| **Test Name** | Reject login when the password is incorrect |
| **Objective** | Verify that Cognito does not issue a token when the user enters an incorrect password. |
| **Prerequisites** | The User account has been confirmed. |
| **Test Steps** | 1. Open the login page. 2. Enter the correct email. 3. Enter an incorrect password. 4. Submit the login request. 5. Observe the response. |
| **Input Data** | Valid email and incorrect password. |
| **Expected Result** | Login is rejected; no token is issued; the frontend displays an appropriate message without exposing internal information. Depending on the frontend or authentication API contract, the response is normalized to HTTP `400` or `401`. |
| **Actual Result** | Record the HTTP status and observed message. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | Screenshot of the failed login message or the response with sensitive data masked. |

---

#### AUTH-06 — Login with an Unconfirmed Account

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-06` |
| **Test Name** | Reject an unconfirmed account |
| **Objective** | Verify that an unconfirmed account cannot log in to the system. |
| **Prerequisites** | The account has been registered but has not entered the confirmation code. |
| **Test Steps** | 1. Open the login page. 2. Enter the credentials of the unconfirmed account. 3. Submit the login request. 4. Observe the response. |
| **Input Data** | Correct email and password for the unconfirmed account. |
| **Expected Result** | Cognito rejects authentication; no token is issued; the interface informs the user that the account must be confirmed. |
| **Actual Result** | Record the HTTP status or observed message. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | Account status in Cognito and a screenshot of the frontend notification. |

---

#### AUTH-07 — Call API Without a Token

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-07` |
| **Test Name** | Reject request without an Authorization token |
| **Objective** | Verify that API Gateway Authorizer protects APIs that require authentication. |
| **Prerequisites** | The test endpoint is connected to an Authorizer. |
| **Test Steps** | 1. Prepare a request to the protected endpoint. 2. Do not include the `Authorization` header. 3. Send the request. 4. Record the status and response body. |
| **Input Data** | Request without a Bearer token. |
| **Expected Result** | API Gateway rejects the request and returns HTTP `401`; the business Lambda is not executed; no data is modified. |
| **Actual Result** | Record the HTTP status and observed response. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | HTTP `401` response and CloudWatch Logs or metrics showing that the business Lambda was not executed. |

---

#### AUTH-08 — Call API with an Invalid Token

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-08` |
| **Test Name** | Reject an invalid JWT |
| **Objective** | Verify that the Authorizer validates the token format, signature, issuer, and audience/client. |
| **Prerequisites** | The endpoint is protected by an Authorizer. |
| **Test Steps** | 1. Create a test token with modified content or an invalid signature. 2. Send the token in the `Authorization: Bearer <token>` header. 3. Call the protected endpoint. 4. Record the response. |
| **Input Data** | Forged, modified, or incorrectly issued JWT. |
| **Expected Result** | The Authorizer rejects the token; the API returns HTTP `401`; the business Lambda is not executed; the system does not trust claims contained in the forged token. |
| **Actual Result** | Record the HTTP status and observed message. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | HTTP `401` response without exposing the full token. |

---

#### AUTH-09 — Call API with an Expired Token

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-09` |
| **Test Name** | Reject an expired JWT |
| **Objective** | Verify that the Authorizer checks the `exp` claim before allowing API access. |
| **Prerequisites** | A previously valid token is available but has expired. |
| **Test Steps** | 1. Prepare an expired token. 2. Send a request to the protected API. 3. Observe the response. 4. Check whether the business Lambda is invoked. |
| **Input Data** | Bearer token with an `exp` claim earlier than the current time. |
| **Expected Result** | API Gateway Authorizer rejects the request and returns HTTP `401`; the business Lambda is not executed; data is not modified. |
| **Actual Result** | Record the HTTP status and observed result. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | HTTP `401` response and related logs or metrics with the token masked. |

---

#### AUTH-10 — User Calls a Standard API Using a Verified Identity

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-10` |
| **Test Name** | User accesses a standard API using Cognito claims |
| **Objective** | Verify that a User can call a standard API and that the backend uses the verified `sub` claim from the Authorizer as the current identity. |
| **Prerequisites** | The User is logged in; the token is still valid; the endpoint allows User access. |
| **Test Steps** | 1. Log in with a User account. 2. Send a valid token to a standard API. 3. Observe the response. 4. Check CloudWatch Logs or generated data. 5. Compare the owner ID with the `sub` claim. |
| **Input Data** | Valid User token and a valid business request. |
| **Expected Result** | The API returns HTTP `200`; the backend obtains the identity from the verified `sub` claim; resources are retrieved or created for the correct user; the client is not required to declare a trusted identity itself. |
| **Actual Result** | Record the HTTP status, returned data, and comparison result. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | HTTP response, CloudWatch Logs with the token masked, and the related DynamoDB record. |

---

#### AUTH-11 — User Is Denied Access to an Admin API

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-11` |
| **Test Name** | Reject User access to Admin API |
| **Objective** | Verify that an authenticated account that does not belong to the Admin group cannot use administrative functions. |
| **Prerequisites** | The User is logged in; the User does not belong to the Cognito Group `Admin`; the endpoint is restricted to Admin users. |
| **Test Steps** | 1. Log in with a User account. 2. Send a valid token to the Admin API. 3. Observe the response. 4. Check the target data. |
| **Input Data** | Valid User token and an administratively valid request format. |
| **Expected Result** | The API returns HTTP `403`; the administrative operation is not performed; no data is created, modified, or deleted. |
| **Actual Result** | Record the HTTP status and the data state after the request. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | HTTP `403` response, Cognito Group status, and unchanged DynamoDB data. |

---

#### AUTH-12 — Admin Successfully Calls an Admin API

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-12` |
| **Test Name** | Admin accesses an administrative API |
| **Objective** | Verify that a user belonging to the Cognito Group `Admin` can perform administrative functions. |
| **Prerequisites** | The Admin account is confirmed and belongs to the `Admin` group; the administrative endpoint has been deployed. |
| **Test Steps** | 1. Log in with an Admin account. 2. Send a valid token to the Admin API. 3. Perform a valid administrative operation. 4. Check the response and updated data. |
| **Input Data** | Valid token containing the `cognito:groups` claim with value `Admin`, together with a valid request. |
| **Expected Result** | The Authorizer validates the token successfully; the backend confirms authorization from the `cognito:groups` claim; the API returns HTTP `200` or another appropriate success code; the administrative operation is performed correctly. |
| **Actual Result** | Record the HTTP status and observed data. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | The account's Cognito Group, successful response, CloudWatch Logs, and related DynamoDB records. |

---

#### AUTH-13 — Do Not Trust `userId` or `role` in the Request Body

| Field | Content |
| --- | --- |
| **Test Case ID** | `AUTH-13` |
| **Test Name** | Prevent identity and privilege spoofing through the request body |
| **Objective** | Verify that the backend only uses `sub` and `cognito:groups` from the verified token and does not trust `userId` or `role` values sent by the client. |
| **Prerequisites** | A standard User account exists; the ID of another User is known; an endpoint exists for creating or updating user-owned resources. |
| **Test Steps** | 1. Log in as User A. 2. Send a request containing the `userId` of User B. 3. Check the owner of the created data or the API response. 4. Send another request with `role: Admin` in the body. 5. Attempt to call an Admin API. 6. Check the response and data. |
| **Input Data** | Valid token for User A; `userId` of User B; `role: Admin` in the request body. |
| **Expected Result** | The system ignores or rejects untrusted `userId` and `role` values; User A cannot modify User B's data; Admin privileges are not granted; the Admin API returns HTTP `403`; no unauthorized changes occur in DynamoDB. |
| **Actual Result** | Record the HTTP status, stored owner ID, and data state after testing. |
| **Status** | `PASS`, `FAIL`, or `BLOCKED`. |
| **Evidence** | Request body with sensitive information masked, HTTP `403` response, CloudWatch Logs, and DynamoDB records demonstrating the actual owner. |

---

#### Related IAM Verification

In addition to the 13 test cases above, the team verifies the IAM Execution Roles of the Lambda Functions related to authentication:

- The Post Confirmation Lambda can only write to the required DynamoDB table.
- Lambda is not granted administrative access to all DynamoDB resources unless necessary.
- Lambda can only write logs to CloudWatch Logs within the configured scope.
- API Gateway is only permitted to invoke the intended Lambda Function.
- Cognito is only permitted to invoke the intended Post Confirmation Lambda.
- AWS Access Keys or Secret Access Keys are not stored in the source code or Lambda environment variables.

Least-privilege permissions can be verified by attempting to make a Lambda access a table or resource that is not included in its IAM Policy. The expected result is that AWS rejects the operation with an `AccessDenied` error, while the valid flow continues to operate normally.

#### Test Result Summary

| ID | Test Content | Expected HTTP Status | Status |
| --- | --- | --- | --- |
| `AUTH-01` | Successful User registration | `200` or an API-specific success code | Not tested |
| `AUTH-02` | Successful account confirmation | `200` | Not tested |
| `AUTH-03` | Post Confirmation and idempotency | Success, with no duplicate data | Not tested |
| `AUTH-04` | Login with correct credentials | `200` | Not tested |
| `AUTH-05` | Login with incorrect password | `400` or `401` | Not tested |
| `AUTH-06` | Login before account confirmation | `400` or `401` | Not tested |
| `AUTH-07` | API request without token | `401` | Not tested |
| `AUTH-08` | API request with invalid token | `401` | Not tested |
| `AUTH-09` | API request with expired token | `401` | Not tested |
| `AUTH-10` | User calls a standard API | `200` | Not tested |
| `AUTH-11` | User calls an Admin API | `403` | Not tested |
| `AUTH-12` | Admin calls an Admin API | `200` or another appropriate success code | Not tested |
| `AUTH-13` | Spoofed `userId` or `role` | `403` or spoofed data is ignored | Not tested |