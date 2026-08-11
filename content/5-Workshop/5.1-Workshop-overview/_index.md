---
title : "Introduction"
date : 2026-07-13 
weight : 1 
chapter : false
pre : " <b> 5.1. </b> "
---

# Project Overview and Deployment Architecture

## Introduction

**Live Auction** is an online auction platform developed to provide a transparent and convenient auction environment with support for real-time data updates.

The system provides two separate interfaces for users and administrators. Users can create accounts, sign in, view their personal information, browse auction sessions, create and manage their own sessions, add one or more items to a session, and participate in bidding. Administrators can manage user accounts, manage product categories, review auction sessions, and create additional administrator accounts.

During the project, the team deployed the system on **Amazon Web Services (AWS)** using a **serverless architecture**. The complete infrastructure was built and managed using **Terraform (Infrastructure as Code)**, which automates the creation and configuration of AWS resources and maintains consistency between deployments.

After the infrastructure was deployed, AWS services were integrated to build the complete system. These services support the storage and distribution of both frontend applications, account authentication and authorization, business API processing, data storage, ordered bid request processing, and real-time auction status updates.

## Objectives

The Workshop was conducted with the following objectives:

* Understand the process of deploying a practical system on AWS.
* Deploy infrastructure using Terraform and the Infrastructure as Code model.
* Integrate AWS services to build the Live Auction system.
* Deploy separate interfaces for users and administrators.
* Implement real-time auction functionality using a serverless architecture.
* Test and evaluate the system after deployment.

## Deployment Architecture

The Live Auction system was deployed using a serverless architecture on AWS. Terraform manages the infrastructure, while AWS services provide account authentication, authorization, business logic processing, data storage, and real-time auction communication.

The User Frontend and Admin Frontend are deployed independently to separate user functionality from administrative functionality. Both interfaces communicate with the backend services through Amazon API Gateway. Access to protected functionality is controlled using tokens issued by Amazon Cognito.


The following diagram illustrates the actual deployment architecture of the Live Auction system on AWS.

{{< figure
    src="/images/5-Workshop/5.1-Workshop-overview/live-auction-deployment-architecture.jpg"
    title="Figure 5.1.1: Deployment architecture of the Live Auction system on AWS"
    width="100%"
>}}

## AWS Services Used

The following table lists the tools and AWS services used during the system deployment.

| Tool/Service           | Role                                                                          |
| ---------------------- | ----------------------------------------------------------------------------- |
| **Terraform**          | Manages and deploys infrastructure using the Infrastructure as Code model.    |
| **AWS IAM**            | Manages Roles, Policies, and access between AWS services.                     |
| **Amazon Cognito**     | Authenticates accounts and distinguishes User and Admin permissions.          |
| **Amazon S3**          | Stores the User Frontend, Admin Frontend, and static resources.               |
| **Amazon CloudFront**  | Distributes both frontend applications through a CDN.                         |
| **AWS Lambda**         | Executes user, administrator, and auction business logic.                     |
| **Amazon API Gateway** | Provides REST API and WebSocket API endpoints for the system.                 |
| **Amazon DynamoDB**    | Stores accounts, categories, auction sessions, items, bids, and system state. |
| **Amazon SQS FIFO**    | Receives and processes bid requests sequentially.                             |

The configuration and deployment of each component are described in detail in the following sections of the Workshop.