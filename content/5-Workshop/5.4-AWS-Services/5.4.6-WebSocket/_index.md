---
title: "API Gateway WebSocket"
date: 2026-07-27
weight: 6
chapter: false
pre: " <b> 5.4.6. </b> "
---

## Overview

The **Live Auction** system uses **Amazon API Gateway WebSocket API** to maintain two-way communication between the frontend and backend.

Unlike a REST API, WebSocket allows the server to proactively send data to the browser without requiring the user to continuously submit new requests. This mechanism is suitable for an online auction system because current prices and auction statuses must be quickly delivered to all users following the auction.

The WebSocket API is created and configured using Terraform in the following module:

```text
infra/07-api
```

The WebSocket API name is:

```text
la-websocket
```

The WebSocket API works with the following Lambda functions:

```text
la-ws-authorizer
la-ws-handler
la-bid-processor
la-broadcast
```

## Role of the WebSocket API

In the Live Auction system, the WebSocket API is used to:

- Establish real-time connections between users and the system.
- Authenticate users when connections are established.
- Track the connection lifecycle.
- Allow users to join auction rooms.
- Receive bid requests.
- Forward bid requests to Lambda for processing.
- Send updated prices to users following an auction.
- Deliver auction and item status updates in real time.
- Detect and remove inactive connections.
- Reduce the need for the frontend to continuously poll the REST API for updated prices.

## WebSocket URL

The WebSocket API is deployed to the following stage:

```text
prod
```

The WebSocket URL follows this structure:

```text
wss://<websocket-api-id>.execute-api.ap-southeast-1.amazonaws.com/prod
```

The `wss://` prefix indicates that the WebSocket connection is encrypted using TLS.

The system also uses a WebSocket Management Endpoint with the following structure:

```text
https://<websocket-api-id>.execute-api.ap-southeast-1.amazonaws.com/prod
```

Lambda functions use this Management Endpoint to proactively send data to WebSocket connections.

## WebSocket Workflow

The general WebSocket workflow is performed as follows:

1. The user signs in through Amazon Cognito.
2. The frontend receives a token after successful authentication.
3. The frontend establishes a connection to the WebSocket URL.
4. The token is included in the `$connect` request.
5. API Gateway invokes `la-ws-authorizer` to validate the token.
6. If the token is valid, API Gateway accepts the connection.
7. API Gateway invokes `la-ws-handler` to process the `$connect` event.
8. The Lambda function stores the Connection ID and related information in DynamoDB.
9. The user sends the `joinRoom` action to join an auction room.
10. The user sends the `placeBid` action to submit a bid.
11. WebSocket Handler validates the request and sends the bid command to SQS FIFO.
12. Bid Processor Lambda processes the message and updates DynamoDB.
13. Broadcast Lambda retrieves the relevant Connection IDs.
14. Broadcast Lambda sends the updated price through the WebSocket Management API.
15. The frontend receives the data and updates the user interface.
16. When the connection is closed, the `$disconnect` route is invoked to remove the Connection ID.

The general flow is:

```text
User Frontend
      ↓
API Gateway WebSocket
      ↓
WebSocket Authorizer
      ↓
WebSocket Handler
      ↓
Amazon SQS FIFO
      ↓
Bid Processor Lambda
      ↓
Amazon DynamoDB
      ↓
Broadcast Lambda
      ↓
API Gateway Management API
      ↓
WebSocket Clients
```

## Route Selection Expression

The WebSocket API uses the following Route Selection Expression:

```text
$request.body.action
```

API Gateway reads the `action` field from the JSON message sent by the frontend and selects the corresponding route.

Example request for joining an auction room:

```json
{
  "action": "joinRoom",
  "sessionId": "<session-id>"
}
```

Example bid request:

```json
{
  "action": "placeBid",
  "itemId": "<item-id>",
  "amount": 1000000
}
```

If the value of `action` is `joinRoom`, API Gateway invokes the `joinRoom` route. If the value is `placeBid`, API Gateway invokes the `placeBid` route.

## WebSocket Routes

The WebSocket API uses four main routes:

| Route           | Authorization     | Role                                                             |
| --------------- | ----------------- | ---------------------------------------------------------------- |
| **$connect**    | Custom Authorizer | Authenticates the user and establishes the WebSocket connection. |
| **$disconnect** | None              | Handles the event when a WebSocket client disconnects.           |
| **joinRoom**    | None              | Registers a connection in an auction room.                       |
| **placeBid**    | None              | Receives bid requests from users.                                |

The route structure is:

```text
la-websocket
├── $connect
├── $disconnect
├── joinRoom
└── placeBid
```

The `$connect` route uses a Custom Authorizer to ensure that only accounts with valid tokens can establish connections.

The remaining routes operate through an authenticated connection that has already been established. Lambda Handler continues to validate the message data and the identity information stored by the system before processing a business operation.

## `$connect` Route

The `$connect` route is invoked when the frontend requests a new WebSocket connection.

The processing flow is:

1. The frontend opens a WebSocket connection.
2. API Gateway reads the token from the query string.
3. API Gateway invokes WebSocket Authorizer.
4. The Authorizer validates the Cognito JWT.
5. If the token is valid, the connection is accepted.
6. WebSocket Handler receives the `$connect` event.
7. The Connection ID is stored in DynamoDB.

The route is configured as follows:

```text
Route key: $connect
Authorization type: CUSTOM
Authorizer: la-ws-authorizer
Integration: la-ws-handler
```

## WebSocket Authorizer

The WebSocket API uses a REQUEST Authorizer:

```text
Authorizer type: REQUEST
Identity source: route.request.querystring.token
Lambda function: la-ws-authorizer
```

The frontend includes the token in the query string when establishing a connection:

```text
wss://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod?token=<cognito-token>
```

The Authorizer validates:

- Whether the token is present.
- The JWT signature.
- Whether the token has expired.
- The Cognito issuer.
- The account identity information.
- Whether the token was issued by the correct Cognito User Pool.

## `$disconnect` Route

The `$disconnect` route is invoked when a WebSocket connection ends.

The route is configured as follows:

```text
Route key: $disconnect
Authorization type: NONE
Integration: la-ws-handler
```

WebSocket Handler uses the Connection ID from the event to:

- Remove connection information from DynamoDB.
- Remove the connection from related auction rooms.
- Prevent messages from being sent to an inactive connection.
- Release connection data that is no longer required.

A `$disconnect` event may occur when:

- The user closes the browser.
- The user leaves the page.
- The network connection is interrupted.
- The connection expires.
- The client intentionally closes the WebSocket connection.

## `joinRoom` Route

The `joinRoom` route allows a WebSocket client to join an auction room.

The route is configured as follows:

```text
Route key: joinRoom
Authorization type: NONE
Integration: la-ws-handler
```

The frontend sends a request with the following structure:

```json
{
  "action": "joinRoom",
  "sessionId": "<session-id>"
}
```

WebSocket Handler associates the Connection ID with the corresponding auction session. Broadcast Lambda later uses this information to send updates to users following that auction.

## `placeBid` Route

The `placeBid` route receives bid requests from users.

The route is configured as follows:

```text
Route key: placeBid
Authorization type: NONE
Integration: la-ws-handler
```

Example request body:

```json
{
  "action": "placeBid",
  "itemId": "<item-id>",
  "amount": 1000000
}
```

WebSocket Handler performs the following operations:

1. Validates the request structure.
2. Validates the connection and account information.
3. Validates the Item ID.
4. Validates the bid amount.
5. Creates a bid message.
6. Sends the message to Amazon SQS FIFO.
7. Returns the request acceptance status to the client.

WebSocket Handler does not directly perform the final bid processing and price update. Instead, the request is sent through SQS FIFO to preserve processing order.

## Lambda Integration

All four WebSocket routes are integrated with:

```text
la-ws-handler
```

The integration is configured as follows:

```text
Integration type: AWS_PROXY
Integration method: POST
```

AWS_PROXY Integration allows API Gateway to forward the complete WebSocket event to Lambda, including:

- Route Key.
- Connection ID.
- Domain Name.
- Stage.
- Request Context.
- Message Body.
- Authorizer information, when available.

WebSocket Authorizer is separately integrated with:

```text
la-ws-authorizer
```

## Connection ID Management

API Gateway assigns a unique Connection ID to every WebSocket connection.

Connection IDs are stored in DynamoDB so that the system can:

- Identify connected clients.
- Determine the user associated with each connection.
- Determine which auction room the client is following.
- Send data to the correct client.
- Remove inactive connections.

A Connection ID remains valid only while its corresponding connection is active.

When Broadcast Lambda sends data to an expired connection, API Gateway may return:

```text
410 Gone
```

When this response occurs, the Lambda function removes the invalid Connection ID from DynamoDB.

## Sending Data to WebSocket Clients

Broadcast Lambda uses the API Gateway Management API to send data to clients.

The primary operation is:

```text
POST @connections/{connection_id}
```

The transmitted data may include:

- Current price.
- Information about the latest bidder.
- Successful or failed bid status.
- Auction item status.
- Remaining time.
- Auction session status.

Example update message:

```json
{
  "type": "bidUpdated",
  "itemId": "<item-id>",
  "currentPrice": 1100000
}
```

The Lambda function requires the following IAM permission:

```text
execute-api:ManageConnections
```

This permission is restricted to the required WebSocket API resources.

## Deployment Stage

The WebSocket API is deployed to the following stage:

```text
prod
```

The stage is configured as follows:

```text
Auto deploy: Enabled
```

When a route or integration configuration is changed, API Gateway automatically updates the stage without requiring a manual deployment.

The stage provides the following endpoints:

```text
WebSocket URL:
wss://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod

Connection URL:
https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod/@connections
```

## Verifying the WebSocket API on AWS Management Console

### Step 1: Access Amazon API Gateway

Sign in to the **AWS Management Console**.

Enter the following service name in the search bar:

```text
API Gateway
```

Select **API Gateway**.

Make sure the selected Region is:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

### Step 2: Verify the WebSocket API

From the API Gateway list, select:

```text
la-websocket
```

On the **Routes** page, verify:

- The API name is `la-websocket`.
- The API uses the WebSocket protocol.
- The Route Selection Expression.
- The configured route list.
- The Authorizer configuration of the `$connect` route.

The system uses the following Route Selection Expression:

```text
$request.body.action
```

The deployed routes are:

```text
$connect
$disconnect
joinRoom
placeBid
```

The `$connect` route uses the `la-ws-authorizer` Lambda Authorizer to authenticate a connection before allowing the client to access the WebSocket API.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-api-overview.png"
    title="Figure 5.4.6.1: Route Selection Expression, routes, and Authorizer of the WebSocket API"
    width="100%"
>}}

### Step 3: Verify the Routes

In the WebSocket API, open:

```text
Routes
```

Verify the following four routes:

```text
$connect
$disconnect
joinRoom
placeBid
```

Confirm the following information:

- Route Key.
- Route authorization configuration.
- Assigned integration.
- The `$connect` route uses a Custom Authorizer.
- The routes are integrated with WebSocket Handler Lambda.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-routes.png"
    title="Figure 5.4.6.2: Routes of the API Gateway WebSocket API"
    width="50%"
>}}

### Step 4: Verify the `$connect` Route

Select:

```text
$connect
```

Verify:

- Authorization is set to Custom.
- An Authorizer is assigned.
- The integration points to a Lambda function.
- The route is deployed to the `prod` stage.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-connect-route.png"
    title="Figure 5.4.6.3: Configuration of the WebSocket API $connect route"
    width="100%"
>}}

### Step 5: Verify WebSocket Authorizer

In the WebSocket API, open:

```text
Authorizers
```

Select the system Authorizer and verify:

- Authorizer Name.
- Authorizer Type is `REQUEST`.
- Lambda Function is `la-ws-authorizer`.
- Identity Source is `route.request.querystring.token`.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-authorizer.png"
    title="Figure 5.4.6.4: Lambda Authorizer of the WebSocket API"
    width="100%"
>}}

### Step 6: Verify Lambda Integration

In the WebSocket API, select each route and open:

```text
Integration request
```

Verify the Lambda Integration associated with the route.

Confirm the following information:

- Integration Type is `Lambda`.
- Lambda Proxy Integration is enabled.
- Lambda Function is `la-ws-handler`.
- Region is `ap-southeast-1`.
- Every route is associated with the correct Lambda Integration.

In the Live Auction system, the following routes use `la-ws-handler` to process WebSocket connections and messages:

```text
$connect
$disconnect
joinRoom
placeBid
```

The following figure shows the Integration Request of the `$connect` route. The remaining routes can be selected individually to confirm that they are also connected to the corresponding Lambda function.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-lambda-integration.png"
    title="Figure 5.4.6.5: Lambda Proxy Integration of the WebSocket API $connect route"
    width="100%"
>}}

### Step 7: Verify the `prod` Stage

In the WebSocket API, open:

```text
Stages → prod
```

Verify:

- WebSocket URL.
- Connection URL.
- Auto Deploy.
- Deployment status.
- Last updated date.
- Stage Name.

The required status is:

```text
Auto deploy: Enabled
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-prod-stage.png"
    title="Figure 5.4.6.6: The prod stage of the API Gateway WebSocket API"
    width="100%"
>}}

### Step 8: Verify the Lambda Trigger

Open the following Lambda function:

```text
la-ws-handler
```

Check:

```text
Function overview
```

or:

```text
Configuration → Triggers
```

Confirm that API Gateway WebSocket is connected to the Lambda function.

Verify:

- API Gateway Trigger.
- API Type is WebSocket.
- Stage is `prod`.
- Lambda Permission allows API Gateway to invoke the function.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-lambda-trigger.png"
    title="Figure 5.4.6.7: API Gateway WebSocket trigger of Lambda Handler"
    width="100%"
>}}

### Step 9: Verify CloudWatch Logs

After the WebSocket API has been used, open:

```text
la-ws-handler
```

Then select:

```text
Monitor → View CloudWatch logs
```

AWS redirects to the following CloudWatch Log Group:

```text
/aws/lambda/la-ws-handler
```

In the Log Group, select the most recently updated Log Stream and inspect its Lambda Log Events.

Verify:

- Lambda Function `la-ws-handler` was invoked.
- The log contains `START`, `END`, and `REPORT` records.
- Lambda completed its execution.
- No `ERROR` record is present.
- No IAM authorization or DynamoDB operation error occurred.
- The `REPORT` record shows execution time and memory usage.
- Tokens, Connection IDs, email addresses, and user bid data are not publicly exposed.

If the application writes business-operation logs, verify the following WebSocket events where available:

```text
$connect
$disconnect
joinRoom
placeBid
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.6-WebSocket/websocket-handler-cloudwatch-log.png"
    title="Figure 5.4.6.8: CloudWatch Logs of Lambda WebSocket Handler"
    width="100%"
>}}

The verification result shows that `la-ws-handler` was invoked and completed its execution. The Log Stream contains the `START`, `END`, and `REPORT` records, while no `ERROR` record is present.

The `REPORT` record provides information about execution duration, billed duration, and memory usage. This confirms that Lambda WebSocket Handler is operational and can receive events forwarded by API Gateway WebSocket.

Because the application does not currently write `routeKey` and `connectionId` as separate CloudWatch log entries, this result is used only to confirm successful Lambda execution. Each route and its Lambda Integration were verified separately through the API Gateway configuration.

## Testing the WebSocket Connection

After verifying the AWS resources, the WebSocket connection can be tested through the frontend or a WebSocket client tool.

Example WebSocket URL:

```text
wss://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod?token=<cognito-token>
```

Expected results:

- Valid token: The connection is established.
- Missing token: The connection is rejected.
- Invalid token: The connection is rejected.
- `joinRoom`: The connection is associated with the auction session.
- `placeBid`: The bid request is accepted and sent to the queue.
- After a successful bid: The frontend receives the updated price.
- When the connection is closed: The Connection ID is removed from DynamoDB.

Complete WebSocket functional testing is presented in **Section 5.5 — System Testing**.

## Result

After inspecting the resources through the AWS Management Console, the team confirmed that:

- WebSocket API `la-websocket` was successfully created using Terraform.
- The Route Selection Expression is `$request.body.action`.
- The `$connect`, `$disconnect`, `joinRoom`, and `placeBid` routes were created.
- The `$connect` route uses a Custom Lambda Authorizer.
- The Authorizer validates the Cognito token from the query string.
- The routes are integrated with `la-ws-handler`.
- The `prod` stage was deployed with Auto Deploy enabled.
- WebSocket Handler can manage Connection IDs and auction rooms.
- Bid requests are forwarded from WebSocket to SQS FIFO.
- Bid Processor processes messages in order.
- Broadcast Lambda can send updated prices to connected WebSocket clients.
- Lambda execution information is recorded in Amazon CloudWatch.
- The WebSocket API is ready to support real-time auction functionality.

The deployment and verification of the system data tables are presented in **Section 5.4.7 — Amazon DynamoDB**.