---
title: "Proposal"
date: 2026-07-13
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# LIVE AUCTION PLATFORM ON AWS

## An Online Auction Platform on Amazon Web Services

### 1. Executive Summary

**Live Auction** is a proposed online auction platform designed to provide a transparent, convenient, and real-time bidding environment. The system enables users to register accounts, manage products, create and participate in auction sessions, and receive live bidding updates throughout the auction process. The project aims to leverage **Amazon Web Services (AWS)** to build a scalable, highly available, and cloud-native auction platform.

The application is developed using **React/Vite** for the frontend, **FastAPI (Python)** for the backend, and **MySQL** during local development. In the proposed architecture, the frontend is intended to be hosted on **Amazon S3**, while backend services are planned to run on AWS compute services. Additional AWS services, including **Amazon RDS for MySQL**, **Amazon API Gateway**, **AWS Lambda**, **Amazon Cognito**, **Amazon DynamoDB**, **Amazon SQS**, and **Amazon CloudFront**, are considered as part of the overall cloud architecture to improve scalability, security, and system performance.

This proposal presents the initial system architecture, the AWS services considered for the project, and the deployment strategy envisioned by the development team. During implementation, certain architectural components may be adjusted according to project requirements and deployment constraints.

---

### 2. Problem Statement

#### Current Challenges

Online auction platforms must process a large number of user requests simultaneously while maintaining data consistency and real-time synchronization. As the number of participants increases, the system may encounter several technical challenges if the architecture is not properly designed.

Some common challenges include:

- Bid information is not updated in real time.
- Concurrent bidding requests may be processed out of order.
- Users may not receive immediate auction updates.
- System performance may degrade under high traffic.
- The architecture becomes difficult to scale as new features are added.
- Infrastructure management and monitoring become increasingly complex.
- User data and application resources require stronger security protection.

In addition, deploying the entire application on a single server introduces a **Single Point of Failure (SPOF)**, making future expansion and maintenance more difficult.

#### Proposed Solution

To address these challenges, the team proposes deploying the **Live Auction** platform on **Amazon Web Services (AWS)** through multiple development phases.

The initial phase focuses on implementing the core components of the application, including the user interface, backend services, database, and image storage. These components are planned to be deployed using appropriate AWS services, providing a solid foundation for future expansion.

In later phases, the architecture is expected to evolve by integrating additional AWS services for user authentication, API management, real-time communication, messaging, scalable data storage, monitoring, and security.

#### Expected Benefits

The proposed solution offers several advantages:

- Provide users with a modern web-based auction platform.
- Support real-time auction updates.
- Separate application components for easier maintenance and scalability.
- Leverage managed AWS services to reduce infrastructure management overhead.
- Improve security, monitoring, and operational reliability.
- Establish a foundation for adopting serverless architecture in future development phases.

---

### 3. Solution Architecture

> **Note:** This chapter presents the **proposed system architecture** created during the design phase of the project. During implementation, certain architectural components may be adjusted according to project requirements and deployment constraints. The final deployment architecture will be presented in **Chapter 5 – Workshop**.

#### 3.1 Initial Deployment Architecture

The initial deployment architecture is designed to support the core functionality of the Live Auction platform while providing a foundation for future expansion.

The proposed architecture consists of the following major components:

1. Users access the system through a web browser.
2. The frontend is developed with React/Vite and planned to be hosted on Amazon S3.
3. The frontend communicates with backend services through REST APIs.
4. The backend is developed using FastAPI and is proposed to be deployed on AWS compute services.
5. Application data is stored in a database solution appropriate for system requirements.
6. Product images and static resources are stored in Amazon S3.
7. AWS Lambda is considered for background processing and event-driven tasks.
8. Amazon CloudWatch is used for monitoring system performance and collecting application logs.

This architecture focuses on delivering the essential functionality of the system while providing a migration path toward a more scalable serverless architecture in future development phases.

#### 3.2 Proposed AWS Architecture

The diagram below illustrates the proposed AWS architecture for the Live Auction platform. The architecture is designed with scalability, real-time auction processing, security, monitoring, and high availability as primary objectives.

Click the diagram below to view the full-size version.

[![Proposed AWS Architecture for Live Auction](/images/2-Proposal/live-auction-proposed-architecture.svg)](/images/2-Proposal/live-auction-proposed-architecture.svg)

> **Note:** This diagram represents the **target architecture** of the system. Some advanced components are planned for future implementation depending on the project scope and development progress.

#### 3.3 Overall Workflow

The overall workflow of the proposed architecture is summarized as follows:

1. Users access the application through a web browser.
2. Requests are routed to the frontend application.
3. Users authenticate before accessing protected resources.
4. The frontend sends API requests to backend services.
5. Backend services process business logic.
6. Application data is stored in the appropriate database services.
7. Images and static assets are stored in Amazon S3.
8. Bid requests may be processed sequentially through a messaging service in the extended architecture.
9. Auction updates are delivered to connected users in real time.
10. AWS monitoring services collect logs and monitor application health.

### 4. Proposed AWS Services

#### Amazon Route 53

Amazon Route 53 is proposed for domain name management and DNS routing. In the extended architecture, Route 53 can be integrated with Health Checks and Failover Routing policies to improve system availability and automatically redirect traffic to backup resources when necessary.

---

#### Amazon CloudFront

Amazon CloudFront is proposed as the Content Delivery Network (CDN) for the Live Auction platform. It distributes static assets such as HTML, CSS, JavaScript, and images from Amazon S3 to users through AWS Edge Locations, reducing latency and improving page loading performance.

CloudFront also supports HTTPS, content caching, and integration with AWS WAF to further enhance the security of the application.

---

#### AWS WAF

AWS Web Application Firewall (AWS WAF) is proposed to protect the web application against common web attacks such as SQL Injection (SQLi), Cross-Site Scripting (XSS), and abnormal traffic patterns.

When integrated with Amazon CloudFront or Amazon API Gateway, AWS WAF helps filter malicious requests, enforce rate limiting, and improve the overall security posture of the system.

---

#### Amazon S3

Amazon Simple Storage Service (Amazon S3) is proposed for storing various types of application assets, including:

- Frontend build artifacts.
- Administration portal.
- Product images.
- Static resources and supporting files.

Separating static content from backend services simplifies system management and improves scalability while enabling efficient content delivery through Amazon CloudFront.

---

#### Amazon EC2

In the initial deployment plan, Amazon EC2 is proposed as the execution environment for the FastAPI backend application.

The backend is intended to be containerized using Docker to ensure deployment consistency across development and production environments while providing flexibility for system administration and testing.

---

#### Amazon ECS and AWS Fargate

As part of the long-term architecture, the project considers migrating backend services to Amazon ECS running on AWS Fargate.

This approach reduces infrastructure management overhead while enabling automatic scaling, improved availability, and simplified container orchestration.

---

#### Amazon ECR

Amazon Elastic Container Registry (Amazon ECR) is proposed as the container image repository for backend services.

Docker images can be built through the CI/CD pipeline, stored in Amazon ECR, and deployed automatically to the target execution environment.

---

#### Elastic Load Balancing

Application Load Balancer (ALB) is proposed to distribute incoming traffic across multiple backend services or containers as the system scales.

ALB also provides health checks, SSL termination, and intelligent request routing to improve availability and reliability.

---

#### Amazon API Gateway

Amazon API Gateway is proposed as the unified entry point for all application APIs.

The service supports both REST APIs and WebSocket APIs while integrating seamlessly with other AWS services such as AWS Lambda and Amazon Cognito.

Within the proposed architecture:

- REST APIs are responsible for handling business operations such as user management, auction management, and product management.
- WebSocket APIs provide real-time communication for auction events, enabling connected users to receive live bidding updates instantly.

#### AWS Lambda

AWS Lambda is proposed as the primary serverless compute service for executing business logic without managing dedicated servers. Lambda functions are triggered only when requests or events occur, allowing the system to scale automatically while reducing operational costs.

Within the proposed architecture, AWS Lambda may be responsible for:

- Processing REST API requests.
- Handling WebSocket connections and events.
- Executing background tasks.
- Processing auction-related events.
- Integrating with other AWS services.

Using AWS Lambda enables the application to adopt an event-driven architecture while improving scalability and reducing infrastructure management efforts.

---

#### Amazon RDS and Amazon Aurora

During the initial deployment phase, **Amazon RDS for MySQL** is proposed as the primary relational database for storing application data such as users, products, auction sessions, and transaction history.

As the platform grows, **Amazon Aurora Serverless** is considered as an alternative database solution to provide improved performance, automatic scaling, and high availability while maintaining MySQL compatibility.

---

#### Amazon DynamoDB

Amazon DynamoDB is proposed for storing low-latency and real-time application data.

Potential use cases include:

- Auction session state.
- Active WebSocket connections.
- Bid history.
- Temporary event-processing data.

Combining relational databases with NoSQL storage allows the platform to leverage the strengths of both data models.

---

#### Amazon SQS

Amazon Simple Queue Service (Amazon SQS) is proposed to process bid requests through a messaging queue.

During high-traffic auction sessions, multiple users may submit bids simultaneously. Using a queue helps preserve request order, reduces processing conflicts, and improves system reliability.

---

#### Amazon EventBridge

Amazon EventBridge is proposed to support an event-driven architecture.

Application events such as auction creation, auction completion, or status changes can be published and delivered to different processing services without introducing tight coupling between system components.

---

#### Amazon Kinesis

Amazon Kinesis is considered as a future enhancement for collecting and processing high-volume streaming data.

Although it is not required within the current project scope, Kinesis may be adopted later for analytics, monitoring, or large-scale event processing.

---

#### Amazon Cognito

Amazon Cognito is proposed as the user authentication and authorization service.

The service supports:

- User registration.
- User authentication.
- JWT-based authorization.
- Session management.
- Password recovery.
- User profile management.

Using Cognito reduces authentication logic within the backend while leveraging AWS-managed security features.

---

#### Amazon CloudWatch

Amazon CloudWatch is proposed for monitoring application performance, collecting logs, and tracking AWS resource utilization.

CloudWatch enables developers to monitor system health, analyze operational metrics, and troubleshoot issues more efficiently.

---

#### AWS Identity and Access Management (IAM)

AWS Identity and Access Management (IAM) is proposed for managing users, roles, groups, and permissions across AWS resources.

Following the **Principle of Least Privilege** helps minimize unnecessary permissions while strengthening the overall security of the platform.

---

### 5. System Component Design

The Live Auction platform is designed using a layered architecture in which each component is responsible for a specific function. This design improves maintainability, scalability, and system modularity.

#### Frontend

The frontend is developed using **React** and **Vite**, providing the user interface for both customers and administrators.

Its primary responsibilities include:

- Displaying application data.
- Sending requests to backend APIs.
- Receiving real-time updates.
- Managing client-side state.
- Navigating between application features.

---

#### Backend

The backend is implemented using **FastAPI (Python)** following the RESTful architecture style.

Its major responsibilities include:

- Processing business logic.
- Managing auction sessions.
- Managing users and products.
- Validating incoming requests.
- Handling authentication and authorization.
- Communicating with databases and AWS services.

---

#### Database

The application requires persistent storage for business data.

In the proposed architecture, **Amazon RDS** is considered for relational data, while **Amazon DynamoDB** may be utilized for high-performance, low-latency data used by real-time auction features.

---

#### Storage

Static assets, including product images and frontend build artifacts, are proposed to be stored in **Amazon S3**.

Separating static content from backend services reduces server workload and improves delivery performance when integrated with Amazon CloudFront.

---

### 6. Implementation Plan

To ensure an organized development process, the project is planned to be implemented in multiple phases. Each phase focuses on a specific group of features and AWS services, allowing the team to gradually build, test, and expand the system while minimizing development risks.

#### Phase 1 – Core System Development

The first phase focuses on implementing the core functionality of the Live Auction platform, including:

- Developing the frontend using React and Vite.
- Implementing backend services using FastAPI.
- Designing the application database.
- Building REST APIs for both users and administrators.
- Implementing user authentication, product management, and auction management features.

During this phase, the development environment is prepared, and the AWS services considered for deployment are evaluated.

---

#### Phase 2 – AWS Deployment

After completing the core application, the team proposes deploying the platform to Amazon Web Services.

The planned deployment activities include:

- Hosting frontend applications on Amazon S3.
- Configuring Amazon CloudFront for global content delivery.
- Implementing user authentication with Amazon Cognito.
- Deploying backend APIs to AWS compute services.
- Configuring the database environment.
- Storing product images and static assets.
- Setting up monitoring and logging services.

---

#### Phase 3 – System Enhancement

Once the platform becomes stable, the architecture is expected to evolve with additional AWS services to improve scalability, availability, and operational efficiency.

Future enhancements may include:

- Migrating toward a serverless architecture.
- Implementing real-time auction communication using WebSocket.
- Integrating messaging services for bid processing.
- Strengthening security and monitoring capabilities.
- Building an automated CI/CD deployment pipeline.

---

### 7. Evaluation and Conclusion

The proposed architecture aims to provide a scalable, maintainable, and secure online auction platform by leveraging a wide range of AWS managed services.

Separating the system into independent components improves maintainability while allowing future scalability and feature expansion. In addition, considering AWS services during the design phase establishes a strong foundation for cloud-native application development.

This proposal serves as the initial architectural plan for the Live Auction project. During implementation, certain architectural decisions may be refined according to project requirements, deployment constraints, and technical considerations.

The final deployment architecture, infrastructure configuration, AWS service integration, and implementation details will be presented in **Chapter 5 – Workshop**.
