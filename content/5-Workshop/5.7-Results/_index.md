---
title: "Results"
date: 2026-08-09
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

## Overview

After the analysis, development, deployment, and testing process, the team completed the **Live Auction** system on **Amazon Web Services (AWS)**.

The system was built using a serverless architecture, with Terraform managing the infrastructure as code. The frontend, backend, account authentication, data storage, and real-time auction components were integrated into a complete system.

Detailed functional testing results and supporting screenshots are presented in **Section 5.5 — System Testing**. Therefore, this section focuses on summarizing the deployment results, knowledge gained, limitations, and future development of the project.

## Deployment Results

The complete system infrastructure was built with **Terraform** and deployed in:

```text
Asia Pacific (Singapore) — ap-southeast-1
```

The main results include:

- The infrastructure was divided into Terraform modules based on functional groups.
- Terraform State was stored and managed centrally.
- Infrastructure changes were reviewed using `terraform plan`.
- AWS resources were deployed using `terraform apply`.
- Resource names followed the project naming convention.
- IAM Roles and IAM Policies followed the principle of least privilege.
- User Frontend and Admin Frontend were deployed independently.
- The system can be updated or redeployed from the Terraform source code.
- Monitoring, backup, and CI/CD components were integrated into the infrastructure.

The following primary AWS services were deployed:

| Service | Result |
| --- | --- |
| **AWS IAM** | Manages Roles, Policies, and access permissions between services. |
| **Amazon Cognito** | Authenticates accounts and separates User and Admin permissions. |
| **Amazon S3** | Stores both frontend applications and auction item images. |
| **Amazon CloudFront** | Distributes frontend content through a CDN. |
| **AWS Lambda** | Executes user, administrator, and auction business logic. |
| **Amazon API Gateway** | Provides REST APIs for both frontend applications. |
| **API Gateway WebSocket** | Provides bidirectional real-time connections. |
| **Amazon DynamoDB** | Stores business data, connections, and auction states. |
| **Amazon SQS FIFO** | Receives and processes bid requests in the correct order. |
| **Amazon CloudWatch** | Monitors Logs, Metrics, and system activity. |
| **AWS Backup** | Creates backups of important data tables. |

## Functional Results

The system provides separate interfaces for users and administrators.

### User Frontend

Users can:

- Register, sign in, and sign out.
- View and update personal information.
- View auction-session lists and details.
- Create and manage their own auction sessions.
- Add one or more items to an auction session.
- Join an active auction.
- Submit bid requests.
- Receive updated prices and auction states in real time.

### Admin Frontend

Administrators can:

- View the system overview.
- Manage user accounts.
- Disable or enable accounts.
- Moderate auction sessions and items.
- Approve, reject, or cancel auction sessions.
- Manage product categories.
- Review administrator audit history.
- Invite and manage administrator accounts.

### Real-Time Auction

The real-time auction workflow was implemented using:

```text
API Gateway WebSocket
AWS Lambda
Amazon SQS FIFO
Amazon DynamoDB
```

When a user submits a bid:

1. The frontend sends the request through the WebSocket API.
2. Lambda WebSocket Handler validates the request.
3. A valid request is sent to Amazon SQS FIFO.
4. Lambda Bid Processor processes the messages in order.
5. The updated price and bid history are stored in DynamoDB.
6. Broadcast Lambda sends the result to WebSocket clients.
7. The user interface displays the updated price without reloading the page.

Detailed testing results for these functions are presented in **Section 5.5 — System Testing**.

## Knowledge and Experience Gained

Through the project, the team gained knowledge and experience in:

- Analyzing online auction system requirements.
- Designing a serverless architecture on AWS.
- Organizing Terraform infrastructure into multiple modules.
- Managing infrastructure through Infrastructure as Code.
- Using Terraform State and Backend.
- Configuring IAM Roles and IAM Policies.
- Implementing authentication and authorization with Amazon Cognito.
- Hosting websites with Amazon S3 and CloudFront.
- Building REST APIs with API Gateway and Lambda.
- Building real-time functionality with WebSocket API.
- Processing ordered messages with Amazon SQS FIFO.
- Designing NoSQL data models with Amazon DynamoDB.
- Using Global Secondary Indexes, TTL, Streams, and Point-in-Time Recovery.
- Monitoring the system through CloudWatch Logs and Metrics.
- Backing up data and preparing recovery plans.
- Identifying and resolving deployment errors.
- Collaborating as a team and managing source code with GitHub.

## Limitations

The system still has several limitations:

- Load testing with a large number of concurrent users has not been completed.
- Long-term operating costs have not been fully evaluated.
- Some CloudWatch Logs do not separately record Route Keys and Connection IDs.
- The user interface and user experience can be further improved.
- A custom domain has not been integrated.
- Comprehensive auction fraud-prevention mechanisms have not been implemented.
- In-depth security testing remains limited.
- The system has not been evaluated with actual production traffic.

## Future Development

The system can be extended in the future by:

- Integrating a custom domain and SSL certificate.
- Completing CI/CD processes for the frontend, backend, and infrastructure.
- Adding notifications through email or mobile devices.
- Adding transaction history and payment functionality.
- Improving fraud detection during bidding.
- Adding AWS WAF protection for CloudFront and API Gateway.
- Improving CloudWatch Dashboards and automated alerts.
- Adding AWS X-Ray for request tracing across services.
- Performing load testing and scalability evaluation.
- Optimizing costs based on actual usage data.
- Completing backup and disaster-recovery mechanisms.
- Improving the interface for mobile devices.

## Conclusion

The team completed the development and deployment of the **Live Auction** system using a serverless architecture on AWS.

Terraform automated the deployment process and ensured infrastructure consistency. AWS services were integrated to support frontend hosting, account authentication, authorization, API processing, data storage, ordered bid processing, and real-time status updates.

The Workshop results show that the system can support the primary operations required by users and administrators while providing a foundation for continued improvement and future expansion.