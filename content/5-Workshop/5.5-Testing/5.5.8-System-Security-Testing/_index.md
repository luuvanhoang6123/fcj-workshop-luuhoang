---
title: "System Security Testing"
date: 2026-08-03
weight: 8
chapter: false
pre: " <b> 5.5.8. </b> "
---

The objective of security testing is to verify that the system actually rejects invalid or unauthorized behavior, rather than only confirming that mechanisms such as JWT, IAM, CORS, or S3 Block Public Access have been configured.

The testing focuses on:

- JWT authentication for REST APIs.
- Authorization between User and Admin roles.
- Resource-level authorization.
- S3 and CORS protection.
- Lambda IAM permission restrictions.
- WebSocket connection and message authentication.
- Protection of sensitive data in logs.
- Control of information returned in error responses.

Test cases involving modification of resource IDs must verify authorization on the actual object being accessed, because an API that accepts an ID without checking ownership may lead to Broken Object Level Authorization. Verifying JWT signatures and expiration is also important for preventing Broken Authentication.

---

#### Testing Scope

The testing scope includes:

```text
Amazon API Gateway
AWS Lambda
JWT authentication
User/Admin authorization
Auction sessions
Auction items
Bids
Image upload
Amazon S3
CloudFront
CORS
API Gateway WebSocket API
AWS IAM
AWS Secrets Manager
Amazon CloudWatch Logs
AWS CloudTrail
````

Destructive testing must not be performed in production. Test cases that attempt to access resources outside an IAM Policy must be performed using a test function, test role, or development/staging environment.

---

#### Prerequisites

Before testing, ensure that:

* The REST API has been deployed through API Gateway and Lambda.
* Protected APIs use an authorizer or an equivalent authentication mechanism.
* JWTs include a signature and expiration time.
* The system has at least two roles: `USER` and `ADMIN`.
* At least two different User accounts are available.
* Each User owns separate test resources.
* S3 Block Public Access has been configured.
* CORS has a trusted-origin allowlist.
* Lambda execution roles have been created.
* WebSocket `$connect` and message routes have been implemented.
* CloudWatch Logs are enabled.
* The testing environment is isolated from production.
* The tester has permission to read the required configurations and logs.

AWS recommends configuring an authorizer for the WebSocket API `$connect` route. A WebSocket Lambda authorizer is applied only during `$connect`, so the application must still validate identity, permissions, and message structure in subsequent routes.

If a required component has not yet been implemented, the corresponding test case must be marked as `BLOCKED`.

---

#### Test Data

| Data                        | Description                                                    |
| --------------------------- | -------------------------------------------------------------- |
| Protected API               | API requiring authentication, for example `GET /users/me`      |
| Admin API                   | API available only to Admin users                              |
| Valid User Token            | Valid, unexpired JWT belonging to a User                       |
| Valid Admin Token           | Valid, unexpired JWT belonging to an Admin                     |
| Forged Token                | JWT with a modified payload or signed using another secret/key |
| Expired Token               | JWT with an expired `exp` claim                                |
| Unsupported Algorithm Token | Token using an algorithm that is not allowed by the system     |
| User A                      | User who owns Resource A                                       |
| User B                      | User who owns Resource B                                       |
| Resource A                  | Session, item, image, or other data owned by User A            |
| Resource B                  | Session, item, image, or other data owned by User B            |
| Trusted Origin              | Frontend origin included in the CORS allowlist                 |
| Untrusted Origin            | Origin not included in the CORS allowlist                      |
| Private S3 Object URL       | Direct URL to an object in a private bucket                    |
| Allowed AWS Resource        | Bucket, secret, or prefix that Lambda is allowed to access     |
| Restricted AWS Resource     | Resource outside the Lambda IAM Policy                         |
| Valid WebSocket Message     | JSON with valid action, fields, and data types                 |
| Invalid WebSocket Message   | Malformed JSON, missing fields, or invalid action              |
| Oversized WebSocket Message | Message larger than the application's allowed limit            |
| Correlation ID              | ID used to locate the corresponding request in logs            |

Do not use production tokens, accounts, or data.

---

### SEC-01 — API Rejects Requests Without a Token

| Field               | Content                                                                                                                                                                                                                                                           |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-01`                                                                                                                                                                                                                                                          |
| **Test Name**       | Reject request without an access token                                                                                                                                                                                                                            |
| **Objective**       | Verify that a protected API does not allow anonymous access.                                                                                                                                                                                                      |
| **Prerequisites**   | At least one protected API is operational.                                                                                                                                                                                                                        |
| **Test Steps**      | 1. Send a request to the protected API without an `Authorization` header. 2. Send a request with an empty `Authorization` header. 3. Send `Authorization: Basic abc`. 4. Send `Authorization: Bearer` without a token. 5. Check the response and CloudWatch Logs. |
| **Expected Result** | All requests are rejected with `401 Unauthorized`; the business Lambda does not modify data; the response does not contain User information; the error uses a stable code such as `AUTH_TOKEN_REQUIRED`.                                                          |
| **Actual Result**   | Record the endpoint, test scenario, status code, and error code.                                                                                                                                                                                                  |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                     |
| **Evidence**        | Request/response with sensitive data masked and CloudWatch Logs.                                                                                                                                                                                                  |

---

### SEC-02 — Forged Token Is Rejected

| Field               | Content                                                                                                                                                                                                                                                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-02`                                                                                                                                                                                                                                                                                                                                                |
| **Test Name**       | Reject JWT with an invalid signature or unsupported algorithm                                                                                                                                                                                                                                                                                           |
| **Objective**       | Verify that the system validates the JWT signature and accepts only configured algorithms.                                                                                                                                                                                                                                                              |
| **Prerequisites**   | A Valid User Token and a tool for creating test tokens are available.                                                                                                                                                                                                                                                                                   |
| **Test Steps**      | 1. Decode the Valid User Token in the test environment. 2. Modify the `sub` or `role` claim without producing a valid new signature. 3. Send the modified token to the protected API. 4. Create a token signed with a different key/secret and send it. 5. Try a token using `alg: none` or another unsupported algorithm. 6. Check responses and logs. |
| **Expected Result** | All forged tokens are rejected with `401 Unauthorized`; the system does not trust the payload before signature verification; only configured algorithms are accepted; no data is modified.                                                                                                                                                              |
| **Actual Result**   | Record the token type, algorithm, status, and error code without recording the full token.                                                                                                                                                                                                                                                              |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                                           |
| **Evidence**        | Request/response with token values masked and JWT verification logs.                                                                                                                                                                                                                                                                                    |

> Do not include a complete access token in documentation or evidence screenshots.

---

### SEC-03 — Expired Token Is Rejected

| Field               | Content                                                                                                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-03`                                                                                                                                                                                                                        |
| **Test Name**       | Reject an expired JWT                                                                                                                                                                                                           |
| **Objective**       | Verify that the `exp` claim is checked at request time.                                                                                                                                                                         |
| **Prerequisites**   | An Expired Token is available, or a short-lived token can be created in the test environment.                                                                                                                                   |
| **Test Steps**      | 1. Send the token while it is still valid if using a short-lived token. 2. Wait until it expires. 3. Send the same request again. 4. Test a token without the `exp` claim if `exp` is required. 5. Check the response and data. |
| **Expected Result** | A valid token is processed according to the User's permissions; expired tokens or tokens missing required claims are rejected with `401 Unauthorized`; no business operation is performed.                                      |
| **Actual Result**   | Record the masked expiration time, test time, status, and error code.                                                                                                                                                           |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                   |
| **Evidence**        | Response and CloudWatch Logs without full token values.                                                                                                                                                                         |

---

### SEC-04 — User Cannot Access Admin APIs

| Field               | Content                                                                                                                                                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `SEC-04`                                                                                                                                                                                                                             |
| **Test Name**       | Verify Admin API authorization                                                                                                                                                                                                       |
| **Objective**       | Verify that an authenticated User without the Admin role is still denied access.                                                                                                                                                     |
| **Prerequisites**   | Valid User Token, Valid Admin Token, and an Admin API are available.                                                                                                                                                                 |
| **Test Steps**      | 1. Call the Admin API using the Valid User Token. 2. Test the available `GET`, `POST`, `PUT`, `PATCH`, or `DELETE` methods. 3. Call the same functionality using the Valid Admin Token. 4. Check data before and after each request. |
| **Expected Result** | A regular User receives `403 Forbidden`; a valid Admin is processed according to the business rules; the User cannot read or modify Admin data; authorization is enforced server-side.                                               |
| **Actual Result**   | Record the endpoint, method, caller role, status, and data changes.                                                                                                                                                                  |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                        |
| **Evidence**        | API responses, data before/after, and authorization logs.                                                                                                                                                                            |

---

### SEC-05 — Role Cannot Be Changed Through the `role` Field

| Field               | Content                                                                                                                                                                                                                                                                                                                                                         |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-05`                                                                                                                                                                                                                                                                                                                                                        |
| **Test Name**       | Prevent mass assignment and role escalation                                                                                                                                                                                                                                                                                                                     |
| **Objective**       | Verify that a User cannot elevate privileges using request data.                                                                                                                                                                                                                                                                                                |
| **Prerequisites**   | A registration, profile update, or User update API is available.                                                                                                                                                                                                                                                                                                |
| **Test Steps**      | 1. Send a valid request as a regular User. 2. Add `"role": "ADMIN"` to the request body. 3. Try adding `"is_admin": true`, `"status": "ACTIVE"`, or similar privilege-related fields. 4. Try sending a role through a query parameter or custom header. 5. Log in again and verify permissions in the database or profile API. 6. Attempt to call an Admin API. |
| **Expected Result** | The server rejects unauthorized fields with `400/422` or ignores them according to the API contract; the database role remains unchanged; a newly issued JWT does not contain Admin privileges; the User still receives `403` when calling an Admin API.                                                                                                        |
| **Actual Result**   | Record the masked payload, response, role before/after, and Admin API result.                                                                                                                                                                                                                                                                                   |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                                                   |
| **Evidence**        | Request body, API response, and User record before/after.                                                                                                                                                                                                                                                                                                       |

Identity and role must come from a verified JWT and trusted server-side data, not from the request body.

---

### SEC-06 — User Cannot Access Another User's Resource by Changing the ID

| Field               | Content                                                                                                                                                                                                                                                                                                           |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-06`                                                                                                                                                                                                                                                                                                          |
| **Test Name**       | Verify Object-Level Authorization                                                                                                                                                                                                                                                                                 |
| **Objective**       | Verify that User A cannot read, modify, or delete private resources belonging to User B.                                                                                                                                                                                                                          |
| **Prerequisites**   | User A and User B each own separate resources.                                                                                                                                                                                                                                                                    |
| **Test Steps**      | 1. Log in as User A. 2. Perform a valid operation on Resource A. 3. Replace the resource ID with the ID of Resource B. 4. Try `GET`, `PUT/PATCH`, and `DELETE` if available. 5. Modify `userId`, `ownerId`, `itemId`, or `sessionId` values in the path, query, and body. 6. Check Resource B after the requests. |
| **Expected Result** | Operations on Resource A are processed normally; access to Resource B is rejected with `403 Forbidden` or `404 Not Found` according to the contract; Resource B is not read, modified, or deleted; hard-to-guess UUIDs are not treated as an authorization mechanism.                                             |
| **Actual Result**   | Record the resource type, caller, owner, operation, status, and data state.                                                                                                                                                                                                                                       |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                     |
| **Evidence**        | Responses and before/after data for both resources.                                                                                                                                                                                                                                                               |

---

### SEC-07 — S3 Bucket Does Not Allow Public Access

| Field               | Content                                                                                                                                                                                                                                                                                                                                                               |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-07`                                                                                                                                                                                                                                                                                                                                                              |
| **Test Name**       | Block public access to S3                                                                                                                                                                                                                                                                                                                                             |
| **Objective**       | Verify that Internet users cannot directly read or list the bucket.                                                                                                                                                                                                                                                                                                   |
| **Prerequisites**   | The bucket and at least one test object exist.                                                                                                                                                                                                                                                                                                                        |
| **Test Steps**      | 1. Open the direct S3 object URL in an incognito browser. 2. Send an unsigned request to the object. 3. Attempt to access the bucket root or perform unauthenticated `ListBucket`. 4. Check all four Block Public Access settings. 5. Check the bucket policy and object ACL. 6. Access the object through CloudFront or a valid presigned URL if the design permits. |
| **Expected Result** | Direct S3 access is rejected with `403 AccessDenied`; the bucket cannot be listed; Block Public Access is enabled; the policy/ACL does not grant permissions to public principals; CloudFront OAC or a valid presigned URL still works according to the design.                                                                                                       |
| **Actual Result**   | Record the masked bucket identifier, request type, status, and public access settings.                                                                                                                                                                                                                                                                                |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                                                         |
| **Evidence**        | `403` response, Block Public Access settings, bucket policy, and ACL.                                                                                                                                                                                                                                                                                                 |

AWS recommends enabling all four S3 Block Public Access settings and reviewing related bucket and identity policies.

---

### SEC-08 — CORS Allows Only Trusted Origins

| Field               | Content                                                                                                                                                                                                                                                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-08`                                                                                                                                                                                                                                                                                                                                                                        |
| **Test Name**       | Verify the CORS allowlist                                                                                                                                                                                                                                                                                                                                                       |
| **Objective**       | Verify that the browser allows access to the API or S3 only from configured frontend origins.                                                                                                                                                                                                                                                                                   |
| **Prerequisites**   | Trusted Origin, Untrusted Origin, and CORS configuration are available.                                                                                                                                                                                                                                                                                                         |
| **Test Steps**      | 1. Send an `OPTIONS` preflight request from the Trusted Origin. 2. Check allow-origin, allow-methods, allow-headers, and credential-related headers. 3. Send the actual request from the Trusted Origin. 4. Repeat with the Untrusted Origin. 5. Test a similar-looking origin such as a fake subdomain or domain with an extra suffix. 6. Verify the result in a real browser. |
| **Expected Result** | Trusted Origin receives the correct CORS headers; Untrusted Origin does not receive `Access-Control-Allow-Origin`; the server does not reflect arbitrary `Origin` values; wildcard origins are not used with credentialed requests; only required methods and headers are allowed.                                                                                              |
| **Actual Result**   | Record the origin, requested method, CORS headers, and browser result.                                                                                                                                                                                                                                                                                                          |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                                                                   |
| **Evidence**        | Preflight request/response and browser console.                                                                                                                                                                                                                                                                                                                                 |

> CORS is not an authentication or authorization mechanism. Requests outside the browser must still be controlled using JWTs, presigned URLs, IAM, or resource policies.

AWS describes CORS as a browser mechanism for allowing applications from one domain to interact with resources on another domain; it does not replace access control.

---

### SEC-09 — Lambda Is Denied Access Outside Its IAM Policy

| Field               | Content                                                                                                                                                                                                                                                                                                                                |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-09`                                                                                                                                                                                                                                                                                                                               |
| **Test Name**       | Verify least-privilege IAM for Lambda                                                                                                                                                                                                                                                                                                  |
| **Objective**       | Verify that Lambda can access only the resources and actions explicitly granted to it.                                                                                                                                                                                                                                                 |
| **Prerequisites**   | A test Lambda role, an Allowed AWS Resource, and a Restricted AWS Resource are available.                                                                                                                                                                                                                                              |
| **Test Steps**      | 1. Make the test Lambda perform an allowed action on the Allowed Resource. 2. Confirm success. 3. Perform the same action on the Restricted Resource. 4. Attempt a disallowed action on the Allowed Resource. 5. Check the execution role, permission boundary, resource policy, and CloudTrail. 6. Check resource data after testing. |
| **Expected Result** | The permitted action on the correct resource succeeds; disallowed actions or resources are rejected with `AccessDenied`; overly broad permissions such as `s3:*`, `secretsmanager:*`, or resource `*` are not used unless necessary; Lambda handles permission errors safely.                                                          |
| **Actual Result**   | Record the role, action, masked resource, decision, and AWS error code.                                                                                                                                                                                                                                                                |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                          |
| **Evidence**        | IAM policy, Lambda logs, CloudTrail event, and resource state.                                                                                                                                                                                                                                                                         |

> Do not modify a production Lambda Function to create unauthorized behavior. Use a controlled test function or test role.

---

### SEC-10 — WebSocket Connection and Messages Are Authenticated and Validated

| Field               | Content                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-10`                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Test Name**       | Verify WebSocket connection and message security                                                                                                                                                                                                                                                                                                                                                                                         |
| **Objective**       | Verify that only valid clients can connect and invalid messages are not processed.                                                                                                                                                                                                                                                                                                                                                       |
| **Prerequisites**   | WebSocket API, `$connect` authorization, and message handlers have been implemented.                                                                                                                                                                                                                                                                                                                                                     |
| **Test Steps**      | 1. Connect to WebSocket without credentials. 2. Connect using a forged or expired token. 3. Connect using a Valid User Token. 4. Send valid JSON. 5. Send a non-JSON string. 6. Send JSON without `action` or another required field. 7. Send an unsupported action. 8. Send incorrect data types, extra fields, or an oversized message. 9. Attempt to send unauthorized `userId`, `role`, or `itemId` values. 10. Check data and logs. |
| **Expected Result** | Unauthenticated connections are rejected; valid connections succeed; identity is derived from the authentication result; invalid messages return controlled errors or close the connection according to the contract; no data is changed; the client cannot impersonate another User using message fields.                                                                                                                               |
| **Actual Result**   | Record the connection/message type, response event, close code, and processing result.                                                                                                                                                                                                                                                                                                                                                   |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                                                                                                                            |
| **Evidence**        | WebSocket client output, API Gateway access logs, and Lambda logs.                                                                                                                                                                                                                                                                                                                                                                       |

Routes that send data using the API Gateway Management API require correctly scoped `execute-api:ManageConnections` permission.

---

### SEC-11 — Logs Do Not Contain Sensitive Data

| Field               | Content                                                                                                                                                                                                                                                                                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-11`                                                                                                                                                                                                                                                                                                                                                    |
| **Test Name**       | Verify that secrets are not leaked into logs                                                                                                                                                                                                                                                                                                                |
| **Objective**       | Verify that logs do not contain passwords, tokens, secret keys, or presigned URL signatures.                                                                                                                                                                                                                                                                |
| **Prerequisites**   | CloudWatch Logs are enabled and the tester has permission to read them.                                                                                                                                                                                                                                                                                     |
| **Test Steps**      | 1. Perform registration, login, and token refresh. 2. Call a protected API. 3. Generate a presigned upload if available. 4. Trigger controlled authentication, S3, and database errors. 5. Search logs for sensitive strings. 6. Check API Gateway access logs, Lambda logs, and application logs. 7. Check log retention and log group access permissions. |
| **Expected Result** | No password, access token, refresh token, cookie, AWS access key, secret key, database password, JWT secret, or complete presigned URL is present; only necessary identifiers are logged; sensitive values are masked; log read permissions are restricted.                                                                                                 |
| **Actual Result**   | Record the log group, time range, search keywords, and number of matches without copying secrets into the report.                                                                                                                                                                                                                                           |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                                               |
| **Evidence**        | Masked log queries and log group configuration.                                                                                                                                                                                                                                                                                                             |

At minimum, search for:

```text
password
passwd
Authorization
Bearer
access_token
refresh_token
id_token
secret
secret_key
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
X-Amz-Signature
X-Amz-Credential
Cookie
Set-Cookie
```

Do not include real secret values in search queries or test documentation.

---

### SEC-12 — Error Messages Do Not Reveal Internal System Structure

| Field               | Content                                                                                                                                                                                                                                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `SEC-12`                                                                                                                                                                                                                                                                                                                  |
| **Test Name**       | Verify information disclosure in error responses                                                                                                                                                                                                                                                                          |
| **Objective**       | Verify that client-facing errors do not expose internal implementation details.                                                                                                                                                                                                                                           |
| **Prerequisites**   | Validation, authentication, authorization, not-found, and controlled internal errors can be generated in the test environment.                                                                                                                                                                                            |
| **Test Steps**      | 1. Send malformed JSON. 2. Omit a required field. 3. Send a non-existent ID. 4. Send an invalid token. 5. Trigger a controlled database/S3 error. 6. Call a non-existent route. 7. Check the response body, headers, and logs.                                                                                            |
| **Expected Result** | The client receives only the status, safe error code, safe message, and correlation ID; the response does not include stack traces, file paths, table names, SQL, database hosts, internal bucket names, Lambda ARNs, secret names, source code, or dependency versions; technical details remain only in protected logs. |
| **Actual Result**   | Record the error type, status, error code, and fields returned.                                                                                                                                                                                                                                                           |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                             |
| **Evidence**        | Masked error responses and corresponding logs identified using the correlation ID.                                                                                                                                                                                                                                        |

A safe error response can look like:

```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred.",
    "requestId": "req-123456"
  }
}
```

The API must not return:

```json
{
  "error": "OperationalError",
  "sql": "SELECT * FROM users...",
  "databaseHost": "auction-db.xxxx.ap-southeast-1.rds.amazonaws.com",
  "file": "/var/task/app/infrastructure/database.py",
  "stackTrace": "..."
}
```

---

### Security Testing Matrix

| ID       | Test Content               | Main Component           | Expected Result                 |
| -------- | -------------------------- | ------------------------ | ------------------------------- |
| `SEC-01` | Request without token      | API Gateway/Lambda       | `401 Unauthorized`              |
| `SEC-02` | Forged token               | JWT verifier             | `401 Unauthorized`              |
| `SEC-03` | Expired token              | JWT verifier             | `401 Unauthorized`              |
| `SEC-04` | User calls Admin API       | Authorization guard      | `403 Forbidden`                 |
| `SEC-05` | Modify role in request     | Validation/authorization | Role remains unchanged          |
| `SEC-06` | Modify resource ID         | Object authorization     | `403` or `404`                  |
| `SEC-07` | Public S3 access           | S3 policy/BPA            | `403 AccessDenied`              |
| `SEC-08` | Untrusted origin           | CORS                     | No CORS permission              |
| `SEC-09` | Lambda exceeds IAM Policy  | IAM                      | `AccessDenied`                  |
| `SEC-10` | Invalid WebSocket          | WebSocket API/Lambda     | Connection/message rejected     |
| `SEC-11` | Sensitive data in logs     | CloudWatch/API Gateway   | No secrets found                |
| `SEC-12` | Internal details in errors | Error handler            | No internal structure disclosed |

---

### Test Result Summary

| ID       | Test Content                                   | Actual Result | Status     |
| -------- | ---------------------------------------------- | ------------- | ---------- |
| `SEC-01` | API rejects request without token              | Not tested    | Not tested |
| `SEC-02` | Forged token is rejected                       | Not tested    | Not tested |
| `SEC-03` | Expired token is rejected                      | Not tested    | Not tested |
| `SEC-04` | User cannot access Admin API                   | Not tested    | Not tested |
| `SEC-05` | Role cannot be changed through request         | Not tested    | Not tested |
| `SEC-06` | Cannot access another User's resource          | Not tested    | Not tested |
| `SEC-07` | S3 does not allow public access                | Not tested    | Not tested |
| `SEC-08` | CORS allows only trusted origins               | Not tested    | Not tested |
| `SEC-09` | Lambda is restricted by IAM Policy             | Not tested    | Not tested |
| `SEC-10` | WebSocket authenticates and validates messages | Not tested    | Not tested |
| `SEC-11` | Logs do not contain sensitive data             | Not tested    | Not tested |
| `SEC-12` | Errors do not disclose internal information    | Not tested    | Not tested |

---

### HTTP Status Code Guidelines

| Scenario                             | Expected Status                                                  |
| ------------------------------------ | ---------------------------------------------------------------- |
| Missing access token                 | `401 Unauthorized`                                               |
| Invalid token signature              | `401 Unauthorized`                                               |
| Expired token                        | `401 Unauthorized`                                               |
| Token missing a required claim       | `401 Unauthorized`                                               |
| Authenticated but lacking permission | `403 Forbidden`                                                  |
| Does not own the resource            | `403 Forbidden` or `404 Not Found` according to the API contract |
| Invalid request body                 | `400 Bad Request` or `422 Unprocessable Entity`                  |
| Resource does not exist              | `404 Not Found`                                                  |
| Request too large                    | `413 Payload Too Large`                                          |
| Unsupported WebSocket message/action | Error event or close code according to the WebSocket contract    |
| Internal error                       | `500 Internal Server Error` with a safe message                  |

Do not use `500 Internal Server Error` for every authentication or validation error.

---

### JWT Verification Requirements

The JWT verifier must validate at minimum:

```text
signature
allowed algorithm
expiration
required claims
subject/user ID
issuer, if used by the system
audience, if used by the system
token type, if access and refresh tokens are differentiated
```

The following values from the request body must not be trusted:

```text
userId
ownerId
email
role
isAdmin
status
```

Access tokens and refresh tokens must not be used interchangeably.

---

### Authorization Verification Requirements

Every API that accesses a resource by ID must verify:

* The User is authenticated.
* The User is still in `ACTIVE` status.
* The User has the appropriate role.
* The User has permission to perform the requested action.
* The User owns the resource or has been granted access to it.
* The resource belongs to the correct related session/item.
* Authorization is revalidated server-side for every request.
* Authorization does not rely on hidden frontend buttons.
* Authorization does not rely on hard-to-guess UUIDs.

---

### WebSocket Verification Requirements

WebSocket security must verify:

* Authentication at `$connect`.
* Invalid or expired tokens are rejected.
* The connection is associated with the verified identity.
* Users cannot declare their own `userId` or `role`.
* Users can join only authorized rooms.
* Every message is valid JSON.
* `action` belongs to an allowlist.
* Required fields exist and use the correct data types.
* Message size limits are enforced.
* Message content is not executed as code or commands.
* Rate limiting or throttling is configured if required.
* Connection ID is not treated as the sole proof of identity.
* Invalid messages do not cause Lambda to crash or return stack traces.

---

### Log Verification Requirements

Logs should contain:

```text
requestId
correlationId
verified userId
action
resourceId
route
HTTP method
status code
authorization decision
AWS error code
Lambda request ID
processing duration
```

Logs must not contain:

```text
password
password hash
access token
refresh token
ID token
session cookie
JWT secret
AWS access key
AWS secret key
database password
Secrets Manager secret value
full presigned URL
X-Amz-Signature
full Authorization header
Base64 image data
```

If a token must be identified during an investigation, log only a non-reversible fingerprint/hash or a masked portion, never the complete token.