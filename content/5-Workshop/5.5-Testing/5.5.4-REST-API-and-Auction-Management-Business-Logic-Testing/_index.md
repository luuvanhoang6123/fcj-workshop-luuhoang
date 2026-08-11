---
title: "REST API and Auction Management Business Logic Testing"
date: 2026-08-03
weight: 4
chapter: false
pre: " <b> 5.5.4. </b> "
---

### REST API and Auction Management Business Logic Testing

#### Test Objectives

This section tests the REST APIs and auction session management business logic implemented through Amazon API Gateway and the Business Logic Lambda.

The testing objectives include:

- Retrieving the list of auction sessions.
- Viewing auction session and item details.
- Creating and updating auction sessions.
- Starting and ending auction sessions.
- Verifying authorization between User and Admin accounts.
- Validating input data.
- Verifying auction session state transition rules.
- Verifying the actual data written to DynamoDB.
- Verifying HTTP status codes and the API JSON structure.
- Verifying that the frontend receives and displays the correct results.

The test cases are numbered from `API-01` to `API-12`.

The team must verify all of the following:

1. API Gateway returns the correct HTTP status.
2. The response body has the correct JSON structure.
3. The Business Logic Lambda performs the correct business operation.
4. Data in DynamoDB is created or updated correctly.
5. The frontend receives and displays the correct result.
6. No data outside the scope of the request is modified.

---

#### Testing Scope

The components under test include:

- User Frontend.
- Admin Frontend.
- Amazon API Gateway REST API.
- API Gateway Authorizer.
- Business Logic Lambda.
- Amazon DynamoDB.
- Amazon CloudWatch Logs.
- Cognito claims such as `sub` and `cognito:groups`.

---

#### General Test Prerequisites

Before executing the test cases, the system must meet the following conditions:

- The REST API has been deployed on API Gateway.
- Routes are connected to the correct Lambda Functions.
- APIs requiring authentication are protected by an Authorizer.
- The Business Logic Lambda has permission to read from or write to the appropriate DynamoDB tables.
- DynamoDB contains tables for auction sessions and items.
- The User Frontend and Admin Frontend can call the API.
- A confirmed User account is available.
- An Admin account belonging to the Cognito Group `Admin` is available.
- Valid Access Tokens are available for both User and Admin accounts.
- CloudWatch Logs are enabled.
- Auction session data is available in the states required for testing.
- The test environment is isolated from real production data.

---

#### Test Data

| Data | Description |
| --- | --- |
| Valid Admin | Account belonging to the Cognito Group `Admin` |
| Valid User | Account that does not belong to the Admin group |
| `SCHEDULED` session | Session that has been created but has not yet started |
| `ACTIVE` session | Session currently in progress |
| `ENDED` session | Session that has ended |
| `CANCELLED` session | Session that has been cancelled, if supported by the system |
| Valid Session ID | ID of an existing auction session |
| Non-existent Session ID | Correctly formatted ID that does not exist in DynamoDB |
| Valid Item ID | ID of an existing item |
| Non-existent Item ID | Correctly formatted ID that does not exist in DynamoDB |
| Valid session creation data | Contains a name, start time, end time, and all required fields |
| Invalid data | Missing fields, incorrect data types, or business rule violations |
| Valid time range | `startTime` is earlier than `endTime` |
| Invalid time range | `startTime` is greater than or equal to `endTime` |

Do not include tokens, passwords, or other sensitive information in the test documentation or evidence.

---

#### HTTP Status Code Conventions

| HTTP Status | Usage |
| --- | --- |
| `200 OK` | Successfully retrieve data or update business data |
| `201 Created` | Successfully create a new auction session |
| `400 Bad Request` | Request contains missing fields, invalid formatting, or input rule violations |
| `401 Unauthorized` | Token is missing or invalid |
| `403 Forbidden` | User is authenticated but does not have permission to perform the operation |
| `404 Not Found` | Auction session or item does not exist |
| `409 Conflict` | The current state does not allow the requested operation or a business conflict occurs |
| `500 Internal Server Error` | An unhandled internal system error occurs |

> The project may use `400` instead of `409` for invalid state transitions. However, the entire API should follow a consistent contract, and the convention must be clearly documented in the API documentation.

---

#### Common JSON Structure to Verify

For successful responses, the API should use a consistent structure, for example:

```json
{
  "data": {},
  "message": "Operation completed successfully",
  "requestId": "example-request-id"
}
````

For failed responses, the API should return a structure that can be handled consistently:

```json
{
  "error": {
    "code": "AUCTION_SESSION_NOT_FOUND",
    "message": "Auction session was not found"
  },
  "requestId": "example-request-id"
}
```

The API must not return the following information to the client:

* Stack traces.
* Internal source code paths.
* AWS credentials.
* Table names or unnecessary infrastructure details.
* Token contents.
* Error details that may expose the internal system structure.

The actual structure may differ from the examples above, but it must remain consistent across endpoints.

---

#### API-01 — Retrieve Auction Session List

| Field               | Content                                                                                                                                                                                                                                                               |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `API-01`                                                                                                                                                                                                                                                              |
| **Test Name**       | Retrieve the list of auction sessions                                                                                                                                                                                                                                 |
| **Objective**       | Verify that the API returns auction session data from DynamoDB and that the frontend displays the data correctly.                                                                                                                                                     |
| **Prerequisites**   | DynamoDB contains multiple auction sessions in different states; the list endpoint has been deployed.                                                                                                                                                                 |
| **Test Steps**      | 1. Record the existing sessions in DynamoDB. 2. Open the auction session list page. 3. Send a request to retrieve the session list. 4. Record the HTTP status and JSON response. 5. Compare the API data with DynamoDB. 6. Verify the list displayed on the frontend. |
| **Input Data**      | Pagination parameters, status filters, or sorting parameters if supported by the API.                                                                                                                                                                                 |
| **Expected Result** | The API returns HTTP `200`; the response contains the session list in the correct structure; data matches DynamoDB; filtering, pagination, and sorting work correctly; the frontend displays the correct name, status, and time of each session.                      |
| **Actual Result**   | Record the number of returned records, HTTP status, response structure, and actual frontend result.                                                                                                                                                                   |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                         |
| **Evidence**        | HTTP request/response, corresponding DynamoDB data, and a screenshot of the frontend list.                                                                                                                                                                            |

---

#### API-02 — View Auction Session or Item Details

| Field               | Content                                                                                                                                                                                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `API-02`                                                                                                                                                                                                                                                    |
| **Test Name**       | View auction session and item details                                                                                                                                                                                                                       |
| **Objective**       | Verify that the API returns the correct details of the requested resource.                                                                                                                                                                                  |
| **Prerequisites**   | Valid auction sessions and items exist in DynamoDB.                                                                                                                                                                                                         |
| **Test Steps**      | 1. Select an existing auction session or item. 2. Send a request to retrieve details using its ID. 3. Record the HTTP status and JSON response. 4. Compare the fields with DynamoDB. 5. Open the detail page on the frontend. 6. Verify the displayed data. |
| **Input Data**      | Valid Session ID or Item ID.                                                                                                                                                                                                                                |
| **Expected Result** | The API returns HTTP `200`; the response ID matches the requested ID; fields such as name, description, status, time, price, and related items match DynamoDB; the frontend displays the correct data.                                                      |
| **Actual Result**   | Record the HTTP status, ID, verified fields, and frontend result.                                                                                                                                                                                           |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                               |
| **Evidence**        | JSON response, DynamoDB record, and screenshot of the detail page.                                                                                                                                                                                          |

---

#### API-03 — Admin Successfully Creates an Auction Session

| Field               | Content                                                                                                                                                                                                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `API-03`                                                                                                                                                                                                                                                                             |
| **Test Name**       | Create an auction session using an Admin account                                                                                                                                                                                                                                     |
| **Objective**       | Verify that an Admin can create a new auction session and that the data is stored correctly.                                                                                                                                                                                         |
| **Prerequisites**   | The Admin is logged in; the token contains `Admin` in the `cognito:groups` claim; the session creation endpoint has been deployed.                                                                                                                                                   |
| **Test Steps**      | 1. Record the data before testing. 2. Log in as Admin. 3. Enter all required session data in the Admin Frontend. 4. Send the session creation request. 5. Verify the HTTP status and response. 6. Locate the new record in DynamoDB. 7. Reopen the session list or detail page.      |
| **Input Data**      | Session name, description, `startTime`, `endTime`, and valid required fields.                                                                                                                                                                                                        |
| **Expected Result** | The API returns HTTP `201`; the response contains a new ID; the session is stored exactly once in DynamoDB; the initial status is `SCHEDULED` or the configured initial status; `createdBy` is derived from the Admin's verified `sub` claim; the frontend displays the new session. |
| **Actual Result**   | Record the HTTP status, generated ID, status, and actual DynamoDB data.                                                                                                                                                                                                              |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                        |
| **Evidence**        | Request with token masked, HTTP `201` response, DynamoDB record, CloudWatch Logs, and Admin Frontend screenshot.                                                                                                                                                                     |

---

#### API-04 — Admin Updates an Auction Session

| Field               | Content                                                                                                                                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `API-04`                                                                                                                                                                                                                                   |
| **Test Name**       | Update auction session information                                                                                                                                                                                                         |
| **Objective**       | Verify that an Admin can update a session while it is in an editable state.                                                                                                                                                                |
| **Prerequisites**   | A session exists in a state that allows editing; the Admin is logged in.                                                                                                                                                                   |
| **Test Steps**      | 1. Record the session data before the update. 2. Modify permitted fields, such as the name or time. 3. Send the update request. 4. Verify the response. 5. Read the record again from DynamoDB. 6. Reload the detail page on the frontend. |
| **Input Data**      | Valid Session ID and valid update fields.                                                                                                                                                                                                  |
| **Expected Result** | The API returns HTTP `200`; only the requested fields are updated; unrelated fields remain unchanged; `updatedAt` is updated if supported; DynamoDB and the frontend display the new values.                                               |
| **Actual Result**   | Record the data before and after the update, HTTP status, and displayed result.                                                                                                                                                            |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                              |
| **Evidence**        | Response, DynamoDB record before and after the update, CloudWatch Logs, and frontend screenshot.                                                                                                                                           |

---

#### API-05 — Start an Auction Session

| Field               | Content                                                                                                                                                                                                                                                                                   |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `API-05`                                                                                                                                                                                                                                                                                  |
| **Test Name**       | Transition session from `SCHEDULED` to `ACTIVE`                                                                                                                                                                                                                                           |
| **Objective**       | Verify that the start-session business operation is performed only when all required conditions are satisfied.                                                                                                                                                                            |
| **Prerequisites**   | The session is in `SCHEDULED` state; required data and items are available; the Admin is logged in.                                                                                                                                                                                       |
| **Test Steps**      | 1. Verify the initial status in DynamoDB. 2. Send the request to start the session. 3. Record the HTTP status and response. 4. Read the session again from DynamoDB. 5. Verify the status on the User Frontend and Admin Frontend. 6. Test a function available only for active sessions. |
| **Input Data**      | Session ID of a `SCHEDULED` session.                                                                                                                                                                                                                                                      |
| **Expected Result** | The API returns HTTP `200`; the status changes to `ACTIVE`; the actual start time is recorded if applicable; the session is updated only once; the frontend displays the session as active and enables appropriate operations.                                                            |
| **Actual Result**   | Record the state before and after, update time, and frontend result.                                                                                                                                                                                                                      |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                             |
| **Evidence**        | API response, DynamoDB records before and after, CloudWatch Logs, and screenshot showing the `ACTIVE` state.                                                                                                                                                                              |

---

#### API-06 — End an Auction Session

| Field               | Content                                                                                                                                                                                                                                                                           |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `API-06`                                                                                                                                                                                                                                                                          |
| **Test Name**       | Transition session from `ACTIVE` to `ENDED`                                                                                                                                                                                                                                       |
| **Objective**       | Verify that an active session can be ended and that the system blocks operations that are no longer valid.                                                                                                                                                                        |
| **Prerequisites**   | The session is in `ACTIVE` state; the Admin is logged in or the automatic ending mechanism is available.                                                                                                                                                                          |
| **Test Steps**      | 1. Verify the initial state. 2. Send the request to end the session. 3. Record the response. 4. Read the updated data in DynamoDB. 5. Reload the session page on the frontend. 6. Attempt an operation that is only allowed while the session is `ACTIVE`, such as placing a bid. |
| **Input Data**      | Session ID of an `ACTIVE` session.                                                                                                                                                                                                                                                |
| **Expected Result** | The API returns HTTP `200`; the status changes to `ENDED`; the end time is recorded; the frontend displays the session as ended; the system no longer accepts operations that are only valid for `ACTIVE` sessions.                                                               |
| **Actual Result**   | Record the status, end time, response, and result of the post-ending operation attempt.                                                                                                                                                                                           |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                     |
| **Evidence**        | Response, DynamoDB data before and after, CloudWatch Logs, and frontend screenshot.                                                                                                                                                                                               |

---

#### API-07 — User Cannot Create or Modify Auction Sessions

| Field               | Content                                                                                                                                                                                                                            |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `API-07`                                                                                                                                                                                                                           |
| **Test Name**       | Reject User attempts to perform session administration operations                                                                                                                                                                  |
| **Objective**       | Verify that a regular User cannot create, update, start, or end an auction session.                                                                                                                                                |
| **Prerequisites**   | The User is logged in but does not belong to the Cognito Group `Admin`; an administrative endpoint is available for testing.                                                                                                       |
| **Test Steps**      | 1. Log in as User. 2. Send a session creation request. 3. Send a request to update an existing session. 4. If applicable, attempt to start or end a session. 5. Verify the HTTP status. 6. Check DynamoDB data after each request. |
| **Input Data**      | Valid User token and correctly formatted requests.                                                                                                                                                                                 |
| **Expected Result** | Administrative APIs return HTTP `403`; the Business Logic Lambda does not perform unauthorized changes; no new session is created; existing sessions are not modified; the frontend does not display or allow Admin functions.     |
| **Actual Result**   | Record the status of each request and the data state after testing.                                                                                                                                                                |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                      |
| **Evidence**        | HTTP `403` responses, unchanged DynamoDB data, and User Frontend screenshot.                                                                                                                                                       |

> Hiding administrative buttons on the frontend does not replace backend authorization checks. The test case must send requests directly to the API using the User's token.

---

#### API-08 — Missing or Invalid Input Data

| Field               | Content                                                                                                                                                                                                                                                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `API-08`                                                                                                                                                                                                                                                                                                                             |
| **Test Name**       | Validate request input                                                                                                                                                                                                                                                                                                               |
| **Objective**       | Verify that the API rejects requests with missing required fields, invalid types, or rule violations.                                                                                                                                                                                                                                |
| **Prerequisites**   | The Admin is logged in and has permission to call the create or update session API.                                                                                                                                                                                                                                                  |
| **Test Steps**      | 1. Send a request without a session name. 2. Send a request without `startTime` or `endTime`. 3. Send a request with an invalid time format. 4. Send a request where `startTime` is greater than or equal to `endTime`. 5. Send a request containing an incorrect data type. 6. Verify the response and DynamoDB after each request. |
| **Input Data**      | Requests with missing fields or invalid data.                                                                                                                                                                                                                                                                                        |
| **Expected Result** | The API returns HTTP `400`; the response identifies the invalid field or rule; no partial data is created or updated in DynamoDB; the frontend displays an understandable message.                                                                                                                                                   |
| **Actual Result**   | Record each input set, HTTP status, error code, and DynamoDB state.                                                                                                                                                                                                                                                                  |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                        |
| **Evidence**        | Invalid request/response examples, unchanged DynamoDB data, and frontend validation screenshots.                                                                                                                                                                                                                                     |

---

#### API-09 — Resource ID Does Not Exist

| Field               | Content                                                                                                                                                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `API-09`                                                                                                                                                                                                                 |
| **Test Name**       | Access or update a non-existent resource                                                                                                                                                                                 |
| **Objective**       | Verify that the API handles non-existent Session IDs or Item IDs correctly.                                                                                                                                              |
| **Prerequisites**   | A correctly formatted ID that does not exist in DynamoDB is available.                                                                                                                                                   |
| **Test Steps**      | 1. Call the detail API using a non-existent ID. 2. Call the update API using that ID. 3. If applicable, attempt to start or end a session using that ID. 4. Record the response. 5. Verify DynamoDB and CloudWatch Logs. |
| **Input Data**      | Non-existent Session ID or Item ID.                                                                                                                                                                                      |
| **Expected Result** | The API returns HTTP `404`; the response contains a consistent error code and message; no unintended resource is created; no data is modified; the frontend displays a resource-not-found message.                       |
| **Actual Result**   | Record the HTTP status, error code, and data state.                                                                                                                                                                      |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                            |
| **Evidence**        | HTTP `404` response, DynamoDB lookup result, and frontend screenshot.                                                                                                                                                    |

---

#### API-10 — Current Session State Does Not Allow the Operation

| Field               | Content                                                                                                                                                                                                                                                                        |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `API-10`                                                                                                                                                                                                                                                                       |
| **Test Name**       | Reject invalid state transitions or operations                                                                                                                                                                                                                                 |
| **Objective**       | Verify that the Business Logic Lambda correctly enforces the auction session state machine.                                                                                                                                                                                    |
| **Prerequisites**   | Sessions exist in `SCHEDULED`, `ACTIVE`, and `ENDED` states.                                                                                                                                                                                                                   |
| **Test Steps**      | 1. Attempt to end a `SCHEDULED` session directly if that flow is not allowed. 2. Attempt to start an `ACTIVE` session again. 3. Attempt to start or edit an `ENDED` session. 4. Attempt to end an `ENDED` session again. 5. Verify the response and data after each operation. |
| **Input Data**      | Session ID and an operation that is invalid for the current state.                                                                                                                                                                                                             |
| **Expected Result** | The API returns HTTP `409` or `400` according to the API contract; the response indicates that the current state does not allow the operation; session state and data remain unchanged; the frontend displays an appropriate message.                                          |
| **Actual Result**   | Record the initial state, attempted operation, HTTP status, and state after the request.                                                                                                                                                                                       |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                  |
| **Evidence**        | Business error response, DynamoDB data before and after, CloudWatch Logs, and frontend screenshot.                                                                                                                                                                             |

Valid state transitions should follow the project's state machine, for example:

```text
SCHEDULED → ACTIVE → ENDED
```

The following transitions are not allowed:

```text
ENDED → ACTIVE
ACTIVE → SCHEDULED
ENDED → SCHEDULED
```

If the project supports the `CANCELLED` state, related transitions must be explicitly defined and tested according to the business contract.

---

#### API-11 — Verify Data Consistency in DynamoDB

| Field               | Content                                                                                                                                                                                                                                                                                                                      |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `API-11`                                                                                                                                                                                                                                                                                                                     |
| **Test Name**       | Verify data after auction session management operations                                                                                                                                                                                                                                                                      |
| **Objective**       | Verify that the actual data in DynamoDB is consistent with the request, response, and business rules.                                                                                                                                                                                                                        |
| **Prerequisites**   | At least one valid create, update, start, or end session operation can be performed.                                                                                                                                                                                                                                         |
| **Test Steps**      | 1. Record the data before the operation. 2. Perform the operation through the REST API. 3. Record the response. 4. Read the corresponding record directly from DynamoDB. 5. Compare the ID, state, time, creator, and related attributes. 6. Verify the number of records. 7. Call the read API again to confirm the result. |
| **Input Data**      | A valid request to create, update, start, or end a session.                                                                                                                                                                                                                                                                  |
| **Expected Result** | DynamoDB contains exactly the data accepted by the business operation; no duplicate records exist; partition and sort keys are correct; state is correct; `createdBy` or `updatedBy` is derived from a verified claim; unrelated attributes are preserved; the read API returns the latest stored data.                      |
| **Actual Result**   | Record the data before and after the operation and the fields that were compared.                                                                                                                                                                                                                                            |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                |
| **Evidence**        | Request/response and DynamoDB records before and after the operation.                                                                                                                                                                                                                                                        |

If the API returns success but DynamoDB contains no data, stores incorrect data, or the frontend continues to display stale data, the test case must be marked `FAIL`.

---

#### API-12 — Verify API Contract and Frontend Results

| Field               | Content                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `API-12`                                                                                                                                                                                                                                                                                                                                                                |
| **Test Name**       | Verify HTTP status, JSON response, and frontend integration                                                                                                                                                                                                                                                                                                             |
| **Objective**       | Verify that the frontend can reliably consume API responses and display correct business results.                                                                                                                                                                                                                                                                       |
| **Prerequisites**   | The User Frontend and Admin Frontend are integrated with the REST API.                                                                                                                                                                                                                                                                                                  |
| **Test Steps**      | 1. Perform a successful data retrieval request. 2. Perform a successful create or update request. 3. Perform a failed validation request. 4. Request a non-existent resource. 5. Perform a request without sufficient permissions. 6. Verify the HTTP status, `Content-Type`, and JSON of each response. 7. Verify how the frontend handles each result.                |
| **Input Data**      | Representative requests for success, invalid input, not found, and insufficient permission cases.                                                                                                                                                                                                                                                                       |
| **Expected Result** | The API returns the correct statuses such as `200`, `201`, `400`, `403`, `404`, or `409`; `Content-Type` is `application/json`; JSON fields have consistent names and data types; the frontend displays updated data after successful operations and appropriate error messages after failures; no JavaScript errors occur because of inconsistent response structures. |
| **Actual Result**   | Record the HTTP status, JSON structure, and frontend result for each scenario.                                                                                                                                                                                                                                                                                          |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                                                           |
| **Evidence**        | Network tab, request/response, frontend screenshots, and console logs with sensitive data removed.                                                                                                                                                                                                                                                                      |

---

#### State Transition Matrix to Verify

| Current State | Operation                      | Expected State | Result   |
| ------------- | ------------------------------ | -------------- | -------- |
| `SCHEDULED`   | Update valid information       | `SCHEDULED`    | Allowed  |
| `SCHEDULED`   | Start session                  | `ACTIVE`       | Allowed  |
| `ACTIVE`      | Start again                    | No change      | Rejected |
| `ACTIVE`      | End session                    | `ENDED`        | Allowed  |
| `ACTIVE`      | Transition back to `SCHEDULED` | No change      | Rejected |
| `ENDED`       | Start again                    | No change      | Rejected |
| `ENDED`       | End again                      | No change      | Rejected |
| `ENDED`       | Edit locked business data      | No change      | Rejected |

The matrix above should be adjusted if the official business contract defines additional states or operations.

---

#### DynamoDB Verification Guidelines

For each API that modifies data, the team should verify:

* The partition key and sort key match the table design.
* The ID in the response matches the ID in DynamoDB.
* Data is created only once.
* The state is updated correctly.
* `createdAt` does not change during an update.
* `updatedAt` changes after a successful update.
* `createdBy` or `updatedBy` is derived from the `sub` claim.
* `userId` or `role` from the request body is not treated as trusted data.
* No partial update occurs if the business operation fails.
* No unrelated attributes are deleted.
* DynamoDB data types match the design.
* Data read back through the API matches the stored data.

If the system uses multiple tables or multiple items for a single business operation, the team must verify all related data, not only the primary item.

---

#### CloudWatch Logs Verification Guidelines

CloudWatch Logs should support request tracing without containing sensitive information.

The information that should be recorded includes:

* Request ID.
* Invoked Lambda Function.
* Business operation name.
* Resource ID.
* Verified User `sub`.
* State before and after the operation.
* Success result or error code.
* Processing time.

The following must not be written to logs:

* Access Token.
* ID Token.
* Refresh Token.
* `Authorization` header.
* Password.
* AWS credentials.
* Unnecessary personal data.

---

#### Test Result Summary

| ID       | Test Content                     | Expected HTTP Status | DynamoDB Verification | Frontend Verification | Status |
| -------- | -------------------------------- | -------------------- | --------------------- | --------------------- | ------ |
| `API-01` | Retrieve session list            | `200`                | Yes                   | Yes                   | Tested |
| `API-02` | View session or item details     | `200`                | Yes                   | Yes                   | Tested |
| `API-03` | Admin creates session            | `201`                | Yes                   | Yes                   | Tested |
| `API-04` | Admin updates session            | `200`                | Yes                   | Yes                   | Tested |
| `API-05` | Start session                    | `200`                | Yes                   | Yes                   | Tested |
| `API-06` | End session                      | `200`                | Yes                   | Yes                   | Tested |
| `API-07` | User creates or modifies session | `403`                | Must remain unchanged | Yes                   | Tested |
| `API-08` | Invalid input data               | `400`                | Must remain unchanged | Yes                   | Tested |
| `API-09` | Resource does not exist          | `404`                | Must remain unchanged | Yes                   | Tested |
| `API-10` | State does not allow operation   | `409` or `400`       | Must remain unchanged | Yes                   | Tested |
| `API-11` | DynamoDB consistency             | Depends on operation | Required              | Read data back        | Tested |
| `API-12` | HTTP status, JSON, and frontend  | Depends on scenario  | When data changes     | Required              | Tested |