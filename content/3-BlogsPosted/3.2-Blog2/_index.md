---
title: "Blog 2 - Lambda Scales Quickly, but the Database Does Not Scale the Same Way"
date: 2026-08-09
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# Lambda Scales Quickly, but the Database Does Not Scale the Same Way

## Overview

During the development of the **Live Auction** system, the team studied the migration of the backend from **FastAPI** to a serverless architecture using **AWS Lambda**, while the database remained on **Amazon RDS for MySQL**.

AWS Lambda allows the system to automatically scale according to the number of incoming requests. However, Lambda's rapid scaling introduces an important concern: the downstream database does not necessarily scale its connections and computing resources at the same rate.

This article analyzes database connection management when AWS Lambda handles many concurrent requests. It also introduces **Reserved Concurrency** and **Amazon RDS Proxy** as possible solutions for protecting the database.

## Traditional Database Connection Model

For a backend running continuously on **Amazon EC2** or in a container, the application usually maintains a connection pool to the database.

The connection flow is generally as follows:

```text
User
  ↓
Backend on EC2 or Container
  ↓
Connection Pool
  ↓
Amazon RDS for MySQL
```

The connection pool maintains a limited number of database connections and reuses them across multiple requests. Therefore, the application does not need to establish a new database connection for every request.

This model is relatively easy to control because:

* The number of backend instances is usually stable.
* The maximum number of connections can be configured in advance.
* Connections can be maintained and reused for a long time.
* The database can process a relatively predictable number of connections.

## What Changes When AWS Lambda Is Used?

AWS Lambda runs application code inside **execution environments**. When the number of concurrent requests increases, Lambda can create additional execution environments to process the workload.

For example, near the end of an auction session, the system may receive:

```text
500 concurrent requests
          ↓
Multiple concurrent Lambda executions
          ↓
Multiple new database connections
```

If each execution opens a direct connection to Amazon RDS, the number of connections can increase rapidly within a short period.

Although Lambda can scale according to the workload, Amazon RDS is still limited by:

* The selected DB instance configuration.
* The maximum number of database connections.
* Available CPU and memory.
* Transaction processing capacity.
* Read and write throughput.
* The time required to establish new connections.

Therefore, the fact that Lambda can continue scaling does not mean that the downstream database can handle the same amount of growth.

## Risks of a Sudden Increase in Connections

Suppose the system receives 500 bid requests simultaneously, and every Lambda execution establishes a new MySQL connection.

The processing flow may become:

```text
500 concurrent requests
          ↓
500 Lambda executions
          ↓
Up to 500 new connection requests
          ↓
Amazon RDS for MySQL
```

The actual number of connections may vary depending on how Lambda reuses its execution environments and existing connections. Nevertheless, sudden scaling can still place significant pressure on the database.

Possible problems include:

* Exhaustion of available database connections.
* Increased connection establishment time.
* Higher API latency.
* Slower transaction processing.
* Request timeouts.
* Negative impact on other workloads using the same database.
* System errors during periods of high traffic.

This risk is especially important for an online auction system because bidding traffic may increase sharply near the end of an auction session.

{{% notice note %}}
Serverless scalability should not be evaluated based only on AWS Lambda. The capacity and limitations of every downstream dependency must also be considered.
{{% /notice %}}

## Limiting Lambda with Reserved Concurrency

One solution for protecting downstream services is to configure **Reserved Concurrency** for a Lambda function.

Reserved Concurrency helps to:

* Limit the number of concurrent executions of a Lambda function.
* Prevent Lambda from scaling beyond the database capacity.
* Reserve concurrency for an important function.
* Prevent one function from consuming all available account concurrency.
* Control the amount of traffic sent to downstream services.

The processing flow after applying a concurrency limit can be represented as follows:

```text
Increasing request volume
          ↓
AWS Lambda
          ↓
Reserved Concurrency limit
          ↓
Amazon RDS receives controlled traffic
```

However, limiting concurrency may cause some requests to be delayed or throttled when traffic exceeds the configured limit. Therefore, the concurrency value should be selected according to the actual database capacity and system requirements.

## Amazon RDS Proxy

Another AWS service designed for serverless applications connecting to relational databases is **Amazon RDS Proxy**.

Instead of allowing Lambda to connect directly to the database:

```text
AWS Lambda
    ↓
Amazon RDS
```

The system can use the following architecture:

```text
AWS Lambda
    ↓
Amazon RDS Proxy
    ↓
Amazon RDS for MySQL
```

RDS Proxy operates between the application and the database. It maintains a pool of database connections and allows multiple Lambda executions to reuse those connections.

## How RDS Proxy Supports Connection Management

When many Lambda executions appear at the same time, the database should not always be required to create the same number of new connections.

RDS Proxy helps to:

* Maintain a connection pool to the database.
* Reuse connections across multiple requests.
* Reduce the number of new connection establishments.
* Reduce direct connection pressure on Amazon RDS.
* Support unpredictable increases in application traffic.
* Improve resiliency during database failover.
* Manage database credentials more securely when combined with AWS Secrets Manager and IAM.

The general flow is:

```text
Multiple Lambda executions
            ↓
      Amazon RDS Proxy
            ↓
 Managed connection pool
            ↓
  Amazon RDS for MySQL
```

For example, when 300 Lambda executions appear simultaneously, RDS Proxy can manage and reuse suitable database connections instead of requiring the database to establish a completely new connection for every execution.

## RDS Proxy Does Not Solve Every Database Limitation

RDS Proxy primarily addresses **connection management**. It does not automatically increase the complete processing capacity of the database.

RDS Proxy cannot replace the need to:

* Optimize SQL queries.
* Design suitable indexes.
* Reduce transaction execution time.
* Select a DB instance with sufficient CPU and memory.
* Optimize the data structure.
* Configure suitable timeouts.
* Monitor performance and database connections.
* Control the request volume processed by the database.

If the queries are slow or the database lacks sufficient resources, adding RDS Proxy will not automatically solve the entire performance problem.


## Application to the Live Auction System

Traffic in an auction system may not be distributed evenly. Near the end of an auction session, many users may submit bid requests within a short period.

If the backend uses Lambda with direct RDS connections, the system should consider:

* The number of users who may bid concurrently.
* The maximum concurrency of the Lambda function.
* The connection limit of the DB instance.
* The execution time of a bidding transaction.
* Duplicate-request protection.
* Correct bid ordering.
* Throttling and timeout handling.
* Monitoring of active database connections.

A possible architecture is:

```text
User
  ↓
Amazon API Gateway
  ↓
AWS Lambda
  ↓
Amazon RDS Proxy
  ↓
Amazon RDS for MySQL
```

In addition to RDS Proxy, the system may use:

* **Reserved Concurrency** to control Lambda executions.
* **Amazon CloudWatch** to monitor errors, latency, and concurrency.
* **Amazon SQS** to buffer suitable workloads.
* Controlled retry mechanisms.
* Appropriate execution and connection timeouts.
* Transactions and locking mechanisms suitable for auction operations.

This article presents an architectural approach studied during the development process. The final Live Auction deployment may use different storage and processing services depending on the actual system requirements.

## Comparison of Connection Approaches

| Criterion | Direct Lambda-to-RDS Connection | Lambda Connection Through RDS Proxy |
| --- | --- | --- |
| Connection management | Each execution manages its connection | Proxy manages and reuses connections |
| Traffic spike handling | May create many connections in a short time | Reduces direct connection pressure |
| Connection pool | Depends on the execution environment | Centrally managed by RDS Proxy |
| Failover behavior | Application handles reconnection | Proxy helps improve reconnection behavior |
| Complexity | Simpler architecture | Requires additional Proxy, IAM, and Secrets configuration |
| Cost | No RDS Proxy cost | Additional RDS Proxy usage cost |
| Suitable workload | Small workload with stable concurrency | Serverless workload with variable concurrency |

## Lessons Learned

Through this research, the team learned that:

* Serverless scalability must be evaluated across the complete architecture.
* Fast Lambda scaling does not mean the database scales in the same way.
* Database connections created by Lambda executions must be controlled.
* Reserved Concurrency can protect downstream services from excessive traffic.
* RDS Proxy is suitable for workloads with many short-lived connections or variable concurrency.
* RDS Proxy addresses connection management but does not replace database optimization.
* Auction systems must pay special attention to traffic spikes near session completion.
* Monitoring concurrency, connection count, latency, and error rate is necessary before operating the system in production.

## Published Article

The article was shared with the **AWS Study Group – First Cloud Journey** community.

**Title:** Lambda Scales Quickly, but the Database Does Not Scale the Same Way

**Article link:** [View the article on AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj/posts/2238842566880703/)

{{< figure
    src="/images/3-BlogsPosted/3.2-Blog2/blog2-facebook-post.png"
    title="Figure 3.2.1: The AWS Lambda and Amazon RDS article published on AWS Study Group"
    width="100%"
>}}

## Results

After completing the article, the team:

* Better understood the scalability differences between AWS Lambda and Amazon RDS.
* Identified the risk of connection exhaustion in serverless architectures.
* Learned how Reserved Concurrency can protect downstream services.
* Understood the role of Amazon RDS Proxy in connection pooling and reuse.
* Connected database connection management with traffic spikes in auction systems.
* Shared the research findings with the AWS Study Group community.

**Hashtags:** `#AWS` `#AWSLambda` `#AmazonRDS` `#RDSProxy` `#Serverless` `#CloudEngineering` `#SystemDesign`