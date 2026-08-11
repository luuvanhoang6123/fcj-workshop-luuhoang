---
title: "Worklog Week 4"
date: 2026-07-13
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Duration

**July 13, 2026 – July 17, 2026**

### Personal objectives

- Join topic selection and auction business requirement analysis.
- Propose a solution for concurrent bids and real-time price updates.
- Draft architecture and implementation roadmap sections in the Proposal.

### Activities completed

| Day | Date | Work | Notes |
| --- | --- | --- | --- |
| Monday | 13/07/2026 | Team discussion; agreed on **Live Auction Platform on AWS**; I summarized MVP scope. | Team meeting |
| Tuesday | 14/07/2026 | Drew User/Admin use cases: registration, session creation, items, bidding, approval. | Internal doc |
| Wednesday | 15/07/2026 | Analyzed race conditions under concurrent bids; proposed SQS FIFO + WebSocket push. | [AWS Architecture](https://aws.amazon.com/architecture/) |
| Thursday | 16/07/2026 | Studied Well-Architected; mapped Cognito, API Gateway, Lambda, DynamoDB, S3, CloudFront. | [Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) |
| Friday | 17/07/2026 | Finalized architecture diagram; proposed Terraform for IaC; I wrote the "Bidding flow" Proposal section. | [Terraform AWS](https://developer.hashicorp.com/terraform/tutorials/aws-get-started) |

### Results and notes

- I documented the bidding flow: client bid → API Gateway → SQS FIFO → Lambda → DynamoDB → WebSocket broadcast.
- Defined dual frontends (User/Admin) and Cognito-based authorization.
- Mentor feedback: clarify auction session timeout — added to Week 5 backlog.
- **Personal role this week:** Business analysis + real-time flow architecture (distinct from frontend work owned by others).
