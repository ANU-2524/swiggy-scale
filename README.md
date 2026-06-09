# SwiftEats Scale Simulation
### Can a single Node.js server survive 14 million users in 60 seconds? (Spoiler: it dies in one.)

---

## The Scenario

**Event:** India vs Pakistan World Cup Final, 8 PM IST  
**Promo:** 50% off all orders, push notification to **180 million users**  
**Expected spike:** ~8% CTR = **14.4 million users in 60 seconds**

**Current system:**  
- 1 Node.js Express process (t3.medium, 1 vCPU, 4GB RAM)  
- 1 PostgreSQL instance (`max_connections = 100`)  
- No cache, no CDN, no load balancer, no replicas, no queue  

**The question:** Can this survive a World Cup Final? What do we build if not?

This repository answers that question with numbers, not opinions.

---

## Documents

| Document | What It Contains |
|---|---|
| [`docs/FAILURE-CASCADE.md`](docs/FAILURE-CASCADE.md) | Full failure cascade: peak RPS math, per-component capacity limits, 5 named failures with trigger RPS, and a T+0s to T+2h incident timeline |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Current monolith annotated with failure points → redesigned 8-component AWS architecture with ASCII diagrams and ADRs |
| [`docs/COST-ESTIMATE.md`](docs/COST-ESTIMATE.md) | Line-item AWS cost estimate (real instance types, real pricing) for baseline steady state and World Cup night peak event, with ROI business case |
| [`docs/RUNBOOK.md`](docs/RUNBOOK.md) | 5-step incident runbook: 8 CloudWatch alarms, triage decision tree, component-specific CLI commands, rollback criteria, and postmortem template |

---

## Key Findings

- **The DB pool exhausts at 423 RPS** — the system receives 12,000+ RPS effective load. That's a 28× overload on the hardest constraint alone.
- **The server dies at T+60 seconds.** All 14.4 million users see a broken app. Not degraded — broken. The Node.js process is OOM-killed by the OS.
- **The promo race condition could cost ₹216 crore** — with no atomic locking, every one of 14.4 million users could apply the promo simultaneously. `SELECT → UPDATE` without `FOR UPDATE` is not a discount; it's a financial exploit.
- **Synchronous Razorpay calls kill the DB before compute does** — at 250 RPS with Razorpay returning 2s responses, payment calls alone exhaust the entire connection pool. Non-payment traffic gets zero connections.
- **The NIC saturates before a single API call lands** — serving static assets from the same Node.js process means 2,461× NIC overload at T+15 seconds. The app never even responds.

---

## Architecture Overview

The redesigned system routes static assets through **CloudFront CDN** (95% cache hit rate — Node.js never sees an image request), distributes API traffic via an **Application Load Balancer** across an **auto-scaling group of 20 pre-scaled Node.js instances**, uses **Redis** for cache and atomic promo-code locking, routes payments through **SQS** to decouple Razorpay latency from DB connections, and fronts PostgreSQL with **PgBouncer** and **two read replicas** to multiply the effective connection pool by 4×.

The monolith's single point of failure becomes a system with no single component that can cause total failure.

---

## Tech Stack Context

| Technology | Role in Analysis |
|---|---|
| **Node.js / Express** | Monolith runtime — single-process event loop saturation modelled at 12,000–15,000 RPS |
| **PostgreSQL** | Bottleneck component — `max_connections=100` is the system's hardest limit |
| **Redis (ElastiCache)** | Cache layer + atomic promo-code locking via `INCR` |
| **AWS EC2 / ASG** | Application compute — t3.large instances with auto-scaling |
| **AWS RDS** | Managed PostgreSQL with read replicas (db.r6g.large) |
| **AWS CloudFront** | CDN — eliminates static asset NIC saturation |
| **AWS ALB** | Load balancer — health checks, SSL termination, rate limiting |
| **AWS SQS** | Payment queue — decouples Razorpay from request lifecycle |
| **PgBouncer** | Connection pooler — multiplexes app connections onto DB |

---

## Cost Summary

| Mode | Monthly Cost |
|---|---|
| Baseline (steady state) | $1,485 / month (~₹1.23 lakh) |
| World Cup Final extra (4-hour window) | $858 (~₹71,248) |
| **One 45-minute outage costs** | **₹189 crore** |

The redesigned infrastructure costs less per month than **30 seconds of downtime** during a major event.
