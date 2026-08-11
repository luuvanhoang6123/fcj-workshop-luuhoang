---
title: "Amazon SQS FIFO"
date: 2026-07-27
weight: 8
chapter: false
pre: " <b> 5.4.8. </b> "
---

## Overview

**Amazon Simple Queue Service (Amazon SQS)** is a fully managed message queue service provided by AWS. It helps decouple system components and supports asynchronous request processing.

In the Live Auction system, **Amazon SQS FIFO** receives bid requests before forwarding them to Lambda for processing. FIFO ensures that bid requests belonging to the same auction are processed in the order in which they were received.

The bid-processing flow is implemented as follows:

```text
User submits a bid
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
Updated price is delivered to users
```

Amazon SQS FIFO acts as an intermediary between Lambda WebSocket Handler and Lambda Bid Processor. It prevents multiple bid requests from being processed concurrently in an incorrect order.

## Role of Amazon SQS FIFO

Amazon SQS FIFO is used in the system to:

- Receive bid requests from WebSocket Handler.
- Process bid requests in the correct order.
- Group messages by auction session or auction item.
- Prevent two bid requests from being processed in the wrong order.
- Separate request reception from business processing.
- Allow Lambda Bid Processor to process requests asynchronously.
- Automatically retry messages when processing fails.
- Transfer messages that cannot be processed to a Dead-letter Queue when configured.

## Related Components

| Component | Role |
| --- | --- |
| **API Gateway WebSocket** | Receives connections and messages from users. |
| **Lambda `la-ws-handler`** | Validates WebSocket messages and sends bid requests to SQS FIFO. |
| **Amazon SQS FIFO** | Temporarily stores and orders bid requests. |
| **Lambda `la-bid-processor`** | Receives messages from SQS and processes bid operations. |
| **Amazon DynamoDB** | Stores auction states and bid history. |
| **Lambda `la-broadcast`** | Sends bid results to the relevant WebSocket connections. |
| **Dead-letter Queue** | Stores messages that cannot be processed after the configured number of retries. |

## Step 1: Open Amazon SQS

Sign in to the **AWS Management Console** and select the correct Region:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

Enter the following service name in the search bar:

```text
SQS
```

Then select:

```text
Simple Queue Service
```

## Step 2: Verify the Queue List

In the Amazon SQS interface, open:

```text
Queues
```

Find the Queue used to process Live Auction bid requests.

A FIFO Queue name must end with:

```text
.fifo
```

Verify:

- Queue name.
- Queue type is FIFO.
- Region is `ap-southeast-1`.
- Queue creation time.
- Queue operating status.
- Dead-letter Queue, if configured.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-queue-list.png"
    title="Figure 5.4.8.1: Amazon SQS queues of the Live Auction system"
    width="100%"
>}}

## Step 3: Verify SQS FIFO Queue Information

Select the FIFO Queue used to process bid requests and open:

```text
Details
```

Verify:

- Type is `FIFO`.
- Queue ARN.
- Queue URL.
- Content-based deduplication.
- Visibility timeout.
- Message retention period.
- Delivery delay.
- Maximum message size.
- Receive message wait time.

The most important property to confirm is:

```text
FIFO
```

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-fifo-details.png"
    title="Figure 5.4.8.2: Amazon SQS FIFO Queue configuration"
    width="100%"
>}}

## Step 4: Verify the FIFO Configuration

In the Queue configuration, inspect the properties specific to FIFO Queues.

Verify:

- The Queue name ends with `.fifo`.
- Queue type is FIFO.
- A duplicate-message prevention mechanism is configured.
- Message order is preserved within the same Message Group.
- Messages use a Message Deduplication ID or Content-based Deduplication.

In the Live Auction system, requests belonging to the same auction session or item can use the same **Message Group ID**. Therefore, SQS preserves the correct bid order within that group.

For example:

```text
MessageGroupId = auction-session-or-item-id
```

Each request also needs a unique identifier to prevent duplicate processing:

```text
MessageDeduplicationId = unique-bid-request-id
```

{{% notice info %}}
SQS FIFO guarantees ordering only within the same Message Group. Messages belonging to different Message Groups can still be processed in parallel.
{{% /notice %}}

## Step 5: Verify the Lambda Trigger

Open the following Lambda function:

```text
la-bid-processor
```

From the **Configuration** tab, select:

```text
Triggers
```

Verify the Amazon SQS Trigger.

Confirm:

- Trigger Type is `SQS`.
- The associated Queue is the FIFO Queue used for bid processing.
- The Trigger is enabled.
- Batch size.
- Event source mapping is active.
- The Lambda function receiving messages is `la-bid-processor`.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-lambda-trigger.png"
    title="Figure 5.4.8.3: SQS Trigger of Lambda Bid Processor"
    width="100%"
>}}

## Step 6: Verify Event Source Mapping

Open the Event Source Mapping information in the Trigger configuration of `la-bid-processor`.

Verify:

- Event source is the SQS FIFO Queue.
- Status is `Enabled`.
- Batch size is appropriate for bid processing.
- Maximum batching window, if configured.
- Partial batch failure reporting, if used.
- Lambda has permission to receive and delete messages from the Queue.

Event Source Mapping allows AWS Lambda to automatically read messages from SQS. After successful processing, the message is removed from the Queue.

If processing fails, the message becomes visible again after the Visibility Timeout expires and can be retried.

## Step 7: Verify the Dead-Letter Queue

If the system uses a Dead-letter Queue, open the primary FIFO Queue and locate:

```text
Dead-letter queue
```

or:

```text
Redrive policy
```

Verify:

- Dead-letter Queue name.
- Dead-letter Queue ARN.
- Maximum receives.
- The Dead-letter Queue exists in the same Region.
- The Queue name follows the project naming convention.

The Dead-letter Queue receives messages that cannot be processed after the configured number of attempts. This mechanism allows the team to investigate errors without losing bid messages.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-dead-letter-queue.png"
    title="Figure 5.4.8.4: Dead-letter Queue configuration of SQS FIFO"
    width="100%"
>}}

{{% notice warning %}}
Keep this figure only if a Dead-letter Queue has actually been configured. If AWS indicates that no Dead-letter Queue exists, the report must not state that this feature has been deployed.
{{% /notice %}}

## Step 8: Verify IAM Permissions

Open the IAM Role used by:

```text
la-bid-processor
```

Verify the permissions required for Lambda to work with Amazon SQS, including:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

The IAM Role used by the Lambda function that sends bid requests to the Queue requires:

```text
sqs:SendMessage
```

These permissions should be restricted to the correct Queue ARN according to the principle of least privilege instead of granting access to all SQS resources.

Additional screenshots are unnecessary if the related IAM permissions have already been presented in **Section 5.4.1 — AWS IAM and Amazon Cognito**.

## Step 9: Verify Monitoring Metrics

In Amazon SQS, select the FIFO Queue and open:

```text
Monitoring
```

Select a period containing activity and inspect:

- Number of messages sent.
- Number of messages received.
- Number of messages deleted.
- Approximate number of messages available.
- Approximate number of messages not visible.
- Age of oldest message.

These metrics help confirm that:

- Bid requests were sent to the Queue.
- Lambda received the messages.
- Messages were deleted after successful processing.
- A large message backlog did not accumulate.
- Messages were not waiting in the Queue for too long.

{{< figure
    src="/images/5-Workshop/5.4-AWS-services/5.4.8-SQS-FIFO/sqs-cloudwatch-metrics.png"
    title="Figure 5.4.8.5: Amazon SQS FIFO monitoring metrics"
    width="100%"
>}}

## Step 10: Verify the Actual Workflow

To verify the complete bid-processing workflow:

1. Sign in to the User Frontend.
2. Open an active auction session.
3. Join the auction.
4. Submit a valid bid.
5. Open Amazon SQS and inspect the Monitoring metrics.
6. Open the CloudWatch Logs of `la-bid-processor`.
7. Inspect the auction data stored in Amazon DynamoDB.
8. Confirm that the updated price appears in the user interface.

If Lambda processes messages quickly, the number of available messages may return to `0` before the Queue is inspected. In this case, use the **Messages sent**, **Messages received**, and **Messages deleted** charts to demonstrate that the Queue was active.

## Result

After verification, Amazon SQS FIFO was confirmed to be integrated into the Live Auction bid-processing workflow.

The achieved results include:

- Bid requests are received through the WebSocket API.
- Lambda `la-ws-handler` sends valid requests to SQS FIFO.
- Messages within the same group preserve their processing order.
- Lambda `la-bid-processor` is triggered to process messages.
- Bid results are stored in Amazon DynamoDB.
- Updated auction states are sent to users through WebSocket.
- IAM controls permissions to send, receive, and delete messages.
- CloudWatch Metrics support activity and backlog monitoring.
- Failed messages can be transferred to a Dead-letter Queue if a Redrive Policy is configured.

Amazon SQS FIFO provides ordered bid processing, reduces direct dependencies between Lambda functions, and improves system stability when multiple users submit bids concurrently.