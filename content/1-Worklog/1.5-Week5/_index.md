---
title: "Worklog Week 5"
date: 2026-07-20
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Duration

**July 20, 2026 – July 24, 2026**

### Personal objectives

- Build serverless backend: Lambda handlers and DynamoDB schema.
- Implement bid logic, bid history, and input validation.
- Integrate APIs with frontend built by teammates.

### Activities completed

| Day | Date | Work | Reference |
| --- | --- | --- | --- |
| Monday | 20/07/2026 | Aligned repo structure (`Infrastructure/`, `backend/`, `frontend/`); I owned auction and bid modules. | Team docs |
| Tuesday | 21/07/2026 | Designed DynamoDB tables: Users, Categories, Sessions, Items, Bids; chose partition/sort keys for queries. | [DynamoDB Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) |
| Wednesday | 22/07/2026 | Wrote REST Lambdas for session/item CRUD; validated starting price and end time. | [FastAPI Tutorial](https://fastapi.tiangolo.com/tutorial/) |
| Thursday | 23/07/2026 | Implemented bid handler pushing to SQS FIFO; drafted consumer Lambda updating highest price. | Team source |
| Friday | 24/07/2026 | Postman tests with staging frontend; fixed response format; listed missing APIs for Week 6. | Internal test doc |

### Results and notes

- Completed Lambda skeletons for sessions, items, and bid intake.
- DynamoDB schema supports session queries and time-ordered bid history.
- Local bid flow ran end-to-end before AWS deploy (mock queue).
- **Challenge:** Ensuring bids exceed current price — solved with DynamoDB conditional writes.
- **Team coordination:** Frontend integrated list/detail APIs; I helped fix basic CORS on API Gateway mock.
