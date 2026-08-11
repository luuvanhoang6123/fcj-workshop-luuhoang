---
title: "WebSocket Connection and Real-Time Update Testing"
date: 2026-08-03
weight: 5
chapter: false
pre: " <b> 5.5.5. </b> "
---


#### Test Objectives

This section tests the system's ability to establish WebSocket connections and update auction data in real time through:

- Amazon API Gateway WebSocket API.
- Lambda WebSocket Handler.
- The `$connect`, `$disconnect`, and `$default` routes.
- Routes for joining auction rooms and sending messages.
- The connection management table in Amazon DynamoDB.
- Lambda Broadcast.
- API Gateway Management API.
- Amazon CloudWatch Logs.
- The auction system frontend.

The test cases are numbered from `WS-01` to `WS-13`.

A WebSocket test case is marked as `PASS` only when all of the following are verified:

1. The client receives the correct connection status or message.
2. The WebSocket Handler performs the correct business logic.
3. The Connection ID is correctly stored, updated, or deleted in DynamoDB.
4. Lambda Broadcast sends data to the correct users and auction room.
5. An invalid or expired connection does not cause the entire broadcast process to fail.
6. Private auction room data is not sent to users in other rooms.
7. Direct evidence is available from the browser, DynamoDB, or CloudWatch Logs.

---

#### Testing Scope

The components under test include:

- User Frontend.
- Amazon API Gateway WebSocket API.
- `$connect` route.
- `$disconnect` route.
- `$default` route.
- Auction room join or leave routes.
- Lambda WebSocket Handler.
- Lambda Broadcast.
- API Gateway Management API.
- DynamoDB Connections Table.
- DynamoDB Auction Room or Subscription Table, if implemented separately.
- Amazon CloudWatch Logs.
- Amazon Cognito or the WebSocket authentication mechanism used by the system.

---

#### General Test Prerequisites

Before performing the tests, the system must meet the following conditions:

- The WebSocket API has been deployed on API Gateway.
- The WebSocket URL for the test environment has been configured in the frontend.
- The `$connect`, `$disconnect`, and `$default` routes are linked to the correct Lambda Functions.
- Business routes such as `join_room`, `leave_room`, or `send_message` have been implemented if the architecture uses dedicated routes.
- Lambda has permission to read, write, and delete data in the DynamoDB connection table.
- Lambda Broadcast has permission to call `execute-api:ManageConnections`.
- CloudWatch Logs are enabled for the related Lambda Functions.
- At least two valid User accounts are available.
- At least two different auction items or auction rooms are available.
- Two browser windows or independent browser sessions can be opened.
- DynamoDB records can be inspected directly.
- The testing environment is isolated from production data.
- The clock on the testing device is synchronized so that log timestamps can be compared accurately.

If a related Lambda Function, route, DynamoDB table, or frontend feature has not yet been implemented, the test case must be marked as `BLOCKED`.

---

#### Test Data

| Data | Description |
| --- | --- |
| User A | Valid account participating in Auction Item A |
| User B | Valid account participating in the same Auction Item A |
| User C | Valid account participating only in Auction Item B |
| Auction Item A | Item with a valid WebSocket room |
| Auction Item B | A different item from Item A |
| Room A | WebSocket room for Auction Item A |
| Room B | WebSocket room for Auction Item B |
| Valid Connection ID | Active Connection ID assigned by API Gateway |
| Expired Connection ID | Connection ID belonging to a client that has closed or lost its connection |
| Valid message | JSON containing the correct `action`, `roomId`, and required fields |
| Invalid message | Malformed JSON, missing fields, or unsupported action |
| Valid event | Status update, bid update, viewer count update, or system notification |
| Valid token | Unexpired token belonging to a valid account |
| Invalid token | Token with an invalid signature, expired token, or incorrectly formatted token |

Access Tokens, ID Tokens, Refresh Tokens, or authentication headers must not be included in screenshots or reports.

---

#### Connection Status and Message Structure Conventions

WebSocket does not use HTTP status codes for every message after the connection has been established. Therefore, results must be verified at two stages:

- Connection handshake stage: verify the HTTP status of the WebSocket handshake.
- Connected stage: verify WebSocket messages and connection status.

Common handshake results include:

| Result | Meaning |
| --- | --- |
| `101 Switching Protocols` | WebSocket connection established successfully |
| `401 Unauthorized` | Authentication information is missing or invalid |
| `403 Forbidden` | User is authenticated but is not allowed to connect |
| Connection closed | API Gateway or Lambda rejects or terminates the connection |

A successful message should use a consistent structure, for example:

```json
{
  "type": "AUCTION_STATUS_UPDATED",
  "roomId": "auction-item-a",
  "data": {
    "status": "ACTIVE"
  },
  "timestamp": "2026-08-09T12:00:00Z"
}
````

An error message should use a structure that the frontend can handle:

```json
{
  "type": "ERROR",
  "error": {
    "code": "INVALID_MESSAGE_FORMAT",
    "message": "The WebSocket message is invalid"
  },
  "timestamp": "2026-08-09T12:00:00Z"
}
```

Messages sent to clients must not contain:

* Access Tokens.
* ID Tokens.
* Refresh Tokens.
* AWS credentials.
* Connection IDs belonging to other users.
* Stack traces.
* DynamoDB table names.
* Unnecessary internal infrastructure details.
* Personal data belonging to other users.

---

#### WS-01 — User Successfully Connects to WebSocket

| Field               | Content                                                                                                                                                                                                                                         |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-01`                                                                                                                                                                                                                                         |
| **Test Name**       | Valid User successfully connects to WebSocket                                                                                                                                                                                                   |
| **Objective**       | Verify that an authenticated User can establish a connection to the API Gateway WebSocket API.                                                                                                                                                  |
| **Prerequisites**   | The WebSocket API and `$connect` route have been deployed; User A has valid authentication information.                                                                                                                                         |
| **Test Steps**      | 1. Log in as User A. 2. Open the Auction Item A detail page. 3. Open the Network tab and select the WebSocket filter. 4. Observe the WebSocket handshake. 5. Verify the connection status on the frontend. 6. Check the `$connect` Lambda logs. |
| **Input Data**      | Valid WebSocket URL and valid authentication information for User A.                                                                                                                                                                            |
| **Expected Result** | The handshake returns `101 Switching Protocols`; the frontend changes to Connected or Live state; the `$connect` Lambda is invoked exactly once; continuous reconnect loops do not occur; logs do not contain tokens.                           |
| **Actual Result**   | Record the actual HTTP status, connection state, connection time, and Request ID.                                                                                                                                                               |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                   |
| **Evidence**        | Network tab, Live status on the frontend, and `$connect` CloudWatch Logs.                                                                                                                                                                       |

---

#### WS-02 — Invalid Connection Is Rejected

| Field               | Content                                                                                                                                                                                                                                           |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-02`                                                                                                                                                                                                                                           |
| **Test Name**       | Reject an invalid WebSocket connection                                                                                                                                                                                                            |
| **Objective**       | Verify that a user without valid authentication information cannot establish a connection.                                                                                                                                                        |
| **Prerequisites**   | `$connect` performs authentication validation or an Authorizer has been configured.                                                                                                                                                               |
| **Test Steps**      | 1. Attempt to connect without authentication information. 2. Attempt to connect using an incorrectly formatted token. 3. Attempt to connect using an expired token. 4. Record the handshake results. 5. Check DynamoDB. 6. Check CloudWatch Logs. |
| **Input Data**      | No token, invalid token, or expired token.                                                                                                                                                                                                        |
| **Expected Result** | The connection is rejected with `401`, `403`, or closed according to the system contract; the frontend does not display Live status; no active Connection ID is stored in DynamoDB; logs contain an error code but not the token contents.        |
| **Actual Result**   | Record each test input and its corresponding result.                                                                                                                                                                                              |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                     |
| **Evidence**        | Handshake results, DynamoDB showing no valid connection record, and CloudWatch Logs with sensitive information masked.                                                                                                                            |

If a token is transmitted through the query string, the team must verify that the token is not recorded in access logs, browser history, or evidence screenshots. A WebSocket URL containing a token must not be publicly disclosed.

---

#### WS-03 — `$connect` Event Stores the Connection ID

| Field               | Content                                                                                                                                                                                                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-03`                                                                                                                                                                                                                                                                                                 |
| **Test Name**       | Store connection information after a successful `$connect`                                                                                                                                                                                                                                              |
| **Objective**       | Verify that the `$connect` Lambda correctly stores the Connection ID and user information in DynamoDB.                                                                                                                                                                                                  |
| **Prerequisites**   | User A can connect; Lambda has permission to write to the Connections Table.                                                                                                                                                                                                                            |
| **Test Steps**      | 1. Record the table contents before connecting. 2. User A opens Auction Item A. 3. Confirm that the connection succeeds. 4. Locate the new record in DynamoDB. 5. Compare the creation time, User ID, and Connection ID with CloudWatch Logs. 6. Verify the expiration attribute if the table uses TTL. |
| **Input Data**      | Valid connection from User A.                                                                                                                                                                                                                                                                           |
| **Expected Result** | One connection record is created; the Connection ID is not empty; the User ID is derived from the verified identity; `connectedAt` is stored correctly; TTL is in the future if used; tokens are not stored in DynamoDB.                                                                                |
| **Actual Result**   | Record the partially masked Connection ID, User ID, creation time, and TTL.                                                                                                                                                                                                                             |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                           |
| **Evidence**        | DynamoDB record and `$connect` CloudWatch Logs.                                                                                                                                                                                                                                                         |

---

#### WS-04 — User Joins the Correct Auction Room

| Field               | Content                                                                                                                                                                                                                                                           |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-04`                                                                                                                                                                                                                                                           |
| **Test Name**       | Join the correct WebSocket room for the auction item                                                                                                                                                                                                              |
| **Objective**       | Verify that User A is associated with Room A when Auction Item A is opened.                                                                                                                                                                                       |
| **Prerequisites**   | User A is connected; Auction Item A exists; the room-join route has been implemented.                                                                                                                                                                             |
| **Test Steps**      | 1. User A opens Auction Item A. 2. The frontend sends a `join_room` message if required by the architecture. 3. Verify the response message. 4. Verify the connection record in DynamoDB. 5. Check the WebSocket Handler logs. 6. Publish a test event to Room A. |
| **Input Data**      | Room ID or Auction Item ID for Item A.                                                                                                                                                                                                                            |
| **Expected Result** | User A's connection is associated with Room A; Lambda verifies that Room A exists; the client receives a join confirmation; Room A events are delivered to User A; duplicate subscriptions are not created for the same connection and room.                      |
| **Actual Result**   | Record the Room ID, received response, and DynamoDB data.                                                                                                                                                                                                         |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                     |
| **Evidence**        | WebSocket frame, DynamoDB record, and CloudWatch Logs.                                                                                                                                                                                                            |

---

#### WS-05 — Two Users Join the Same Auction Item

| Field               | Content                                                                                                                                                                                                                                                                                 |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-05`                                                                                                                                                                                                                                                                                 |
| **Test Name**       | Two Users join Room A                                                                                                                                                                                                                                                                   |
| **Objective**       | Verify that the system can manage multiple concurrent connections in the same auction room.                                                                                                                                                                                             |
| **Prerequisites**   | User A and User B are available; two independent browser sessions can be opened.                                                                                                                                                                                                        |
| **Test Steps**      | 1. Open Auction Item A as User A in the first window. 2. Open the same Item A as User B in the second window. 3. Confirm that both connections are in Live state. 4. Check the viewer count if supported by the frontend. 5. Check DynamoDB records. 6. Publish a test event to Room A. |
| **Input Data**      | Two different Users connected to the same Room A.                                                                                                                                                                                                                                       |
| **Expected Result** | Two different Connection IDs are associated with Room A; the viewer count is `2` if supported; both windows receive the Room A event; one connection does not overwrite the other.                                                                                                      |
| **Actual Result**   | Record the number of connections, viewer count, and messages received in each window.                                                                                                                                                                                                   |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                           |
| **Evidence**        | Screenshots of both windows operating simultaneously, WebSocket frames, DynamoDB data, and broadcast logs.                                                                                                                                                                              |

---

#### WS-06 — User Sends a Valid Message

| Field               | Content                                                                                                                                                                                                                                                                                              |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-06`                                                                                                                                                                                                                                                                                              |
| **Test Name**       | Process a valid WebSocket message                                                                                                                                                                                                                                                                    |
| **Objective**       | Verify that the WebSocket Handler correctly receives, validates, and processes a valid message.                                                                                                                                                                                                      |
| **Prerequisites**   | User A is connected and has joined Room A.                                                                                                                                                                                                                                                           |
| **Test Steps**      | 1. User A sends a message containing a supported action. 2. Verify the outgoing frame. 3. Verify the server response. 4. Check CloudWatch Logs. 5. If the message causes a broadcast, verify the result for User B. 6. If the message changes data, verify the corresponding source data.            |
| **Input Data**      | Valid JSON with the correct `action`, `roomId`, and required fields.                                                                                                                                                                                                                                 |
| **Expected Result** | The Handler reads the correct action; verifies the User and Room; processes the business operation exactly once; the client receives an ACK or appropriate response; related Users receive the correct event; the client cannot define its own trusted identity or permissions through message data. |
| **Actual Result**   | Record the message type, response, and actual broadcast result.                                                                                                                                                                                                                                      |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                        |
| **Evidence**        | WebSocket frames with sensitive data masked, CloudWatch Logs, and related data.                                                                                                                                                                                                                      |

---

#### WS-07 — Invalid Message Format Is Rejected

| Field               | Content                                                                                                                                                                                                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `WS-07`                                                                                                                                                                                                                                                                                          |
| **Test Name**       | Reject an invalid WebSocket message                                                                                                                                                                                                                                                              |
| **Objective**       | Verify that Lambda safely handles messages that are not valid JSON, are missing required fields, or contain unsupported actions.                                                                                                                                                                 |
| **Prerequisites**   | User A is connected to WebSocket.                                                                                                                                                                                                                                                                |
| **Test Steps**      | 1. Send a non-JSON string. 2. Send JSON without `action`. 3. Send an unsupported action. 4. Send a message without `roomId` when required. 5. Send a field with an incorrect data type. 6. Check the response, logs, and data after each attempt.                                                |
| **Input Data**      | Malformed messages or messages that violate the schema.                                                                                                                                                                                                                                          |
| **Expected Result** | The server returns a consistently structured error message; no business operation is performed; invalid messages are not broadcast; no unintended data changes occur; Lambda does not produce an unhandled error; the connection may remain open or be closed according to the defined contract. |
| **Actual Result**   | Record each test message, error code, and connection state after the error.                                                                                                                                                                                                                      |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                    |
| **Evidence**        | Sent and received frames, unchanged DynamoDB or business data, and CloudWatch Logs.                                                                                                                                                                                                              |

---

#### WS-08 — Auction Status Is Broadcast to All Participants

| Field               | Content                                                                                                                                                                                                                                                                                       |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-08`                                                                                                                                                                                                                                                                                       |
| **Test Name**       | Broadcast status updates to all members of Room A                                                                                                                                                                                                                                             |
| **Objective**       | Verify that Lambda Broadcast sends auction status updates to every active connection in the correct room.                                                                                                                                                                                     |
| **Prerequisites**   | User A and User B are participating in Room A; Lambda Broadcast and the Management API are available.                                                                                                                                                                                         |
| **Test Steps**      | 1. Open Auction Item A in two windows. 2. Perform a business operation that changes state or generates a valid event. 3. Record the event publication time. 4. Observe the message in both windows. 5. Verify that the interfaces update. 6. Check Lambda Broadcast logs.                     |
| **Input Data**      | Events such as `AUCTION_STATUS_UPDATED`, `BID_UPDATED`, or `VIEWER_COUNT_UPDATED`.                                                                                                                                                                                                            |
| **Expected Result** | User A and User B receive the same event type, Room ID, and updated data; the frontend updates without requiring a page reload; duplicate messages are not sent beyond the designed behavior; logs show the number of target connections and the numbers of successful and failed deliveries. |
| **Actual Result**   | Record the message received in each window and the observed latency.                                                                                                                                                                                                                          |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                 |
| **Evidence**        | Screenshots of both windows, WebSocket messages, and Lambda Broadcast CloudWatch Logs.                                                                                                                                                                                                        |

---

#### WS-09 — User Leaves the Page or Disconnects

| Field               | Content                                                                                                                                                                                                                                            |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-09`                                                                                                                                                                                                                                            |
| **Test Name**       | User leaves the auction room                                                                                                                                                                                                                       |
| **Objective**       | Verify that the system correctly handles a User closing the page, navigating away, or losing the connection.                                                                                                                                       |
| **Prerequisites**   | User A and User B are both participating in Room A.                                                                                                                                                                                                |
| **Test Steps**      | 1. Confirm that both Users are connected. 2. Close User B's tab or navigate away from Item A. 3. Observe the state in User A's window. 4. Check viewer count or room-leave notification if supported. 5. Check CloudWatch Logs. 6. Check DynamoDB. |
| **Input Data**      | User B closes the tab, navigates away, or loses network connectivity.                                                                                                                                                                              |
| **Expected Result** | User B is no longer treated as an active member of Room A; viewer count decreases from `2` to `1` if supported; User A receives the corresponding update event; User A's connection remains unaffected.                                            |
| **Actual Result**   | Record both client states, viewer count, and the time the disconnection is detected.                                                                                                                                                               |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                      |
| **Evidence**        | Screenshots before and after leaving the page, User A frames, DynamoDB data, and CloudWatch Logs.                                                                                                                                                  |

When a device suddenly loses network connectivity, `$disconnect` may not be processed immediately. If the system uses heartbeat or TTL mechanisms, the maximum time required to detect and clean up the connection should be documented.

---

#### WS-10 — `$disconnect` Removes or Deactivates the Connection

| Field               | Content                                                                                                                                                                                                                                                                    |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-10`                                                                                                                                                                                                                                                                    |
| **Test Name**       | Clean up the connection record when `$disconnect` occurs                                                                                                                                                                                                                   |
| **Objective**       | Verify that the `$disconnect` Lambda removes or marks the disconnected Connection ID as inactive.                                                                                                                                                                          |
| **Prerequisites**   | User B has an active Connection ID stored in DynamoDB.                                                                                                                                                                                                                     |
| **Test Steps**      | 1. Record User B's connection record before disconnection. 2. Close User B's WebSocket connection. 3. Check the `$disconnect` logs. 4. Read the DynamoDB record again. 5. Perform another broadcast to Room A. 6. Verify the list of connections used by Lambda Broadcast. |
| **Input Data**      | Active Connection ID belonging to User B.                                                                                                                                                                                                                                  |
| **Expected Result** | `$disconnect` receives the correct Connection ID; the record is deleted or marked inactive according to the design; User B is no longer included in the broadcast list; cleanup is idempotent and does not fail if repeated.                                               |
| **Actual Result**   | Record the record state before and after `$disconnect`.                                                                                                                                                                                                                    |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                              |
| **Evidence**        | `$disconnect` CloudWatch Logs, DynamoDB records before and after, and the next broadcast log.                                                                                                                                                                              |

---

#### WS-11 — Expired Connection Does Not Cause the Entire Broadcast to Fail

| Field               | Content                                                                                                                                                                                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-11`                                                                                                                                                                                                                                                                   |
| **Test Name**       | Isolate failure of an expired connection                                                                                                                                                                                                                                  |
| **Objective**       | Verify that an invalid Connection ID does not prevent Lambda from sending data to other active connections.                                                                                                                                                               |
| **Prerequisites**   | Room A contains at least one valid connection and one expired or non-existent Connection ID.                                                                                                                                                                              |
| **Test Steps**      | 1. Keep User A connected. 2. Create a condition where an old User B connection record remains in DynamoDB. 3. Publish an event to Room A. 4. Observe the message received by User A. 5. Check Lambda Broadcast results. 6. Check the error log for the old Connection ID. |
| **Input Data**      | One valid Connection ID and one expired Connection ID in Room A.                                                                                                                                                                                                          |
| **Expected Result** | User A still receives the event; Lambda handles errors per Connection ID; one failed delivery does not terminate the entire broadcast loop; logs show at least one success and one failure; Lambda does not fail entirely because of one stale connection.                |
| **Actual Result**   | Record the number of target connections, successful deliveries, failed deliveries, and User A result.                                                                                                                                                                     |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                             |
| **Evidence**        | User A message, CloudWatch Logs, and the expired Connection ID record.                                                                                                                                                                                                    |

---

#### WS-12 — `GoneException` Is Handled and the Connection Is Removed from DynamoDB

| Field               | Content                                                                                                                                                                                                                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-12`                                                                                                                                                                                                                                                                       |
| **Test Name**       | Clean up a stale connection when the Management API returns `GoneException`                                                                                                                                                                                                   |
| **Objective**       | Verify that Lambda Broadcast recognizes HTTP `410 Gone` and removes an invalid Connection ID.                                                                                                                                                                                 |
| **Prerequisites**   | DynamoDB still contains a Connection ID for a closed connection; Lambda Broadcast has permission to delete or update the record.                                                                                                                                              |
| **Test Steps**      | 1. Identify an expired Connection ID. 2. Confirm that the record still exists before broadcasting. 3. Invoke Lambda Broadcast. 4. Check the `postToConnection` logs. 5. Verify that `GoneException` is caught. 6. Read DynamoDB again. 7. Invoke the broadcast a second time. |
| **Input Data**      | Connection ID that no longer exists in API Gateway but is still stored in DynamoDB.                                                                                                                                                                                           |
| **Expected Result** | The Management API returns an error equivalent to `410 Gone`; Lambda catches the error without terminating the entire broadcast; the stale connection record is removed or deactivated; the next broadcast no longer attempts delivery to that Connection ID.                 |
| **Actual Result**   | Record the error code, cleanup action, and record state after processing.                                                                                                                                                                                                     |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                 |
| **Evidence**        | CloudWatch Logs containing `GoneException` or `410`, DynamoDB before and after, and logs from the subsequent broadcast.                                                                                                                                                       |

Not every Management API error should be treated as an expired connection. Only errors that clearly indicate the connection no longer exists, such as `GoneException`, should be used as a reason to delete the record.

---

#### WS-13 — Users Outside the Room Do Not Receive Private Room Data

| Field               | Content                                                                                                                                                                                                                                                                                               |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `WS-13`                                                                                                                                                                                                                                                                                               |
| **Test Name**       | Isolate data between auction rooms                                                                                                                                                                                                                                                                    |
| **Objective**       | Verify that Room A events are sent only to connections belonging to Room A.                                                                                                                                                                                                                           |
| **Prerequisites**   | User A and User B are in Room A; User C is in Room B.                                                                                                                                                                                                                                                 |
| **Test Steps**      | 1. Open Room A as User A and User B. 2. Open Room B as User C. 3. Confirm that all three connections are active. 4. Publish an event belonging only to Room A. 5. Check the messages in all three windows. 6. Check the DynamoDB query used by Lambda Broadcast. 7. Repeat using an event for Room B. |
| **Input Data**      | An event specific to Auction Item A and an event specific to Auction Item B.                                                                                                                                                                                                                          |
| **Expected Result** | User A and User B receive the Room A event; User C does not receive it; User C only receives the Room B event; Lambda queries connections using the correct Room ID; bid data, status, or private auction information is not sent across rooms.                                                       |
| **Actual Result**   | Record which messages were or were not received in each window.                                                                                                                                                                                                                                       |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                         |
| **Evidence**        | Screenshots of the three windows or WebSocket frames, broadcast queries/logs, and subscription data in DynamoDB.                                                                                                                                                                                      |

---

#### Event Distribution Matrix to Verify

| User                 | Joined Room            | Room A Event                                | Room B Event                   |
| -------------------- | ---------------------- | ------------------------------------------- | ------------------------------ |
| User A               | Room A                 | Must receive                                | Must not receive               |
| User B               | Room A                 | Must receive                                | Must not receive               |
| User C               | Room B                 | Must not receive                            | Must receive                   |
| Expired connection   | Room A                 | Delivery fails and connection is cleaned up | Not applicable                 |
| User who left Room A | No active subscription | Must not receive                            | Receives only if joined Room B |

---

#### DynamoDB Connections Table Verification Guidelines

For the connection management table, verify the following:

* Connection ID is obtained from `requestContext.connectionId`.
* User ID is obtained from the verified identity.
* `userId`, `role`, or `connectionId` sent by the client in a message is not trusted.
* Each connection is associated with the correct User.
* Each subscription is associated with the correct Room ID or Auction Item ID.
* Two tabs belonging to the same User may have different Connection IDs.
* Closed connections are removed or deactivated.
* TTL is configured correctly if automatic cleanup is used.
* Access Tokens, ID Tokens, or Refresh Tokens are not stored.
* One user's connection does not overwrite another user's connection.
* Duplicate subscriptions do not exist unless explicitly designed.
* `GoneException` results in cleanup of the corresponding Connection ID.
* Broadcast queries only connections belonging to the target room.
* Failure of one connection does not prevent other connections from receiving updates.

If the table uses a single-table design, the team must verify the correct partition key, sort key, and indexes used to query connections by Room ID.

---

#### Lambda Broadcast Verification Guidelines

Lambda Broadcast should be tested according to the following criteria:

* Receives the correct Room ID and event type.
* Queries the correct connection list for the room.
* Does not rely entirely on a list of Connection IDs provided by the client.
* Sends data using the correct WebSocket API endpoint and stage.
* Uses valid JSON message structures.
* Does not send sensitive data.
* Continues processing when delivery to one connection fails.
* Catches and handles `GoneException`.
* Removes expired Connection IDs.
* Records the number of target connections.
* Records the number of successful and failed deliveries.
* Does not log full tokens or personal data.
* Provides a mechanism to limit message size.
* Does not send duplicate events outside the intended design.
* Prevents an error in one room from affecting another room.

---

#### CloudWatch Logs Verification Guidelines

CloudWatch Logs should contain the information required for tracing, including:

* Request ID.
* Route key such as `$connect`, `$disconnect`, `$default`, or `join_room`.
* Partially masked Connection ID when included in the report.
* Verified User ID or Cognito `sub`.
* Room ID or Auction Item ID.
* Message type or event type.
* Number of target connections.
* Number of successful deliveries.
* Number of failed deliveries.
* Error codes such as `INVALID_MESSAGE_FORMAT` or `GONE_CONNECTION`.
* Processing time.
* Result of stale connection cleanup.

The following must not be logged:

* Access Token.
* ID Token.
* Refresh Token.
* Headers or query parameters containing authentication information.
* Passwords.
* AWS Access Key ID.
* AWS Secret Access Key.
* AWS Session Token.
* Entire WebSocket messages if they contain sensitive data.
* Stack traces in responses returned to clients.

---

#### Test Result Summary

| ID      | Test Content                    | Main Expected Result                              | DynamoDB Verification | Status |
| ------- | ------------------------------- | ------------------------------------------------- | --------------------- | ------ |
| `WS-01` | User connects successfully      | Handshake `101`, frontend displays Live           | Yes                   | Tested |
| `WS-02` | Invalid connection              | Rejected, no active connection                    | Yes                   | Tested |
| `WS-03` | `$connect` stores Connection ID | Connection record is created correctly            | Required              | Tested |
| `WS-04` | User joins the correct room     | Connection is associated with the correct Room ID | Required              | Tested |
| `WS-05` | Two Users join the same item    | Both connections receive the event                | Required              | Tested |
| `WS-06` | Send valid message              | Handler processes and responds correctly          | When data changes     | Tested |
| `WS-07` | Invalid message format          | Rejected, not broadcast                           | Must remain unchanged | Tested |
| `WS-08` | Broadcast auction status        | All room members receive the update               | Yes                   | Tested |
| `WS-09` | User leaves the page            | User is no longer considered an active connection | Yes                   | Tested |
| `WS-10` | `$disconnect` cleans connection | Connection ID is removed or deactivated           | Required              | Tested |
| `WS-11` | Expired connection exists       | Valid connections still receive data              | Yes                   | Tested |
| `WS-12` | Handle `GoneException`          | Stale connection is removed from the table        | Required              | Tested |
| `WS-13` | Isolation between rooms         | Users outside the room do not receive data        | Yes                   | Tested |

---