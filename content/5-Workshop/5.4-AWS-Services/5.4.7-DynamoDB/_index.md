---
title: "Amazon DynamoDB"
date: 2026-07-27
weight: 7
chapter: false
pre: " <b> 5.4.7. </b> "
---

## Overview

The **Live Auction** system uses **Amazon DynamoDB** to store business data, auction states, bid history, WebSocket connections, product categories, and administrator audit events.

DynamoDB is a serverless NoSQL database service provided by AWS. It supports automatic scaling and is suitable for systems that require fast, real-time data access.

The DynamoDB tables are created and configured with Terraform in:

```text
infra/04-data
```

The system uses the following billing mode:

```text
PAY_PER_REQUEST
```

With this mode, the team does not need to provision Read Capacity Units or Write Capacity Units in advance. DynamoDB charges based on actual read and write requests.

## Role of Amazon DynamoDB

In the Live Auction system, DynamoDB is used to:

- Store the current state of auction items.
- Store bid events and bid history.
- Store WebSocket client Connection IDs.
- Associate users with bidder aliases for each auction item.
- Prevent duplicate request processing.
- Store auction sessions and auction items.
- Store product categories.
- Store administrator audit events.
- Support additional query patterns through Global Secondary Indexes.
- Automatically remove temporary data through Time to Live.
- Support data recovery through Point-in-Time Recovery.
- Provide DynamoDB Streams for tables that require change-event processing.

## DynamoDB Tables

The system uses eight primary DynamoDB tables:

| DynamoDB Table | Partition Key | Sort Key | Role |
| --- | --- | --- | --- |
| **la_item_auction_state** | `item_id` | None | Stores the current state of each auction item. |
| **la_bid_events** | `item_id` | `sk` | Stores bid events and bid history. |
| **la_websocket_connections** | `item_id` | `connection_id` | Stores connections of users following auction items. |
| **la_item_bidder_aliases** | `item_id` | `user_id` | Stores bidder aliases for each auction item. |
| **la_idempotency** | `id` | None | Prevents the same request from being processed multiple times. |
| **la_auction_catalog** | `pk` | `sk` | Stores auction sessions, items, and related entities. |
| **la_category_catalog** | `category_id` | None | Stores product categories. |
| **la_admin_audit_events** | `pk` | `sk` | Stores administrator activity history. |

Table names follow this convention:

```text
<name-prefix>_<table-purpose>
```

## Item Auction State Table

The following table stores the current state of each auction item:

```text
la_item_auction_state
```

Primary key configuration:

```text
Partition key: item_id
Attribute type: String
```

The stored data may include:

- Item ID.
- Current price.
- Latest bidder.
- Item status.
- Last updated time.
- Version or update-control information.
- Start and end times.

DynamoDB Stream is enabled with:

```text
Stream: Enabled
View type: NEW_AND_OLD_IMAGES
```

`NEW_AND_OLD_IMAGES` allows each Stream event to contain the item data from both before and after the change.

The table also uses:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## Bid Events Table

The following table stores bid events for each auction item:

```text
la_bid_events
```

Primary key configuration:

```text
Partition key: item_id
Sort key: sk
```

The `sk` Sort Key arranges events belonging to the same auction item according to the ordering model designed for the system.

The stored data may include:

- Item ID.
- Bidder subject.
- Bid amount.
- Bid timestamp.
- Processing status.
- Event ID.
- Idempotency information.

DynamoDB Stream is enabled with:

```text
Stream: Enabled
View type: NEW_IMAGE
```

`NEW_IMAGE` includes the new item data after an event is written.

The table has the following Global Secondary Index:

```text
Index name: bidder_sub-sk-index
Partition key: bidder_sub
Sort key: sk
Projection: ALL
```

This index supports querying bid events by user.

The table also uses:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## WebSocket Connections Table

The following table stores information about WebSocket clients following auction items:

```text
la_websocket_connections
```

Primary key configuration:

```text
Partition key: item_id
Sort key: connection_id
```

This key structure allows the system to retrieve all Connection IDs following a particular item.

The stored data may include:

- Item ID.
- Connection ID.
- User ID or Cognito Subject.
- Connection time.
- Expiration time.
- Auction-room information.

Time to Live is enabled:

```text
TTL attribute: ttl
TTL status: Enabled
```

TTL enables DynamoDB to automatically remove expired connection records.

When API Gateway returns `410 Gone`, Lambda may also actively remove the invalid Connection ID.

## Item Bidder Aliases Table

The following table stores bidder aliases for each auction item:

```text
la_item_bidder_aliases
```

Primary key configuration:

```text
Partition key: item_id
Sort key: user_id
```

The table helps the system:

- Associate users with auction items.
- Display aliases instead of actual account information.
- Protect bidder identities.
- Maintain a consistent alias for a user within the same item.

Server-side encryption is enabled for this table.

## Idempotency Table

The following table prevents the same request from being processed multiple times:

```text
la_idempotency
```

Primary key configuration:

```text
Partition key: id
```

For example, when a bid request is resent because of a network error, the system can check its Idempotency ID before processing it again.

TTL is enabled:

```text
TTL attribute: expiration
TTL status: Enabled
```

Expired idempotency records are automatically removed by DynamoDB.

This mechanism helps:

- Prevent duplicate data writes.
- Prevent the same message from being processed multiple times.
- Improve safety when Lambda retries an operation.
- Support message processing from SQS FIFO.

## Auction Catalog Table

The following table stores auction sessions, auction items, and related business entities:

```text
la_auction_catalog
```

Primary key configuration:

```text
Partition key: pk
Sort key: sk
```

The table uses a composite-key design to store multiple entity types in a single table.

It has two Global Secondary Indexes:

```text
Index name: gsi1
Partition key: gsi1pk
Sort key: gsi1sk
Projection: ALL
```

```text
Index name: gsi2
Partition key: gsi2pk
Sort key: gsi2sk
Projection: ALL
```

These indexes support additional query patterns, such as:

- Querying auction sessions by status.
- Querying sessions created by a user.
- Querying items belonging to a session.
- Sorting data by time.
- Querying entities according to business relationships.

The table also uses:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## Category Catalog Table

The following table stores product categories used when creating auction items:

```text
la_category_catalog
```

Primary key configuration:

```text
Partition key: category_id
```

The table has two Global Secondary Indexes.

Index for querying by slug:

```text
Index name: slug-index
Partition key: slug
Projection: ALL
```

Index for querying by status:

```text
Index name: status-index
Partition key: status
Sort key: created_at
Projection: ALL
```

These indexes support:

- Finding a category by slug.
- Retrieving categories by status.
- Sorting categories by creation time.
- Paginating category results.

The table also uses:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## Admin Audit Events Table

The following table stores administrator activity history:

```text
la_admin_audit_events
```

Primary key configuration:

```text
Partition key: pk
Sort key: sk
```

An audit event may include:

- The administrator who performed the action.
- Action type.
- Affected resource.
- Operation result.
- Execution time.
- Request ID.
- Related descriptive data.

The table has two Global Secondary Indexes.

Index for querying by actor:

```text
Index name: actor-index
Partition key: actor_sub
Sort key: sk
Projection: ALL
```

Index for querying by resource:

```text
Index name: resource-index
Partition key: resource_key
Sort key: sk
Projection: ALL
```

TTL is enabled:

```text
TTL attribute: expires_at
TTL status: Enabled
```

The table also uses:

```text
Server-side encryption: Enabled
Point-in-Time Recovery: Enabled
```

## Global Secondary Indexes

A Global Secondary Index, or GSI, allows data to be queried using keys other than the table’s primary key.

The main indexes used by the system are:

| Table | Index |
| --- | --- |
| **la_bid_events** | `bidder_sub-sk-index` |
| **la_auction_catalog** | `gsi1`, `gsi2` |
| **la_category_catalog** | `slug-index`, `status-index` |
| **la_admin_audit_events** | `actor-index`, `resource-index` |

These indexes use:

```text
Projection type: ALL
```

This setting copies all item attributes into the index to support query operations.

## Time to Live

Time to Live, or TTL, allows DynamoDB to automatically remove an item after its expiration timestamp has been reached.

The following tables use TTL:

| Table | TTL Attribute | Purpose |
| --- | --- | --- |
| **la_websocket_connections** | `ttl` | Removes expired WebSocket connections. |
| **la_idempotency** | `expiration` | Removes duplicate-prevention records that are no longer required. |
| **la_admin_audit_events** | `expires_at` | Removes audit events after their retention period. |

TTL prevents obsolete records from remaining indefinitely and reduces the need for manual cleanup.

TTL deletion does not occur immediately at the exact expiration time. DynamoDB removes expired items asynchronously afterward.

## Point-in-Time Recovery

Point-in-Time Recovery, or PITR, allows a DynamoDB table to be restored to a supported point in time.

PITR is enabled for the following important tables:

```text
la_item_auction_state
la_bid_events
la_auction_catalog
la_category_catalog
la_admin_audit_events
```

This configuration supports:

- Recovery after accidental deletion.
- Recovery after incorrect data writes.
- Reducing the impact of application data errors.
- Improving the system’s recovery capability.

## Server-Side Encryption

Server-side encryption is enabled for the DynamoDB tables.

DynamoDB automatically encrypts:

- Table data.
- Indexes.
- DynamoDB Streams.
- Backups.

Encryption protects data stored on AWS.

## DynamoDB Streams

DynamoDB Streams are enabled for two tables:

| Table | Stream View Type |
| --- | --- |
| **la_item_auction_state** | `NEW_AND_OLD_IMAGES` |
| **la_bid_events** | `NEW_IMAGE` |

A Stream records information about changes made to items in a table.

Stream data can be used to:

- Detect auction-price changes.
- Trigger asynchronous processing.
- Send updates to WebSocket clients.
- Track change history.
- Support additional event-driven functions.

## Verifying DynamoDB on AWS Management Console

### Step 1: Access Amazon DynamoDB

Sign in to the **AWS Management Console**.

Enter the following service name in the search bar:

```text
DynamoDB
```

Select **DynamoDB**.

Make sure the selected Region is:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

### Step 2: Verify the Table List

From the navigation menu, select:

```text
Tables
```

Enter the following prefix in the search box:

```text
la_
```

Verify the eight tables used by the system.

Confirm:

- Table Name.
- Status.
- Partition Key.
- Sort Key.
- Billing Mode.
- Region.
- All required tables have been deployed.

Every table should have the following status:

```text
Active
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-table-list.png"
    title="Figure 5.4.7.1: DynamoDB tables of the Live Auction system"
    width="100%"
>}}

### Step 3: Verify Table Information

Select:

```text
la_auction_catalog
```

Open:

```text
Settings → General information
```

Verify:

- Table Status.
- Partition Key.
- Sort Key.
- Billing Mode.
- Table ARN.
- Point-in-Time Recovery.
- Encryption.
- Item Count.
- Table Size.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-table-overview.png"
    title="Figure 5.4.7.2: Information about the la_auction_catalog table"
    width="100%"
>}}

### Step 4: Inspect Table Data

In the `la_auction_catalog` table, select:

```text
Explore items
```

Select **Run** to display part of the stored data.

Verify:

- Partition Key `pk`.
- Sort Key `sk`.
- Data attributes.
- Items representing auction sessions or auction items.
- Data written to the table by Lambda.

Do not directly edit or delete items through the AWS Console.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-auction-catalog-items.png"
    title="Figure 5.4.7.3: Auction session and item data in la_auction_catalog"
    width="100%"
>}}

### Step 5: Verify Global Secondary Indexes

In the `la_auction_catalog` table, open:

```text
Indexes
```

Verify:

```text
gsi1
gsi2
```

Confirm:

- Index Name.
- Partition Key.
- Sort Key.
- Index Status.
- Projection Type.
- Item Count.
- Index Size.

The required status is:

```text
Active
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-global-secondary-indexes.png"
    title="Figure 5.4.7.4: Global Secondary Indexes of la_auction_catalog"
    width="100%"
>}}

### Step 6: Verify Time to Live

Select:

```text
la_websocket_connections
```

Open:

```text
Additional settings
```

or locate:

```text
Time to Live (TTL)
```

Verify:

```text
TTL status: On
TTL attribute: ttl
```

The `la_idempotency` table can also be checked for the `expiration` TTL attribute.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-websocket-ttl.png"
    title="Figure 5.4.7.5: TTL configuration of the WebSocket Connections table"
    width="100%"
>}}

### Step 7: Verify Point-in-Time Recovery

Select an important table, such as:

```text
la_auction_catalog
```

Open:

```text
Backups
```

Locate:

```text
Point-in-time recovery
```

Confirm the following status:

```text
Point-in-time recovery: On
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-point-in-time-recovery.png"
    title="Figure 5.4.7.6: Point-in-Time Recovery of the DynamoDB table"
    width="100%"
>}}

### Step 8: Verify DynamoDB Stream

Select:

```text
la_item_auction_state
```

Open:

```text
Exports and streams
```

Verify the DynamoDB Stream configuration:

```text
DynamoDB stream: On
View type: New and old images
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-stream.png"
    title="Figure 5.4.7.7: DynamoDB Stream of the auction-state table"
    width="100%"
>}}

### Step 9: Verify Monitoring Metrics

For a table containing activity data, open:

```text
Monitor
```

Verify the following metrics:

- Read requests.
- Write requests.
- Throttled requests.
- System errors.
- User errors.
- Successful request latency.

These metrics help monitor table activity while the system is in use.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.7-DynamoDB/dynamodb-monitoring-metrics.png"
    title="Figure 5.4.7.8: Amazon DynamoDB table monitoring metrics"
    width="100%"
>}}

## Verifying WebSocket Connection Data

After a user connects to the WebSocket API, open:

```text
DynamoDB
→ Tables
→ la_websocket_connections
→ Explore table items
```

Expected results:

- An item exists for the active connection.
- The item contains `item_id`.
- The item contains `connection_id`.
- The item contains the `ttl` attribute.
- When the connection ends, Lambda removes the record or it expires automatically.

Actual Connection IDs should not be included in the public report unless necessary.

Complete WebSocket connection testing is presented in **Section 5.5 — System Testing**.

## Result

After inspecting the resources through the AWS Management Console, the team confirmed that:

- Eight DynamoDB tables were deployed using Terraform.
- All tables are in the `Active` state.
- The tables use the `PAY_PER_REQUEST` billing mode.
- Partition Keys and Sort Keys are configured for their respective business requirements.
- Auction Catalog stores auction sessions, items, and related entities.
- Bid Events stores bid history and supports user-based queries.
- WebSocket Connections stores active WebSocket connections.
- TTL is enabled for connection, idempotency, and audit-event data.
- Global Secondary Indexes support additional query patterns.
- Point-in-Time Recovery is enabled for important tables.
- Server-side encryption protects stored data.
- DynamoDB Streams are enabled for auction-state and bid-event tables.
- Lambda functions can read and write data through the configured IAM permissions.
- DynamoDB is ready to support the REST API and real-time auction workflows.

The deployment and verification of the bid-processing queue are presented in **Section 5.4.8 — Amazon SQS FIFO**.