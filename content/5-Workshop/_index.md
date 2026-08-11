---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Building and Deploying the Live Auction System on AWS

#### Overview

**Live Auction** is an online auction platform that allows users to register accounts, follow auction sessions, and submit bids in real time.

The system provides two separate interfaces for users and administrators. Users can manage their personal information, create auction sessions, add one or more items, follow auctions, and submit bids. Administrators can manage user accounts, manage product categories, approve auction sessions, and create additional administrator accounts.

In this Workshop, the team presents the process of building and deploying the **Live Auction** system on **Amazon Web Services (AWS)**. The infrastructure is deployed using **Terraform**, which automates the creation and management of AWS resources while ensuring consistency across deployments.

The User Frontend and Admin Frontend are stored separately in **Amazon S3** and distributed through **Amazon CloudFront**. Account authentication and authorization are implemented using **Amazon Cognito** together with **AWS IAM**. Business APIs are implemented using **AWS Lambda** and **Amazon API Gateway**.

For real-time auction functionality, the system uses **API Gateway WebSocket**, **Amazon SQS FIFO**, and **Amazon DynamoDB** to receive bid requests, process messages in the correct order, store auction states, and deliver updates to connected users.

The Workshop covers environment preparation, infrastructure deployment with Terraform, AWS service verification, system testing, resource cleanup guidance, and a summary of the achieved results.

#### Contents

1. [Project Introduction and Deployment Architecture](5.1-Workshop-overview/)
2. [Environment Preparation](5.2-Preparation/)
3. [Infrastructure Deployment with Terraform](5.3-Infrastructure/)
4. [Deployed AWS Services](5.4-AWS-Services/)
5. [System Testing](5.5-Testing/)
6. [Resource Cleanup](5.6-Cleanup/)
7. [Results](5.7-Results/)