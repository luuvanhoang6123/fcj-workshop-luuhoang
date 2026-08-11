---
title: "End-to-End Bidding Flow Testing"
date: 2026-08-03
weight: 6
chapter: false
pre: " <b> 5.5.6. </b> "
---

#### Objectives

This testing section verifies the complete real-time bidding flow of the Live Auction system, from the moment a user submits a bid request on the frontend until the new price is processed, stored, and broadcast to users who are watching the auction.

The flow to be tested is:

```text
Frontend
-> WebSocket API
-> la-ws-handler
-> SQS FIFO
-> la-bid-processor
-> DynamoDB
-> la-broadcast
-> WebSocket API
-> Frontend
````

The testing process must not only confirm that the frontend displays the new price, but must also demonstrate that:

* The bid request is authenticated.
* The message is sent to the correct SQS FIFO queue.
* Duplicate messages do not create multiple bids.
* Requests are processed in the correct order within the same auction.
* Price updates are atomic.
* The highest bidder is determined correctly.
* Bid history is stored completely.
* Results are broadcast to the correct auction room.
* The frontend updates without requiring a page reload.
* Tokens, authentication information, and internal infrastructure data are not exposed.

---

#### Related Components

The components that must be tested include:

* Auction item detail page on the frontend.
* API Gateway WebSocket API.
* WebSocket route used for bidding, for example `place_bid`.
* Lambda `la-ws-handler`.
* Amazon SQS FIFO.
* Dead-letter queue, if configured.
* Lambda `la-bid-processor`.
* DynamoDB Auction or Current Bid Table.
* DynamoDB Bid History Table.
* Lambda `la-broadcast`.
* API Gateway Management API.
* CloudWatch Logs.
* JWT authentication mechanism or Amazon Cognito.
* Access control mechanism for joining auction sessions.

---

#### General Test Prerequisites

Before testing, the system must meet the following conditions:

* The WebSocket API has been deployed.
* The frontend is configured with the correct WebSocket URL for the testing environment.
* The bidding route is linked to `la-ws-handler`.
* `la-ws-handler` has permission to send messages to SQS FIFO.
* SQS FIFO is configured correctly.
* `la-bid-processor` is configured to receive messages from SQS.
* `la-bid-processor` has permission to read and update DynamoDB data.
* `la-broadcast` has permission to call `execute-api:ManageConnections`.
* The tables for storing the current price and bid history already exist.
* The DynamoDB Connections or Subscriptions Table contains room data.
* At least two valid User accounts are available.
* At least one account exists that is not part of the auction or is not allowed to bid.
* At least one auction exists in each of the following states: not started, active, and ended.
* CloudWatch Logs are enabled.
* The testing environment is isolated from production.
* The team has permission to inspect SQS, DynamoDB, and CloudWatch Logs.
* The testing device clock is synchronized.

If any component in the flow has not been implemented, the related test case must be marked as `BLOCKED`.

---

#### Test Data

| Data                  | Description                                                                       |
| --------------------- | --------------------------------------------------------------------------------- |
| User A                | Valid User allowed to place bids                                                  |
| User B                | Second valid User in the same auction                                             |
| User C                | Valid User who does not belong to or is not allowed to participate in the auction |
| Anonymous User        | User who is not logged in                                                         |
| Auction Active        | Auction with status `ACTIVE`                                                      |
| Auction Scheduled     | Auction that has not started, with status `SCHEDULED`                             |
| Auction Ended         | Auction that has ended, with status `ENDED`                                       |
| Current Price         | Current item price, for example `1,000,000 VND`                                   |
| Minimum Increment     | Minimum bid increment, for example `100,000 VND`                                  |
| Valid Bid             | Valid bid amount, for example `1,100,000 VND`                                     |
| Low Bid               | Bid equal to or lower than the current price                                      |
| Invalid Increment Bid | Bid higher than the current price but below the required increment                |
| Request ID            | Unique identifier for a bid request                                               |
| Client Message ID     | Client-generated identifier used to support idempotency                           |
| Room ID               | WebSocket room identifier for the item                                            |
| Auction ID            | Auction session identifier                                                        |
| Item ID               | Auction item identifier                                                           |
| Expired Token         | Expired token                                                                     |
| Invalid Token         | Token with an invalid signature or invalid format                                 |

Production data must not be used during testing.

---

#### Bid Request Structure

Example message sent by the frontend:

```json
{
  "action": "place_bid",
  "requestId": "bid-request-001",
  "auctionId": "auction-active-001",
  "itemId": "item-001",
  "amount": 1100000
}
```

The frontend must not be allowed to send or determine trusted values such as:

```json
{
  "userId": "trusted-user-id",
  "role": "ADMIN",
  "currentPrice": 1000000,
  "isWinner": true
}
```

The bidder's identity must be obtained from a verified token or authentication context.

The current price, auction state, minimum increment, and highest bidder must be read from a trusted server-side data source.

---

#### Successful Result Structure

Example broadcast event for a successful bid:

```json
{
  "type": "BID_ACCEPTED",
  "requestId": "bid-request-001",
  "auctionId": "auction-active-001",
  "itemId": "item-001",
  "data": {
    "amount": 1100000,
    "highestBidderId": "masked-user-id",
    "bidSequence": 15
  },
  "timestamp": "2026-08-09T12:00:00Z"
}
```

#### Failure Result Structure

```json
{
  "type": "BID_REJECTED",
  "requestId": "bid-request-001",
  "error": {
    "code": "MINIMUM_INCREMENT_NOT_MET",
    "message": "The bid does not meet the minimum increment"
  },
  "timestamp": "2026-08-09T12:00:00Z"
}
```

The response must not contain:

* Access Token.
* ID Token.
* Refresh Token.
* AWS credentials.
* Unnecessary internal SQS message contents.
* DynamoDB table names.
* Stack traces.
* Connection IDs belonging to other users.
* Email addresses or unnecessary personal information.

---

### BID-01 — Valid Bid

| Field               | Content                                                                                                                                                                                                                                                                                                                               |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-01`                                                                                                                                                                                                                                                                                                                              |
| **Test Name**       | User places a valid bid during an active auction                                                                                                                                                                                                                                                                                      |
| **Objective**       | Verify that a valid bid request passes through the entire flow and is stored successfully.                                                                                                                                                                                                                                            |
| **Prerequisites**   | Auction Active is running; User A is logged in and has joined the correct room; current price is `1,000,000 VND`; minimum increment is `100,000 VND`.                                                                                                                                                                                 |
| **Test Steps**      | 1. User A opens the auction page. 2. Enter a bid of `1,100,000 VND`. 3. Submit the bid request. 4. Check the WebSocket frame. 5. Check `la-ws-handler` logs. 6. Verify that the message is sent to SQS FIFO. 7. Verify that `la-bid-processor` processes the message. 8. Check DynamoDB. 9. Check the broadcast message and frontend. |
| **Expected Result** | The request is accepted; the message is placed in SQS exactly once; the current price is updated to `1,100,000 VND`; User A becomes the highest bidder; one bid history record is created; users in the room receive `BID_ACCEPTED`; the frontend updates without reloading.                                                          |
| **Actual Result**   | Record the Request ID, Message ID, price before and after, highest bidder, and processing time.                                                                                                                                                                                                                                       |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                                                                         |
| **Evidence**        | WebSocket frames, CloudWatch Logs, SQS metrics, DynamoDB before and after, and frontend interface.                                                                                                                                                                                                                                    |

---

### BID-02 — Bid Equal to or Lower Than the Current Price

| Field               | Content                                                                                                                                                                                                                                         |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-02`                                                                                                                                                                                                                                        |
| **Test Name**       | Reject a bid that is not higher than the current price                                                                                                                                                                                          |
| **Objective**       | Verify that the system does not accept a bid equal to or lower than the current price.                                                                                                                                                          |
| **Prerequisites**   | Current price is `1,100,000 VND`; the auction is active.                                                                                                                                                                                        |
| **Test Steps**      | 1. Submit a bid of `1,100,000 VND`. 2. Submit another bid of `1,000,000 VND` using a different Request ID. 3. Check the result of each request. 4. Check DynamoDB and bid history. 5. Check broadcast behavior.                                 |
| **Expected Result** | Both requests are rejected with an error code such as `BID_NOT_HIGHER_THAN_CURRENT_PRICE`; current price remains unchanged; highest bidder remains unchanged; no successful bid history entry is created; no `BID_ACCEPTED` event is broadcast. |
| **Actual Result**   | Record each submitted amount, error code, and resulting price.                                                                                                                                                                                  |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                   |
| **Evidence**        | Error frames, CloudWatch Logs, and unchanged DynamoDB data.                                                                                                                                                                                     |

---

### BID-03 — Minimum Increment Not Met

| Field               | Content                                                                                                                                                                                                                                                   |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-03`                                                                                                                                                                                                                                                  |
| **Test Name**       | Reject a bid that does not meet the minimum increment                                                                                                                                                                                                     |
| **Objective**       | Verify that each new bid satisfies the minimum bid increment.                                                                                                                                                                                             |
| **Prerequisites**   | Current price is `1,100,000 VND`; minimum increment is `100,000 VND`.                                                                                                                                                                                     |
| **Test Steps**      | 1. User A submits a bid of `1,150,000 VND`. 2. Observe the processing flow. 3. Check the response. 4. Check DynamoDB. 5. Check whether other users receive an update.                                                                                     |
| **Expected Result** | The request is rejected with `MINIMUM_INCREMENT_NOT_MET`; the server may return a minimum acceptable bid of `1,200,000 VND`; current price and highest bidder remain unchanged; no successful bid history entry is created; no price update is broadcast. |
| **Actual Result**   | Record the submitted amount, minimum acceptable bid, and error code.                                                                                                                                                                                      |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                             |
| **Evidence**        | WebSocket response, processor logs, and DynamoDB.                                                                                                                                                                                                         |

---

### BID-04 — Bid Before the Auction Starts

| Field               | Content                                                                                                                                                                                                               |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-04`                                                                                                                                                                                                              |
| **Test Name**       | Reject bids submitted before the auction start time                                                                                                                                                                   |
| **Objective**       | Verify that a `SCHEDULED` auction does not accept bids.                                                                                                                                                               |
| **Prerequisites**   | Auction Scheduled exists and the current time is earlier than `startTime`.                                                                                                                                            |
| **Test Steps**      | 1. Open the auction before it starts. 2. Submit an otherwise valid bid amount. 3. Check the response. 4. Check auction state and time in DynamoDB. 5. Check bid history.                                              |
| **Expected Result** | The request is rejected with `AUCTION_NOT_STARTED`; current price is not updated; highest bidder is not updated; no successful bid history entry is created; the frontend continues to display the not-started state. |
| **Actual Result**   | Record the auction state, start time, server time, and error code.                                                                                                                                                    |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                         |
| **Evidence**        | WebSocket response, DynamoDB, and CloudWatch Logs.                                                                                                                                                                    |

---

### BID-05 — Bid After the Auction Has Ended

| Field               | Content                                                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-05`                                                                                                                                                                                    |
| **Test Name**       | Reject bids submitted after the auction has ended                                                                                                                                           |
| **Objective**       | Verify that an ended auction does not accept additional bids.                                                                                                                               |
| **Prerequisites**   | Auction Ended has status `ENDED` or the current time is later than `endTime`.                                                                                                               |
| **Test Steps**      | 1. Open the ended auction. 2. Submit a bid higher than the current price. 3. Check the response. 4. Check DynamoDB. 5. Check winner and bid history.                                        |
| **Expected Result** | The request is rejected with `AUCTION_ENDED`; current price, winner, and highest bidder remain unchanged; no successful bid history entry is created; no `BID_ACCEPTED` event is broadcast. |
| **Actual Result**   | Record the end time, server time, auction state, and error code.                                                                                                                            |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                               |
| **Evidence**        | Error frame, DynamoDB, and CloudWatch Logs.                                                                                                                                                 |

---

### BID-06 — Invalid or Unauthenticated User

| Field               | Content                                                                                                                                                                                                                          |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-06`                                                                                                                                                                                                                         |
| **Test Name**       | Reject unauthenticated bid requests                                                                                                                                                                                              |
| **Objective**       | Verify that only authenticated users can place bids.                                                                                                                                                                             |
| **Prerequisites**   | The system has a WebSocket authentication mechanism.                                                                                                                                                                             |
| **Test Steps**      | 1. Attempt to bid while not logged in. 2. Attempt with an expired token. 3. Attempt with a token containing an invalid signature. 4. Attempt to send a spoofed `userId` in the message. 5. Check SQS, DynamoDB, and logs.        |
| **Expected Result** | The connection or request is rejected with `UNAUTHENTICATED` or an equivalent code; the client-provided `userId` is not trusted; no valid bid is created; the price is not updated; no valid business message is created in SQS. |
| **Actual Result**   | Record the token type, connection result, or corresponding error code.                                                                                                                                                           |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                    |
| **Evidence**        | Handshake or error frame, no valid bid in SQS, unchanged DynamoDB data, and logs with tokens masked.                                                                                                                             |

---

### BID-07 — Unauthorized User Submits a Bid

| Field               | Content                                                                                                                                                                                                        |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-07`                                                                                                                                                                                                       |
| **Test Name**       | Reject a User who does not have permission to participate in the auction                                                                                                                                       |
| **Objective**       | Verify that User C cannot place a bid in an auction they are not permitted to join.                                                                                                                            |
| **Prerequisites**   | User C is authenticated but does not have membership, registration, or participation permission for Auction Active.                                                                                            |
| **Test Steps**      | 1. Log in as User C. 2. Send a bid request to Auction Active. 3. Check the authorization result. 4. Check SQS and processor logs. 5. Check DynamoDB and broadcast behavior.                                    |
| **Expected Result** | The request is rejected with `NOT_AUCTION_PARTICIPANT` or `FORBIDDEN`; price is not updated; no successful bid history entry is created; highest bidder does not change; no successful bid event is broadcast. |
| **Actual Result**   | Record the masked User ID, Auction ID, and error code.                                                                                                                                                         |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                  |
| **Evidence**        | Error frame, authorization logs, and unchanged DynamoDB data.                                                                                                                                                  |

---

### BID-08 — Resubmit the Same Bid Request

| Field               | Content                                                                                                                                                                                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-08`                                                                                                                                                                                                                                                      |
| **Test Name**       | Handle idempotency and duplicate messages                                                                                                                                                                                                                     |
| **Objective**       | Verify that the same request does not create multiple bids when resent or redelivered by SQS.                                                                                                                                                                 |
| **Prerequisites**   | A `requestId` or `clientMessageId` exists; an idempotency mechanism has been implemented.                                                                                                                                                                     |
| **Test Steps**      | 1. Submit a valid bid with `requestId=bid-request-008`. 2. Wait until the request is processed successfully. 3. Resubmit the exact same request. 4. If possible, simulate SQS redelivery. 5. Check bid history, current price, and broadcast behavior.        |
| **Expected Result** | Only one bid is applied; only one business history record is created; current price is updated only once; the duplicate request receives the previous result or `DUPLICATE_REQUEST`; duplicate business events are not broadcast outside the intended design. |
| **Actual Result**   | Record the number of submissions, processor executions, bid history records, and broadcast events.                                                                                                                                                            |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                 |
| **Evidence**        | Two sent frames, idempotency logs, SQS message attributes, DynamoDB, and broadcast logs.                                                                                                                                                                      |

> The system must not rely only on SQS FIFO deduplication. `la-bid-processor` still requires an idempotency mechanism because a message can be redelivered after the visibility timeout expires or when Lambda successfully processes the message but does not complete batch acknowledgement.

---

### BID-09 — Multiple Users Place Sequential Bids

| Field               | Content                                                                                                                                                                                                                                 |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-09`                                                                                                                                                                                                                                |
| **Test Name**       | Process a sequence of bids from multiple Users                                                                                                                                                                                          |
| **Objective**       | Verify that consecutive valid bids are processed in the correct order.                                                                                                                                                                  |
| **Prerequisites**   | User A and User B are watching the same auction; current price is `1,000,000 VND`; minimum increment is `100,000 VND`.                                                                                                                  |
| **Test Steps**      | 1. User A bids `1,100,000 VND`. 2. Wait for a successful result. 3. User B bids `1,200,000 VND`. 4. User A bids `1,300,000 VND`. 5. Check message ordering in SQS FIFO. 6. Check DynamoDB and broadcast events.                         |
| **Expected Result** | Bids are processed in order within the same Auction/Item message group; prices become `1,100,000`, `1,200,000`, and `1,300,000 VND`; highest bidder changes A → B → A; bid history contains exactly three records in the correct order. |
| **Actual Result**   | Record the sequence, User, amount, timestamp, and result of each bid.                                                                                                                                                                   |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                           |
| **Evidence**        | WebSocket frames from both Users, logs, SQS attributes, and DynamoDB bid history.                                                                                                                                                       |

---

### BID-10 — Highest Bidder Is Updated Correctly

| Field               | Content                                                                                                                                                                                                                             |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-10`                                                                                                                                                                                                                            |
| **Test Name**       | Correctly update the highest bidder                                                                                                                                                                                                 |
| **Objective**       | Verify that the highest bidder always corresponds to the highest valid accepted bid.                                                                                                                                                |
| **Prerequisites**   | Multiple bids have been submitted by User A and User B.                                                                                                                                                                             |
| **Test Steps**      | 1. Record the initial highest bidder. 2. User A submits a valid bid. 3. Check the highest bidder. 4. User B submits a higher bid. 5. Check the highest bidder again. 6. Submit an invalid bid from User A. 7. Check the final data. |
| **Expected Result** | The highest bidder changes only when a new bid is accepted; rejected bids do not change the highest bidder; `highestBidAmount`, `highestBidderId`, and bid history reference the same result.                                       |
| **Actual Result**   | Record the highest bidder and amount after each step.                                                                                                                                                                               |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                       |
| **Evidence**        | DynamoDB before and after, processor logs, and broadcast messages.                                                                                                                                                                  |

---

### BID-11 — Bid History Is Stored Correctly

| Field               | Content                                                                                                                                                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-11`                                                                                                                                                                                                                                                                        |
| **Test Name**       | Store complete and accurate bid history                                                                                                                                                                                                                                         |
| **Objective**       | Verify that every successful bid creates one traceable history record.                                                                                                                                                                                                          |
| **Prerequisites**   | At least three successful bids exist in the same auction.                                                                                                                                                                                                                       |
| **Test Steps**      | 1. Perform multiple valid bids. 2. Query bid history by Auction ID or Item ID. 3. Check User ID, amount, timestamp, Request ID, and sequence. 4. Verify display order. 5. Compare with processor logs and current price.                                                        |
| **Expected Result** | Each accepted bid has exactly one record; amount and User ID are correct; history order is correct; Request ID is not duplicated except for idempotent processing; the latest bid matches the current price and highest bidder; rejected bids do not appear as successful bids. |
| **Actual Result**   | Record the number of submitted bids, accepted bids, and history records.                                                                                                                                                                                                        |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                                   |
| **Evidence**        | DynamoDB query results, CloudWatch Logs, and frontend data.                                                                                                                                                                                                                     |

---

### BID-12 — Broadcast to All Watching Users

| Field               | Content                                                                                                                                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Test Case ID**    | `BID-12`                                                                                                                                                                                                                                   |
| **Test Name**       | Broadcast bid results to the correct room                                                                                                                                                                                                  |
| **Objective**       | Verify that every active connection in the correct room receives the updated bid result.                                                                                                                                                   |
| **Prerequisites**   | User A and User B are watching Room A; User C is watching Room B.                                                                                                                                                                          |
| **Test Steps**      | 1. Open Room A as User A and User B. 2. Open Room B as User C. 3. User A places a valid bid in Room A. 4. Observe messages in all three windows. 5. Check `la-broadcast` logs. 6. Check the queried connection list.                       |
| **Expected Result** | User A and User B receive `BID_ACCEPTED` for Room A; User C does not receive the event; the message contains the correct Auction ID, Item ID, amount, and sequence; failure of one connection does not cause the entire broadcast to fail. |
| **Actual Result**   | Record the messages received or not received by each User.                                                                                                                                                                                 |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                              |
| **Evidence**        | Three browser windows, WebSocket frames, DynamoDB subscriptions, and broadcast logs.                                                                                                                                                       |

---

### BID-13 — Frontend Updates the Price Without Reloading

| Field               | Content                                                                                                                                                                                                                                                                |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Test Case ID**    | `BID-13`                                                                                                                                                                                                                                                               |
| **Test Name**       | Real-time frontend update                                                                                                                                                                                                                                              |
| **Objective**       | Verify that the frontend processes broadcast events and updates displayed data immediately.                                                                                                                                                                            |
| **Prerequisites**   | User A and User B have the same auction page open; WebSocket status is Connected or Live.                                                                                                                                                                              |
| **Test Steps**      | 1. Record the displayed price in both windows. 2. User A submits a valid bid. 3. Do not reload the page. 4. Observe the new price in both windows. 5. Check the highest bidder, bid history, and frontend notifications. 6. Check the browser console and Network tab. |
| **Expected Result** | Both windows display the new price without reloading; highest bidder and bid history are updated according to the design; no duplicate UI entry is created; no JavaScript errors occur; the displayed price matches DynamoDB and the broadcast message.                |
| **Actual Result**   | Record the price before and after, update time, and UI state.                                                                                                                                                                                                          |
| **Status**          | `PASS`, `FAIL`, or `BLOCKED`.                                                                                                                                                                                                                                          |
| **Evidence**        | Video or screenshots before and after, WebSocket frame, browser console, and DynamoDB.                                                                                                                                                                                 |

---

### Data Verification Matrix for Each Request Type

| Scenario                              | Current Price    | Highest Bidder   | Bid History             | Successful Broadcast                 |
| ------------------------------------- | ---------------- | ---------------- | ----------------------- | ------------------------------------ |
| Valid bid                             | Must update      | Must update      | Add exactly one record  | Yes                                  |
| Bid equal to/lower than current price | No change        | No change        | No successful bid added | No                                   |
| Minimum increment not met             | No change        | No change        | No successful bid added | No                                   |
| Auction not started                   | No change        | No change        | No successful bid added | No                                   |
| Auction ended                         | No change        | No change        | No successful bid added | No                                   |
| Unauthenticated User                  | No change        | No change        | No record added         | No                                   |
| Unauthorized User                     | No change        | No change        | No successful bid added | No                                   |
| Duplicate request                     | Update only once | Update only once | Only one record         | No duplicate broadcast beyond design |

---

### SQS FIFO Verification Guidelines

SQS FIFO should be tested according to the following criteria:

* Messages are sent to the correct queue.
* `MessageGroupId` is determined by Auction ID or Item ID.
* Bids for the same item are processed in order.
* A single `MessageGroupId` should not be used for the entire system if it causes unrelated auctions to block each other.
* `MessageDeduplicationId` or content-based deduplication is configured correctly.
* Messages contain a Request ID for tracing and idempotency.
* Messages do not contain access tokens.
* Messages do not trust a User ID supplied by the client.
* User ID in internal messages must come from a verified authentication context.
* Failed messages are retried according to configuration.
* Messages that exceed the allowed number of processing attempts are sent to a DLQ, if configured.
* Messages are not deleted before business processing is completed.
* Partial batch failure handling is implemented if Lambda reads messages in batches.

---

### Concurrent Update Verification Guidelines for DynamoDB

`la-bid-processor` must use a conditional write, transaction, or equivalent concurrency-control mechanism.

At minimum, the following update conditions must be verified:

* The auction is still in `ACTIVE` state.
* Server time is still within the auction period.
* The new bid is higher than the current price.
* The new bid satisfies the minimum increment.
* The version or current price has not already been changed by another request.
* The Request ID has not already been processed.

The system should not be implemented as:

```text
Read current price
→ validate in Lambda
→ perform unconditional update
```

This approach may allow a lower bid to overwrite a higher bid when two Lambda executions run concurrently.

Updating the current price and writing bid history must remain consistent. If one operation succeeds while the other fails, the system should use a transaction or a clearly defined recovery mechanism.

---

### CloudWatch Logs Verification Guidelines

Logs should contain sufficient information for tracing:

* Request ID.
* Lambda Request ID.
* WebSocket route key.
* Auction ID.
* Item ID.
* Verified User ID.
* Bid amount.
* SQS Message ID.
* Message Group ID.
* Receive count.
* Idempotency result.
* Price before and after the update.
* Conditional write result.
* Broadcast event type.
* Number of target connections.
* Number of successful and failed broadcasts.
* Processing time for each component.
* Error code when a bid is rejected.

Logs must not contain:

* Access Token.
* ID Token.
* Refresh Token.
* Authorization header.
* Passwords.
* AWS credentials.
* Complete personal data.
* Stack traces in responses returned to the frontend.

---

### Test Result Summary

| ID       | Test Content                             | Main Expected Result                           | Status     |
| -------- | ---------------------------------------- | ---------------------------------------------- | ---------- |
| `BID-01` | Valid bid                                | Price, highest bidder, and history are updated | Not tested |
| `BID-02` | Bid equal to or lower than current price | Rejected, data remains unchanged               | Not tested |
| `BID-03` | Minimum increment not met                | Rejected with the appropriate error            | Not tested |
| `BID-04` | Auction not started                      | Bid is not accepted                            | Not tested |
| `BID-05` | Auction ended                            | Bid is not accepted                            | Not tested |
| `BID-06` | Invalid User                             | Authentication is rejected                     | Not tested |
| `BID-07` | User not part of auction                 | Authorization is rejected                      | Not tested |
| `BID-08` | Resubmit the same request                | Processed only once                            | Not tested |
| `BID-09` | Multiple Users bid sequentially          | Processed in the correct order                 | Not tested |
| `BID-10` | Update highest bidder                    | Highest bidder is always correct               | Not tested |
| `BID-11` | Store bid history                        | Complete, correct, and non-duplicated history  | Not tested |
| `BID-12` | Broadcast result                         | Correct User and correct Room                  | Not tested |
| `BID-13` | Frontend update                          | New price displayed without reload             | Not tested |

---