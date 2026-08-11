---
title: "System Testing"
date: 2026-08-03
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

# SYSTEM TESTING

After deploying the infrastructure and AWS services, the team tested the **Live Auction** system to verify that its main functions operated correctly in the deployed environment.

The testing process covered account authentication and authorization, REST APIs, auction-management business logic, WebSocket connectivity, the real-time bidding flow, image uploads to Amazon S3, content delivery through Amazon CloudFront, and system security mechanisms.

## Testing Contents

1. [Overview and Test Environment](5.5.1-Overview-and-Test-Environment/)
2. [Test Methodology and Test Case Format](5.5.2-Test-Methodology-and-Test-Case-Format/)
3. [Authentication and Authorization Testing](5.5.3-Authentication-and-Authorization-Testing/)
4. [REST API and Auction Management Business Logic Testing](5.5.4-REST-API-and-Auction-Management-Business-Logic-Testing/)
5. [WebSocket Connection and Real-Time Update Testing](5.5.5-WebSocket-Connection-and-Real-Time-Update-Testing/)
6. [End-to-End Bidding Flow Testing](5.5.6-End-to-End-Bidding-Flow-Testing/)
7. [S3, Image Upload, and CloudFront Testing](5.5.7-S3-Image-Upload-and-CloudFront-Testing/)
8. [System Security Testing](5.5.8-System-Security-Testing/)