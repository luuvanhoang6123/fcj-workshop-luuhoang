---
title: "Terraform Overview"
date: 2026-07-13
weight: 1
chapter: false
pre: "<b>5.3.1. </b>"
---

## What is Terraform?

Terraform is an **Infrastructure as Code (IaC)** tool developed by HashiCorp that enables infrastructure to be defined and managed through configuration files instead of manual operations in the AWS Management Console.

Using Terraform, all AWS resources required by the system, such as IAM, Amazon S3, Amazon CloudFront, Amazon Cognito, AWS Lambda, Amazon API Gateway, Amazon DynamoDB, and Amazon SQS, are described as code. When deployment is required, Terraform automatically creates or updates these resources according to the defined configuration.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/terraform-overview.png" alt="Terraform Overview" width="60%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.1.</b> Terraform is used to manage the AWS infrastructure of the Live Auction system.
    </figcaption>
</figure>

---

## Benefits of Terraform

During the development of the project, the team selected Terraform for the following advantages:

- Automates infrastructure deployment using code.
- Ensures consistency across multiple deployments.
- Simplifies infrastructure management and change tracking.
- Facilitates team collaboration through Git version control.
- Supports reusable and scalable infrastructure configurations.

---

## Terraform in the Live Auction System

For the **Live Auction** system, Terraform is used to provision and manage the AWS infrastructure required by the application.

The managed AWS resources include:

- AWS Identity and Access Management (IAM)
- Amazon Cognito
- Amazon S3
- Amazon CloudFront
- AWS Lambda
- Amazon API Gateway
- API Gateway WebSocket
- Amazon DynamoDB
- Amazon SQS FIFO

Using Terraform enables the team to provision the entire AWS infrastructure with only a few commands while ensuring that every team member deploys the system using the same infrastructure configuration.

---

## Result

After becoming familiar with Terraform and preparing the configuration files, the team organized the infrastructure source code within the **Infrastructure** directory. The directory structure and organization will be presented in the next section.