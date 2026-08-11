---
title: "Published Blog Posts"
date: 2026-08-03
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

During the internship, my team members and I researched and shared the knowledge we gained about AWS services with the [AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj) community.

Currently, our team has published the following blog post:

### [Blog 1 - Live Auction: Building a Real-Time Auction Platform on AWS Serverless](3.1-Blog1/)

This blog post presents the evolution of the **Live Auction** platform from an initial architecture using React/Vite, FastAPI, and MySQL into a serverless architecture on AWS. It explains how the system handles user authentication, concurrent bid requests, ordered bid processing, real-time auction updates, image storage, monitoring, and infrastructure deployment.

The architecture integrates **Amazon S3**, **Amazon CloudFront**, **Amazon Cognito**, **Amazon API Gateway**, **AWS Lambda**, **Amazon DynamoDB**, and **Amazon SQS FIFO**. The infrastructure is managed with **Terraform**, while AWS monitoring, backup, security, and CI/CD services support system deployment and operation.

### [Blog 2 - Lambda Scales Quickly, but the Database Does Not Scale the Same Way](3.2-Blog2/)

This article analyzes database connection management when migrating a backend from a long-running model on **Amazon EC2 or containers** to an **AWS Lambda** architecture, while the data continues to be stored in **Amazon RDS for MySQL**.

It explains the risk of a sudden increase in database connections when Lambda scales in response to concurrent requests, especially near the end of an auction session. The article also introduces **Reserved Concurrency** and **Amazon RDS Proxy** as solutions for protecting the database, reusing connections, and reducing direct connection pressure on Amazon RDS.

### [Blog 3 - Optimizing Amazon S3 Storage Costs: Understanding Storage Classes and Lifecycle Policies](3.3-Blog3/)

This article introduces common **Amazon S3** storage classes, including S3 Standard, Standard-IA, One Zone-IA, and the S3 Glacier storage classes. It explains how to select an appropriate storage class based on access frequency, resiliency requirements, and data retrieval time.

The article also explains how to use an **S3 Lifecycle Policy** to automatically transition objects to lower-cost storage classes or delete them after their retention period expires. This allows the system to optimize storage costs without requiring a separate cron job or Lambda function.