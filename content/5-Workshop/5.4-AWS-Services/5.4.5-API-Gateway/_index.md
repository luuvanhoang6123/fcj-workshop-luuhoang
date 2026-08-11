---
title: "Amazon API Gateway"
date: 2026-07-27
weight: 5
chapter: false
pre: " <b> 5.4.5. </b> "
---

## Overview

The **Live Auction** system uses an **Amazon API Gateway REST API** to receive application requests from the User Frontend and Admin Frontend.

API Gateway acts as the communication entry point between the frontend applications and the backend Lambda functions. When a frontend sends a request, API Gateway validates authentication information, applies access control policies, and forwards the request to the corresponding Lambda function.

The REST API is created and configured using Terraform in the following module:

```text
infra/07-api
```

The REST API name follows this structure:

```text
<name-prefix>-control-plane
```

With the current resource prefix, the API is named:

```text
la-control-plane
```

## Role of Amazon API Gateway

In the Live Auction system, Amazon API Gateway is used to:

* Provide REST APIs for the User Frontend.
* Provide REST APIs for the Admin Frontend.
* Route requests to the corresponding Lambda functions.
* Validate Cognito JWTs sent by authenticated accounts.
* Require an API Key for the configured API methods.
* Apply request throttling.
* Apply a quota to the API Key.
* Configure CORS for the allowed frontend origins.
* Write Access Logs to Amazon CloudWatch.
* Collect monitoring Metrics.
* Cache selected query APIs.
* Return consistent error responses to the frontend applications.

## REST API request flow

The REST API request flow operates as follows:

1. The User Frontend or Admin Frontend sends an HTTPS request to API Gateway.
2. The request includes a Cognito token in the `Authorization` header.
3. The request includes an API Key in the `x-api-key` header.
4. API Gateway validates the Cognito token through a Cognito User Pool Authorizer.
5. API Gateway validates the API Key and Usage Plan.
6. API Gateway validates the Method, Resource, and request parameters.
7. The request is forwarded to a Lambda function through Lambda Proxy Integration.
8. The Lambda processes the operation and accesses DynamoDB or other related services.
9. The Lambda returns the result to API Gateway.
10. API Gateway returns an HTTP response to the frontend.

The general flow is:

```text
User/Admin Frontend
        ↓
Amazon API Gateway REST API
        ↓
Cognito Authorizer + API Key
        ↓
Lambda Proxy Integration
        ↓
AWS Lambda
        ↓
Amazon DynamoDB and related services
```

## REST API structure

The REST API uses the following base path:

```text
/api/v1
```

The APIs are divided into the following primary operation groups:

| API group               | Purpose                                                                 |
| ----------------------- | ----------------------------------------------------------------------- |
| **User API**            | Retrieves information about the current account.                        |
| **Auction Session API** | Creates, retrieves, and manages auction sessions.                       |
| **Auction Item API**    | Retrieves, adds, and manages auction items.                             |
| **Bid API**             | Retrieves bid history or bid data associated with a user.               |
| **Category API**        | Retrieves product categories.                                           |
| **Admin API**           | Manages users, Admin accounts, categories, items, and auction sessions. |

## User APIs

### Account information

```text
GET /api/v1/users/me
```

This API returns information about the current account based on the Cognito token.

### Auction sessions

```text
GET  /api/v1/auction-sessions
POST /api/v1/auction-sessions
GET  /api/v1/auction-sessions/mine
GET  /api/v1/auction-sessions/{session_id}
PUT  /api/v1/auction-sessions/{session_id}/rules
POST /api/v1/auction-sessions/{session_id}/items
POST /api/v1/auction-sessions/{session_id}/schedule
```

These APIs support:

* Retrieving auction sessions.
* Creating an auction session.
* Retrieving sessions created by the current user.
* Retrieving auction session details.
* Updating auction session rules.
* Adding one or more items to a session.
* Configuring the auction session schedule.

### Auction items

```text
GET  /api/v1/auction-items
GET  /api/v1/auction-items/{item_id}
POST /api/v1/auction-items/{item_id}/images/presign
```

These APIs support:

* Retrieving auction items.
* Retrieving auction item details.
* Creating the information required to upload item images to Amazon S3.

### Bid history

```text
GET /api/v1/bids/my
```

This API retrieves bid data associated with the current user.

### Product categories

```text
GET /api/v1/categories
GET /api/v1/categories/{category_id}
```

These APIs allow the frontend applications to retrieve the category list and category details.

## Administrator APIs

### Auction session management

```text
GET  /api/v1/admin/auction-sessions
GET  /api/v1/admin/auction-sessions/{session_id}
POST /api/v1/admin/auction-sessions/{session_id}/approve
POST /api/v1/admin/auction-sessions/{session_id}/reject
POST /api/v1/admin/auction-sessions/{session_id}/cancel
POST /api/v1/admin/auction-sessions/{session_id}/close
```

Administrators can:

* Retrieve auction sessions requiring management.
* View auction session details.
* Approve an auction session.
* Reject an auction session.
* Cancel an auction session.
* Close an auction session.

### Auction item management

```text
POST /api/v1/admin/items/{item_id}/pause
POST /api/v1/admin/items/{item_id}/resume
POST /api/v1/admin/items/{item_id}/approve
POST /api/v1/admin/items/{item_id}/close
POST /api/v1/admin/items/{item_id}/cancel
```

These APIs are used to manage item states during an auction.

### User account management

```text
GET   /api/v1/admin/users
GET   /api/v1/admin/users/{user_id}
PATCH /api/v1/admin/users/{user_id}/status
```

Administrators can:

* Retrieve user accounts.
* View user account details.
* Change the status of a user account.

### Admin account management

```text
GET   /api/v1/admin/admin-accounts
POST  /api/v1/admin/admin-accounts
PATCH /api/v1/admin/admin-accounts/{user_id}/status
POST  /api/v1/admin/admin-accounts/{user_id}/reset-invitation
```

These APIs support:

* Retrieving Admin accounts.
* Creating an additional Admin account.
* Changing the status of an Admin account.
* Resending an account activation invitation.

### Category management

```text
GET   /api/v1/admin/categories
POST  /api/v1/admin/categories
PATCH /api/v1/admin/categories/{category_id}
POST  /api/v1/admin/categories/{category_id}/archive
```

Administrators can:

* Retrieve product categories.
* Create a category.
* Update a category.
* Archive a category that is no longer used.

### Dashboard and audit events

```text
GET /api/v1/admin/dashboard
GET /api/v1/admin/audit-events
```

These APIs provide overview data and administrative operation history.

## Lambda Integration

The REST API uses Lambda Proxy Integration:

```text
Integration type: AWS_PROXY
Integration HTTP method: POST
```

Each API group is integrated with the corresponding Lambda function:

| Lambda Function        | API group                                                          |
| ---------------------- | ------------------------------------------------------------------ |
| **la-session-service** | Session creation, rule updates, and session management operations. |
| **la-item-service**    | Item creation and image upload operations.                         |
| **la-query-service**   | User, session, item, category, and bid query APIs.                 |
| **la-admin-command**   | Administrative APIs for sessions, accounts, items, and categories. |

API Gateway uses Lambda Permissions to invoke the corresponding functions.

## Cognito User Pool Authorizer

The REST API uses a Cognito User Pool Authorizer to validate the token in the following header:

```text
Authorization
```

The authentication process operates as follows:

1. A user signs in through Amazon Cognito.
2. Cognito returns a token to the frontend.
3. The frontend sends the token in the `Authorization` header.
4. API Gateway validates the token against the Cognito User Pool.
5. If the token is valid, the request is forwarded to a Lambda function.
6. If the token is invalid or expired, API Gateway rejects the request.

The Cognito Authorizer verifies the identity of the account. For administrative APIs, the Lambda function also verifies that the account belongs to the `admin` group before processing the operation.

## API Key and Usage Plan

In addition to a Cognito token, the REST API requires an API Key in the following header:

```text
x-api-key
```

The API Key is connected to a Usage Plan.

The default Usage Plan configuration is:

```text
Rate limit: 50 requests/second
Burst limit: 100 requests
Daily quota: 10,000 requests
```

The Usage Plan helps to:

* Limit the number of requests sent to the API.
* Prevent excessive API calls.
* Reduce the risk of endpoint abuse.
* Track usage associated with the API Key.
* Protect Lambda functions and backend services.

## CORS configuration

The REST API configures CORS to allow the User Frontend and Admin Frontend to send requests to API Gateway.

The allowed headers include:

```text
Content-Type
Authorization
X-Api-Key
```

The allowed methods include:

```text
GET
POST
PUT
PATCH
OPTIONS
```

The `OPTIONS` method processes CORS Preflight Requests before a browser sends the main request.

CORS allows only the frontend origins configured for the system.

## Deployment Stage

The REST API is deployed to the following Stage:

```text
prod
```

The Invoke URL has the following structure:

```text
https://<rest-api-id>.execute-api.ap-southeast-1.amazonaws.com/prod
```

The `prod` Stage is configured with:

* Access Logging.
* CloudWatch Metrics.
* `INFO` Logging Level.
* Throttling.
* API Cache.
* Cache Data Encryption.
* Usage Plan and API Key.

## API Cache

The `prod` Stage enables API Gateway Cache with the following default size:

```text
0.5 GB
```

Caching is enabled for selected query APIs:

```text
GET /api/v1/auction-sessions
GET /api/v1/auction-items
```

The default cache duration is:

```text
60 seconds
```

Cache Keys can include:

* `status`
* `pageSize`
* `cursor`
* `sessionId`
* `categoryId`

Using the cache reduces Lambda invocations for frequently performed queries.

## Access Logging and Metrics

API Gateway writes Access Logs to Amazon CloudWatch.

The logs can contain:

* Request ID.
* Source IP.
* HTTP Method.
* Resource Path.
* Status Code.
* Response Length.
* Response Latency.
* Integration Latency.
* Cognito Subject.

CloudWatch Metrics support the monitoring of:

* Count.
* Latency.
* Integration Latency.
* `4XX` errors.
* `5XX` errors.
* Cache Hit.
* Cache Miss.

Data Trace should not be enabled in an environment containing real data if the logs may include tokens or sensitive request content.

## Gateway Responses

The REST API configures Gateway Responses for:

```text
DEFAULT_4XX
DEFAULT_5XX
```

CORS Headers are also included in error responses so that the frontend applications can read the error content.

This configuration provides consistent error responses across the APIs.

## Verifying the REST API on the AWS Management Console

### Step 1: Access Amazon API Gateway

Sign in to the **AWS Management Console**.

Enter the following value in the search bar:

```text
API Gateway
```

Select **API Gateway**.

Ensure that the selected Region is:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

### Step 2: Verify the API list

In the API Gateway interface, open:

```text
APIs
```

Find the following REST API:

```text
la-control-plane
```

Verify:

* API Name.
* API ID.
* API Type is REST.
* Endpoint Type.
* Created Date.
* AWS Region.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-api-list.png"
    title="Figure 5.4.5.1: APIs of the Live Auction system on Amazon API Gateway"
    width="100%"
>}}

### Step 3: Verify Resources and Methods

Select the following REST API:

```text
la-control-plane
```

Open:

```text
Resources
```

Inspect the API Resource tree.

The following information should be confirmed:

* Resource Paths begin with `/api/v1`.
* The `auction-sessions`, `auction-items`, `users`, `categories`, and `admin` groups exist.
* HTTP Methods include `GET`, `POST`, `PUT`, `PATCH`, and `OPTIONS`.
* Resources contain Path Parameters such as `{session_id}`, `{item_id}`, `{user_id}`, and `{category_id}`.
* Resources include the `OPTIONS` method for processing CORS Preflight Requests.
* Administrative Resources are placed under `/api/v1/admin`.

The complete deployed REST API structure is shown below:

```text
/
└── api
    └── v1
        ├── users
        │   └── me
        │       ├── GET
        │       └── OPTIONS
        │
        ├── auction-sessions
        │   ├── GET
        │   ├── POST
        │   ├── OPTIONS
        │   ├── mine
        │   │   ├── GET
        │   │   └── OPTIONS
        │   └── {session_id}
        │       ├── GET
        │       ├── OPTIONS
        │       ├── items
        │       │   ├── POST
        │       │   └── OPTIONS
        │       ├── rules
        │       │   ├── PUT
        │       │   └── OPTIONS
        │       └── schedule
        │           ├── POST
        │           └── OPTIONS
        │
        ├── auction-items
        │   ├── GET
        │   ├── OPTIONS
        │   └── {item_id}
        │       ├── GET
        │       ├── OPTIONS
        │       └── images
        │           └── presign
        │               ├── POST
        │               └── OPTIONS
        │
        ├── bids
        │   └── my
        │       ├── GET
        │       └── OPTIONS
        │
        ├── categories
        │   ├── GET
        │   ├── OPTIONS
        │   └── {category_id}
        │       ├── GET
        │       └── OPTIONS
        │
        └── admin
            ├── dashboard
            │   ├── GET
            │   └── OPTIONS
            │
            ├── audit-events
            │   ├── GET
            │   └── OPTIONS
            │
            ├── users
            │   ├── GET
            │   ├── OPTIONS
            │   └── {user_id}
            │       ├── GET
            │       ├── OPTIONS
            │       └── status
            │           ├── PATCH
            │           └── OPTIONS
            │
            ├── admin-accounts
            │   ├── GET
            │   ├── POST
            │   ├── OPTIONS
            │   └── {user_id}
            │       ├── status
            │       │   ├── PATCH
            │       │   └── OPTIONS
            │       └── reset-invitation
            │           ├── POST
            │           └── OPTIONS
            │
            ├── categories
            │   ├── GET
            │   ├── POST
            │   ├── OPTIONS
            │   └── {category_id}
            │       ├── PATCH
            │       ├── OPTIONS
            │       └── archive
            │           ├── POST
            │           └── OPTIONS
            │
            ├── auction-sessions
            │   ├── GET
            │   ├── OPTIONS
            │   └── {session_id}
            │       ├── GET
            │       ├── OPTIONS
            │       ├── approve
            │       │   ├── POST
            │       │   └── OPTIONS
            │       ├── reject
            │       │   ├── POST
            │       │   └── OPTIONS
            │       ├── cancel
            │       │   ├── POST
            │       │   └── OPTIONS
            │       └── close
            │           ├── POST
            │           └── OPTIONS
            │
            └── items
                └── {item_id}
                    ├── approve
                    │   ├── POST
                    │   └── OPTIONS
                    ├── cancel
                    │   ├── POST
                    │   └── OPTIONS
                    ├── close
                    │   ├── POST
                    │   └── OPTIONS
                    ├── pause
                    │   ├── POST
                    │   └── OPTIONS
                    └── resume
                        ├── POST
                        └── OPTIONS
```

The Resource tree shows that the REST API is divided into two primary groups:

* **User API:** Provides account information, auction session creation and retrieval, item management, bid data, and product categories.
* **Admin API:** Provides user account management, Admin account management, category management, auction session management, item management, and Audit Events.

Because the Resource tree on the AWS Management Console contains many paths and cannot be displayed completely on one screen, the following figure shows only part of the deployed structure. The complete list is presented in the Resource tree above.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-resources-methods.png"
    title="Figure 5.4.5.2: Part of the REST API Resources and HTTP Methods on the AWS Management Console"
    width="50%"
>}}

### Step 4: Verify Method Execution

Select a Method, such as:

```text
GET /api/v1/auction-sessions
```

Verify:

* Method Request.
* Integration Request.
* Method Response.
* Integration Response.
* Authorization.
* API Key Required.
* Integrated Lambda Function.

The following values should be confirmed:

```text
Authorization: Cognito User Pool Authorizer
API Key Required: True
Integration type: Lambda Function
Lambda Proxy Integration: Enabled
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-method-execution.png"
    title="Figure 5.4.5.3: Method and Lambda Integration configuration of the REST API"
    width="100%"
>}}

### Step 5: Verify the Cognito Authorizer

From the REST API navigation menu, select:

```text
Authorizers
```

Verify the Cognito Authorizer.

The following information should be confirmed:

* Authorizer Type is Cognito.
* The Cognito User Pool is connected.
* Token Source is `Authorization`.
* AWS Region of the User Pool.
* The Authorizer is used by the API Methods.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-cognito-authorizer.png"
    title="Figure 5.4.5.4: Cognito User Pool Authorizer of the REST API"
    width="100%"
>}}

### Step 6: Verify the prod Stage

From the REST API navigation menu, select:

```text
Stages → prod
```

Verify:

* Invoke URL.
* Deployment ID.
* Last Updated Date.
* Cache Cluster.
* Logging.
* CloudWatch Metrics.
* Throttling.
* Method Settings.

The Stage should not be modified directly through the AWS Console because it is managed by Terraform.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-prod-stage.png"
    title="Figure 5.4.5.5: prod Stage of the Amazon API Gateway REST API"
    width="100%"
>}}

### Step 7: Verify the API Key

Return to the API Gateway interface and open:

```text
API keys
```

Find the project's API Key.

Verify:

* API Key Name.
* Enabled state.
* Created Date.
* Connected Usage Plan.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-api-key.png"
    title="Figure 5.4.5.6: API Key configured for the REST API"
    width="100%"
>}}

### Step 8: Verify the Usage Plan

In API Gateway, open:

```text
Usage plans
```

Select the system's Usage Plan and verify:

* Rate Limit.
* Burst Limit.
* Quota.
* Connected API Stage.
* Connected API Key.

The default project values are:

```text
Rate limit: 50 requests/second
Burst limit: 100 requests
Quota: 10,000 requests/day
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-usage-plan.png"
    title="Figure 5.4.5.7: Throttling and Quota of the API Gateway Usage Plan"
    width="100%"
>}}

### Step 9: Verify CloudWatch Logs and Metrics

In the REST API, select:

```text
Dashboard
```

Select the following Stage:

```text
prod
```

Next, select a date range containing operational data and inspect the following Metrics:

* Number of API Calls.
* Latency.
* Integration Latency.
* `4XX` errors.
* `5XX` errors, if present.
* Cache Hit and Cache Miss, if data is available.

The Dashboard confirms that the REST API received actual requests during system operation. The Latency and Integration Latency graphs help monitor the response time of API Gateway and the Lambda Integration.

The `4XX error` graph records requests rejected because of client-side errors, such as a missing Cognito token, missing API Key, invalid path, or insufficient account permissions.

The Access Logs of the `prod` Stage are stored in the CloudWatch Log Group configured in the **Logs and tracing** section of the Stage.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.5-API-Gateway/api-gateway-cloudwatch-metrics.png"
    title="Figure 5.4.5.8: API Call, Latency, and Error Metrics of the REST API"
    width="100%"
>}}

## Verifying API responses

After verifying the configuration on the AWS Console, a test request can be sent to the Invoke URL.

A valid request requires:

```text
Authorization: <cognito-token>
x-api-key: <api-key>
Content-Type: application/json
```

The expected results include:

* Missing Cognito token: the API returns an authentication error.
* Invalid token: the API rejects the request.
* Missing API Key: the API rejects the request.
* Valid token and API Key: the request is forwarded to a Lambda function.
* A User account calls an Admin API: the Lambda rejects the request because the account does not have the required permission.
* A valid Admin account calls an Admin API: the administrative request is processed.

Actual tokens and API Keys must not be included in Hugo source files or report screenshots.

Complete API testing is presented in **Section 5.5 — System Testing**.

## Results

After verifying the resources directly through the AWS Management Console, the team confirmed that:

* The `la-control-plane` REST API was successfully created by Terraform.
* Resources and HTTP Methods were configured under the `/api/v1` path.
* User and Admin APIs were organized into separate groups.
* Methods were integrated with the corresponding Lambda functions.
* The Cognito User Pool Authorizer was configured.
* API Methods required a Cognito token and API Key.
* The `prod` Stage was deployed.
* CORS allowed the two frontend applications to send requests to the API.
* API Cache was enabled for the required query APIs.
* Access Logging and CloudWatch Metrics were configured.
* Throttling and Daily Quota were applied through the Usage Plan.
* Gateway Responses were configured for `4XX` and `5XX` errors.
* The REST API was ready to receive application requests from the User Frontend and Admin Frontend.

The configuration and verification of the API Gateway WebSocket API are presented in **Section 5.4.6 — API Gateway WebSocket**.