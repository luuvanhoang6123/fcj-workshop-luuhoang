---
title: "Worklog Week 7"
date: 2026-08-03
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Duration

**August 3, 2026 – August 7, 2026**

### Personal objectives

- Design and execute test cases for the real-time bidding flow (my focus area).
- Validate WebSocket, SQS FIFO, and DynamoDB integration on live AWS.
- Fix issues found; review IAM and Terraform before final walkthrough.

### Activities completed

| Day | Date | Work | Reference |
| --- | --- | --- | --- |
| Monday | 03/08/2026 | Built test matrix: auth, session CRUD, valid/invalid bids, WebSocket reconnect. | Team test doc |
| Tuesday | 04/08/2026 | Tested Cognito registration, email confirm, User vs Admin login; verified tokens on API calls. | [Cognito User Pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html) |
| Wednesday | 05/08/2026 | Tested REST Lambdas on AWS: create session, add item, submit bid; verified DynamoDB records. | [Lambda Testing](https://docs.aws.amazon.com/lambda/latest/dg/testing-guide.html) |
| Thursday | 06/08/2026 | Simulated 3 clients bidding in sequence; verified SQS FIFO order and WebSocket broadcast. | [WebSocket API](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html) |
| Friday | 07/08/2026 | Fixed CORS and wrong DynamoDB table env var; re-ran `terraform plan` after IAM tweak; end-to-end demo. | Source & Terraform |

### Results and notes

- **Pass:** Login → join session → bid → real-time UI update worked with 2–3 clients.
- **Pass:** Bids below current price rejected; bid history shown in correct time order.
- **Fixed:** CORS missing `Authorization` header; consumer Lambda used wrong `BIDS_TABLE` var.
- **Fixed:** WebSocket disconnect left stale connection IDs — added cleanup in `$disconnect` handler.
- I drafted Workshop section 5.5 (Testing) for scenarios I tested directly.
- **Remaining for Week 8:** S3/CloudFront image upload tests and personal final report.
