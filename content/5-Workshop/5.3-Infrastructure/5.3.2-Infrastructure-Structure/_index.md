---
title: "Infrastructure Directory Structure"
date: 2026-07-13
weight: 2
chapter: false
pre: "<b>5.3.2. </b>"
---


The entire infrastructure of the **Live Auction** system is managed under the **infra/** directory using Terraform following the **Infrastructure as Code (IaC)** approach. Instead of defining all AWS resources in a single configuration file, the infrastructure is organized into multiple independent modules, each representing a specific layer of the system. This modular organization simplifies development, testing, maintenance, and future expansion while minimizing the impact of changes between different components.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/infrastructure-structure.png" alt="Infrastructure Directory Structure" width="45%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.2.</b> Overall Infrastructure directory structure of the project.
    </figcaption>
</figure>

---

## 00-bootstrap Directory

The **00-bootstrap/** directory is used during the initial setup phase of the project. It contains the scripts required to prepare the Terraform environment before provisioning the main AWS infrastructure.

This directory includes the **bootstrap-remote-state.ps1** script, which creates the resources required for the Terraform Backend, allowing the Terraform State to be stored remotely on AWS instead of locally. In addition, the **tests/** directory contains scripts used to validate the bootstrap process.

Separating the bootstrap stage from the remaining infrastructure ensures that Terraform Backend only needs to be configured once and allows all team members to share the same infrastructure state.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/bootstrap-structure.png" alt="Bootstrap Structure" width="55%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.3.</b> Directory structure of <code>00-bootstrap</code>.
    </figcaption>
</figure>

---

## 01-foundation Directory

After completing the bootstrap process, Terraform continues with the **01-foundation/** directory, which contains the common foundation of the infrastructure.

This module includes the **backend.tf** configuration file used to configure the Terraform Backend and connect to the remote Terraform State. It serves as the foundation for all subsequent infrastructure modules.

Separating the foundation layer enables common configurations to be defined once and reused throughout the entire project.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/foundation-structure.png" alt="Foundation Structure" width="45%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.4.</b> Directory structure of <code>01-foundation</code>.
    </figcaption>
</figure>

---

## 03-identity Directory

The **03-identity/** directory manages all identity and authentication resources of the system.

This module provisions services such as **AWS Identity and Access Management (IAM)** and **Amazon Cognito**, which are responsible for user management, access control, and authentication within the Live Auction platform.

In addition to standard Terraform configuration files such as **main.tf**, **variables.tf**, **outputs.tf**, **providers.tf**, **backend.tf**, and **versions.tf**, this module also contains **terraform.lock.hcl**, the **.terraform/** directory, and the generated **tfplan** file used during the planning stage.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/identity-structure.png" alt="Identity Structure" width="35%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.5.</b> Structure of the <code>03-identity</code> module.
    </figcaption>
</figure>

---

## 04-data Directory

The **04-data/** directory is responsible for provisioning the data layer of the system. This module manages the Terraform configuration for **Amazon DynamoDB**, which serves as the primary database of the Live Auction platform.

The DynamoDB tables created in this module are used to store auction information, auction session states, bidding history, WebSocket connections, and other application data. Keeping the data layer in a separate module simplifies database maintenance and future scalability.

Besides the **main.tf** file that defines the DynamoDB resources, the module also includes **variables.tf**, **outputs.tf**, **providers.tf**, **backend.tf**, and **versions.tf** to standardize the deployment process.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/data-structure.png" alt="Data Structure" width="35%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.6.</b> Structure of the <code>04-data</code> module.
    </figcaption>
</figure>

---

## 05-messaging Directory

The **05-messaging/** directory manages the messaging services of the system. In the Live Auction project, this module provisions **Amazon SQS FIFO**, which is responsible for receiving and processing bid requests from users.

During live auctions, multiple users may submit bids simultaneously. Using a FIFO queue ensures that bid requests are processed in the exact order they are received, reducing conflicts and maintaining the consistency of auction sessions.

In addition to the standard Terraform configuration files, this module also stores the output generated by the **terraform plan** command for deployment verification.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/messaging-structure.png" alt="Messaging Structure" width="35%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.7.</b> Structure of the <code>05-messaging</code> module.
    </figcaption>
</figure>

---

## 06-compute Directory

The **06-compute/** directory is responsible for the compute layer of the system. This module provisions the **AWS Lambda** functions that implement the core business logic of the Live Auction application.

These Lambda functions process REST API requests, handle WebSocket events, perform user authentication, manage auction sessions, execute background tasks, and process messages received from Amazon SQS.

The module also contains the **stage3-control-plane/** directory, which organizes deployment resources related to the Control Plane stage. This separation makes the infrastructure easier to manage, maintain, and test.

Like other modules, it includes **main.tf**, **variables.tf**, **outputs.tf**, **providers.tf**, **backend.tf**, **versions.tf**, and **terraform.lock.hcl**.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/compute-structure.png" alt="Compute Structure" width="40%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.8.</b> Structure of the <code>06-compute</code> module.
    </figcaption>
</figure>

---

## 07-api Directory

The **07-api/** directory is responsible for provisioning the communication layer between the system and its users through **Amazon API Gateway**.

Besides configuring REST APIs for application features such as authentication, user management, and auction management, this module also provisions **API Gateway WebSocket**, enabling real-time communication between clients and the backend during live auctions.

This module contains the standard Terraform configuration files as well as deployment-related resources required for API provisioning.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/api-structure.png" alt="API Structure" width="30%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.9.</b> Structure of the <code>07-api</code> module.
    </figcaption>
</figure>

---

## 09-edge Directory

The **09-edge/** directory manages the edge services of the infrastructure. This module provisions **Amazon S3** for hosting the user and administrator web applications together with static assets, while **Amazon CloudFront** is configured to distribute the content through AWS's global CDN.

Using CloudFront reduces latency, improves page loading performance, and provides a better user experience across different geographic locations.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/edge-structure.png" alt="Edge Structure" width="30%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.10.</b> Structure of the <code>09-edge</code> module.
    </figcaption>
</figure>

---

## tests Directory

In addition to the infrastructure modules, the project includes a **tests/** directory for validating the deployment process.

This directory contains several PowerShell scripts organized into multiple deployment stages such as **stage1**, **stage2**, **stage3**, and **stage4**. These scripts verify the functionality of Identity, Compute, API, Data, Messaging, as well as Integration Tests and End-to-End Tests.

Maintaining dedicated test scripts allows the team to verify infrastructure after each deployment and detect issues early during development.

<figure style="text-align: center;">
    <img src="/images/5-Workshop/5.3-Infrastructure/tests-structure.png" alt="Tests Structure" width="30%">
    <figcaption style="text-align: center;">
        <b>Figure 5.3.11.</b> Structure of the <code>tests</code> directory.
    </figcaption>
</figure>

---

## Common Terraform Configuration Files

Most infrastructure modules follow the same directory layout to maintain consistency throughout the project. Each Terraform configuration file serves a specific purpose.

| Configuration File     | Description                                               |
| ---------------------- | --------------------------------------------------------- |
| **main.tf**            | Defines the AWS resources within the module.              |
| **variables.tf**       | Declares input variables used by the module.              |
| **outputs.tf**         | Defines output values shared with other modules.          |
| **providers.tf**       | Configures the Terraform provider and AWS connection.     |
| **backend.tf**         | Configures the remote Terraform Backend.                  |
| **versions.tf**        | Specifies the required Terraform and Provider versions.   |
| **terraform.lock.hcl** | Locks provider versions to ensure consistent deployments. |
| **tfplan**             | Stores the execution plan generated by `terraform plan`.  |

Standardizing the module structure makes the infrastructure easier to understand, maintain, and extend.

---

## Result

After organizing the Infrastructure directory, the entire Terraform codebase was structured into separate functional modules. This organization improves deployment consistency, simplifies maintenance, and provides a solid foundation for the initialization, planning, and provisioning steps described in the following sections.