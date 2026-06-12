# 🎟️ BookMyShow - Scalable Ticket Booking Platform

> A BookMyShow-style ticket booking platform designed to handle 5,00,000 concurrent users, guarantee zero double-bookings, maintain sub-500ms response times, and operate within a $2,000/month AWS budget.

---

# 📌 Project Overview

ShowTime is a high-scale event ticketing platform built for handling flash-sale scenarios such as:

- Coldplay Concert Tickets
- IPL Final Tickets
- Cricket World Cup Matches
- Major Music Festivals

The primary challenge is ensuring:

- Zero double-bookings
- High throughput during ticket launches
- Low latency under massive load
- Cost-effective infrastructure

This repository contains the architecture and design decisions for the system.

---

# 🎯 Business Requirements

## Functional Requirements

Users should be able to:

- Browse events
- View seat availability
- Select seats
- Make payments
- Receive booking confirmations
- View booking history

Administrators should be able to:

- Create events
- Configure venues
- Manage seat inventory
- Track bookings

---

# 🚨 Critical Constraints

## Constraint 1: 5 Lakh Concurrent Users

At exactly 12:00 PM:

- 500,000 users may attempt booking simultaneously.
- Majority target premium seats.
- Traffic spike lasts 5–15 minutes.

### Estimated Peak RPS

Assume:

- Each user generates approximately 3 requests.
- Requests spread over 60 seconds.

```text
Peak RPS
= (500,000 × 3) / 60
= 25,000 RPS
```

System must survive 25K+ requests/sec.

---

## Constraint 2: Zero Double Booking

### Definition

A double-booking occurs when:

```text
Seat A-12
Booked by User A
AND
Booked by User B
```

for the same event.

### Acceptable Double Bookings

```text
0
```

Even a single duplicate booking is considered a system failure.

---

## Constraint 3: AWS Budget = $2,000/month

Infrastructure must fit inside:

```text
$2,000/month
```

This prevents us from solving scale problems simply by adding servers.

Every architecture decision must balance:

- Performance
- Correctness
- Cost

---

# ⚖️ Engineering Tradeoffs

## Speed vs Correctness

Strong locking guarantees correctness but increases latency.

Examples:

- Table Lock → Safe but slow
- Row Lock → Better
- Distributed Lock → Fastest

---

## Scale vs Budget

Without budget limits:

```text
100+ EC2 instances
Large Redis Cluster
Massive DB Cluster
```

With $2K budget:

```text
6 API Servers
1 Primary DB
2 Read Replicas
3 Redis Nodes
SQS
```

Efficiency becomes mandatory.

---

## Correctness vs Cost

Distributed locking is more scalable but requires additional infrastructure.

Database locking is cheaper but bottlenecks earlier.

---

# 🏗 Architecture Goals

## Target Metrics

| Metric | Target |
|----------|----------|
| Concurrent Users | 500,000 |
| API Response Time | <500ms |
| Double Bookings | 0 |
| Booking Success | >99.9% |
| AWS Budget | <$2,000 |

---

# 📂 Repository Structure

```text
bookmy-show/
│
├── README.md
│
├── docs/
│   ├── SCHEMA.md
│   ├── CONCURRENCY.md
│   ├── CACHE.md
│   ├── QUEUE.md
│   ├── ARCHITECTURE.md
│   ├── DESIGN-DECISIONS.md
│   └── DESIGN-UPDATES.md
│
└── assets/
```

---

# 🗃 Database Design Overview

Detailed schema:

```text
docs/SCHEMA.md
```

Core entities:

- users
- venues
- events
- seats
- bookings
- booking_seats

---

## Why UUID?

Instead of:

```sql
id SERIAL
```

we use:

```sql
id UUID
```

Benefits:

- Prevent enumeration attacks
- Globally unique
- Supports distributed systems
- Client-side generation possible

---

## Why Version Column?

Seats contain:

```sql
version INTEGER
```

Used for:

- Optimistic Locking
- Conflict Detection
- Safe concurrent updates

---

## Why held_until?

When a user reaches payment:

```text
Seat Status = HELD
```

for:

```text
10 minutes
```

This prevents abandoned carts from blocking inventory forever.

---

# 🔒 Concurrency Strategy

Detailed analysis:

```text
docs/CONCURRENCY.md
```

---

## Options Evaluated

### Option A

PostgreSQL

```sql
SELECT ... FOR UPDATE
```

Pros:

- ACID guarantees
- Simple implementation

Cons:

- Connection pool bottleneck
- Deadlocks possible

---

### Option B

Redis Distributed Lock

```text
SETNX
```

Pros:

- Extremely fast
- Handles massive concurrency

Cons:

- Additional infrastructure
- Requires lock management

---

## Final Decision

### Hybrid Approach

Use:

```text
Redis SETNX
```

for seat reservation.

Use:

```text
PostgreSQL Transaction
```

for final booking confirmation.

Reason:

- High throughput
- Strong consistency
- Fits budget

---

# ⚡ Cache Strategy

Detailed design:

```text
docs/CACHE.md
```

---

## Cached Data

### Event Details

```text
Key:
event:{eventId}

TTL:
3600s
```

---

### Availability Count

```text
availability:{eventId}:{category}
```

TTL:

```text
30 seconds
```

Reason:

Balances freshness and DB load.

---

### Seat Map

```text
seatmap:{eventId}
```

TTL:

```text
24 hours
```

Reason:

Almost never changes.

---

## Not Cached

### Individual Seat Status

Never cached.

Reason:

Can create stale availability and race conditions.

---

# 📬 Async Order Processing

Detailed design:

```text
docs/QUEUE.md
```

---

## Why Async?

Payment gateways take:

```text
200ms - 2000ms
```

Holding DB connections that long causes pool exhaustion.

Instead:

```text
API
  ↓
Create Pending Booking
  ↓
Publish SQS Message
  ↓
Return 202 Accepted
```

---

## Queue Flow

```text
User
  ↓
API
  ↓
SQS
  ↓
Payment Worker
  ↓
Database
  ↓
Notification Service
```

---

## SQS Message

```json
{
  "bookingId": "uuid",
  "userId": "uuid",
  "eventId": 101,
  "seatIds": [12, 13],
  "totalAmount": 4999,
  "paymentToken": "token",
  "idempotencyKey": "uuid"
}
```

---

## Visibility Timeout

```text
30 seconds
```

Reason:

2× expected maximum payment processing duration.

---

## Dead Letter Queue

```text
Max Receive Count = 3
```

After 3 failures:

```text
Move message to DLQ
```

for investigation.

---

# 💰 Budget Breakdown

| Component | Monthly Cost |
|------------|------------|
| EC2 API Servers | $719 |
| PostgreSQL RDS | $262 |
| Read Replicas | $262 |
| Redis Cluster | $359 |
| ECS Workers | $73 |
| ALB + CloudFront + SQS | $165 |
| Total | $1,840 |

---

# 🎯 Final Architecture Decisions

| Area | Decision |
|--------|---------|
| Database | PostgreSQL |
| Primary Keys | UUID |
| Concurrency | Redis + PostgreSQL Hybrid |
| Cache | Redis |
| Queue | AWS SQS |
| Workers | ECS Fargate |
| Notifications | Async |
| Availability Cache TTL | 30s |
| Seat Hold Duration | 10 mins |
| Visibility Timeout | 30s |

---

# 🚀 Future Improvements

When scale exceeds current capacity:

- Redis Cluster Sharding
- Event-based Inventory Service
- CQRS Read Model
- Kafka instead of SQS
- Multi-region deployment
- Seat Reservation Service isolation

---

# 📖 Documentation

| Document | Purpose |
|-----------|-----------|
| SCHEMA.md | Database schema |
| CONCURRENCY.md | Locking strategy |
| CACHE.md | Cache design |
| QUEUE.md | Async order flow |
| ARCHITECTURE.md | System diagram |
| DESIGN-DECISIONS.md | Engineering rationale |
| DESIGN-UPDATES.md | Iteration history |

---

# 👨‍💻 Author

ShowTime Architecture Assignment

Designed for:

- 500,000 concurrent users
- Zero double bookings
- Sub-500ms API latency
- $2,000/month infrastructure budget
