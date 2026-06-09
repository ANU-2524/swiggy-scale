# COST-ESTIMATE.md
## SwiftEats Redesigned Architecture — AWS Monthly Cost Estimate

**Region:** ap-south-1 (Mumbai) — lowest latency for Indian users  
**Pricing basis:** AWS on-demand pricing as of 2024 (ap-south-1 region)  
**Currency:** USD for AWS costs; INR for business case  
**Exchange rate used:** 1 USD = ₹83

---

## Scenario A: Baseline (Normal Traffic — Steady State)

Normal daily traffic assumption:
- Average concurrent users: ~50,000
- Average RPS: ~2,000
- No major promotional events

### EC2 — Node.js Application Servers

| Item | Type | Count | Hourly Rate (USD) | Hours/Month | Monthly Cost |
|---|---|---|---|---|---|
| Node.js app servers | t3.large (2 vCPU, 8GB) | 4 | $0.0912 | 720 | **$262.66** |

```
Calculation: $0.0912 × 720h × 4 instances = $262.66/month
```

t3.large chosen over t3.medium: double the RAM (8GB vs 4GB), handles ~25,000 RPS per instance before saturation. 4 instances = 100,000 RPS theoretical max — ~50× headroom over normal load.

### RDS PostgreSQL

| Item | Type | Count | Hourly Rate (USD) | Hours/Month | Monthly Cost |
|---|---|---|---|---|---|
| Primary DB | db.r6g.large (2 vCPU, 16GB) | 1 | $0.240 | 720 | **$172.80** |
| Read Replica 1 | db.r6g.large (2 vCPU, 16GB) | 1 | $0.240 | 720 | **$172.80** |
| Read Replica 2 | db.r6g.large (2 vCPU, 16GB) | 1 | $0.240 | 720 | **$172.80** |
| RDS Storage | gp3 SSD, 500GB per instance | 3 | $0.138/GB-mo | 500GB | **$207.00** |

```
Calculation:
  Primary: $0.240 × 720 = $172.80
  Replica 1: $0.240 × 720 = $172.80
  Replica 2: $0.240 × 720 = $172.80
  Storage: $0.138 × 500GB × 3 instances = $207.00
  RDS Subtotal: $725.40/month
```

r6g.large chosen: memory-optimized instance class, Graviton2 processor (40% better price/performance vs x86 for DB workloads). 16GB RAM for PostgreSQL shared_buffers and work_mem.

### ElastiCache Redis

| Item | Type | Count | Hourly Rate (USD) | Hours/Month | Monthly Cost |
|---|---|---|---|---|---|
| Redis primary node | cache.r6g.large (2 vCPU, 13.07GB) | 1 | $0.166 | 720 | **$119.52** |
| Redis replica node | cache.r6g.large (2 vCPU, 13.07GB) | 1 | $0.166 | 720 | **$119.52** |

```
Calculation: $0.166 × 720h × 2 nodes = $239.04/month
```

2-node cluster (1 primary + 1 replica) for high availability. r6g.large: 13GB RAM comfortably fits session cache + menu cache + promo counters for 50,000 concurrent users.

### Application Load Balancer (ALB)

| Item | Rate | Quantity | Monthly Cost |
|---|---|---|---|
| ALB base cost | $0.022/hour | 720 hours | $15.84 |
| LCU (Load Balancer Capacity Units) | $0.008/LCU-hour | ~10 LCU avg | $57.60 |

```
Calculation:
  Base: $0.022 × 720h = $15.84
  LCU: 1 LCU ≈ 1 new connection/s + 1,000 active connections + 1 GB/hour processed
  Normal load: ~10 LCU estimated
  LCU cost: $0.008 × 10 LCU × 720h = $57.60
  ALB Total: $73.44/month
```

### CloudFront CDN

| Item | Rate | Quantity | Monthly Cost |
|---|---|---|---|
| Data transfer out (first 10TB) | $0.0085/GB (India PoP) | 5,000 GB/month | $42.50 |
| HTTPS request cost | $0.0120/10,000 requests | 50M requests/month | $60.00 |

```
Assumptions (normal traffic):
  50,000 daily active users × 2MB static assets × 50 sessions/month = 5TB/month
  50M requests = 50,000 users × 1,000 requests/month

Calculation:
  Transfer: $0.0085 × 5,000GB = $42.50
  Requests: $0.012 × (50,000,000/10,000) = $60.00
  CloudFront Total: $102.50/month
```

### SQS (Payment Queue)

| Item | Rate | Quantity | Monthly Cost |
|---|---|---|---|
| SQS Standard (first 1M free) | $0.40/million messages | 2M messages/month | **$0.40** |

```
Assumptions: 10,000 orders/day × 30 days = 300,000 orders/month
  Each order = ~6 SQS messages (enqueue, process, retry, DLQ check, result, ack) = 1.8M
  Rounded to 2M messages/month
Calculation: (2M - 1M free) × $0.40 = $0.40/month
```

### Payment Worker EC2

| Item | Type | Count | Hourly Rate (USD) | Hours/Month | Monthly Cost |
|---|---|---|---|---|---|
| Payment workers | t3.medium (2 vCPU, 4GB) | 2 | $0.0456 | 720 | **$65.66** |

```
Calculation: $0.0456 × 720h × 2 instances = $65.66/month
2 workers handle ~100 concurrent Razorpay calls comfortably at normal load.
```

### PgBouncer

| Item | Type | Count | Hourly Rate (USD) | Hours/Month | Monthly Cost |
|---|---|---|---|---|---|
| PgBouncer host | t3.small (2 vCPU, 2GB) | 1 | $0.0228 | 720 | **$16.42** |

```
Calculation: $0.0228 × 720h × 1 instance = $16.42/month
PgBouncer is CPU-light — t3.small is sufficient for connection pooling at normal load.
```

---

### Baseline Monthly Total

| Service | Monthly Cost (USD) |
|---|---|
| EC2 — Node.js (4× t3.large) | $262.66 |
| RDS PostgreSQL Primary (db.r6g.large) | $172.80 |
| RDS Read Replica 1 (db.r6g.large) | $172.80 |
| RDS Read Replica 2 (db.r6g.large) | $172.80 |
| RDS Storage (500GB × 3 instances) | $207.00 |
| ElastiCache Redis (2× cache.r6g.large) | $239.04 |
| ALB (base + LCU) | $73.44 |
| CloudFront (5TB transfer + 50M requests) | $102.50 |
| SQS (2M messages) | $0.40 |
| Payment Workers (2× t3.medium) | $65.66 |
| PgBouncer (1× t3.small) | $16.42 |
| **BASELINE TOTAL** | **$1,485.52/month** |

**≈ ₹1,23,298/month (~₹1.23 lakh/month)**

---

## Scenario B: Peak Event (World Cup Night — 4-Hour Window)

**Pre-scale plan:** Scale to 20 Node.js instances at T-30 minutes. Auto-scale cap at 40 instances.  
**Event window:** 4 hours (8 PM – 12 AM IST)  
**Traffic assumption:** 14.4M users, 1,200,000 peak RPS (raw), ~60,000 RPS effective to ALB after CDN

### Additional EC2 — Scaled Up Node.js Instances

| Item | Type | Extra Instances Above Baseline | Hourly Rate | Hours | Extra Cost |
|---|---|---|---|---|---|
| Additional Node.js | t3.large | 16 (from 4 → 20) | $0.0912 | 4 | **$5.84** |
| Max auto-scale burst | t3.large | +20 more (to 40 max) | $0.0912 | 2 (avg) | **$3.65** |

```
Calculation:
  Pre-scale 16 extra instances: $0.0912 × 4h × 16 = $5.84
  Auto-scale burst (20 instances for avg 2h): $0.0912 × 2h × 20 = $3.65
  Extra EC2: $9.49 for the event window
```

### CloudFront Surge Transfer

| Item | Rate | Extra Volume | Extra Cost |
|---|---|---|---|
| Extra data transfer | $0.0085/GB | 28,800 GB (28.8 TB) | **$244.80** |
| Extra request cost | $0.0120/10,000 | 500M requests | **$600.00** |

```
Assumptions:
  14.4M users × 2MB static assets = 28.8TB transferred through CloudFront
  95% cache hit rate means CloudFront serves from edge (no origin egress for most)
  500M requests: 14.4M users × ~35 API + static requests each

Calculation:
  Transfer: $0.0085 × 28,800GB = $244.80
  Requests: $0.012 × (500,000,000/10,000) = $600.00
  Extra CloudFront: $844.80
```

### Additional Payment Workers

| Item | Type | Extra Count | Hourly Rate | Hours | Extra Cost |
|---|---|---|---|---|---|
| Extra payment workers | t3.large | 8 (from 2 → 10) | $0.0912 | 4 | **$2.92** |

```
Calculation: $0.0912 × 4h × 8 instances = $2.92
10 payment workers handle ~1,000 concurrent Razorpay calls.
```

### SQS Peak Messages

| Item | Rate | Extra Messages | Extra Cost |
|---|---|---|---|
| SQS (event night orders) | $0.40/million | 3M extra messages | **$1.20** |

```
Peak estimate: 500,000 orders in event window × 6 messages = 3M messages
Cost: 3M × $0.40 = $1.20
```

---

### Peak Event Extra Cost (4-Hour Window)

| Item | Extra Cost (USD) |
|---|---|
| Extra EC2 instances (Node.js scale-out) | $9.49 |
| CloudFront surge (28.8TB + 500M requests) | $844.80 |
| Extra payment workers (8× t3.large, 4h) | $2.92 |
| SQS peak message volume | $1.20 |
| **PEAK EVENT EXTRA TOTAL** | **$858.41** |

**≈ ₹71,248 for 4 hours of World Cup Final traffic**

---

## Combined Cost Summary

| Period | Cost (USD) | Cost (INR) |
|---|---|---|
| Monthly baseline (normal operations) | $1,485.52 | ₹1,23,298 |
| World Cup Final event window (4 hours) | $858.41 | ₹71,248 |
| **Total (month with one event)** | **$2,343.93** | **₹1,94,546** |

---

## Business Justification: ROI of This Architecture

### The Cost of Doing Nothing

From the failure cascade analysis, the monolith fails completely at T+60 seconds.  
Estimated recovery time: 45 minutes minimum.

```
Revenue loss during outage:
  Loss rate:   ₹4.2 crore/minute (given)
  Downtime:    45 minutes
  Total loss:  ₹4.2 crore × 45 = ₹189 crore ($22.8M)
```

### The Cost of the Redesigned Architecture

```
Monthly infrastructure cost:  ₹1,23,298/month  ($1,485/month)
Peak event extra cost:        ₹71,248           ($858)
Annual infrastructure cost:   ₹1,23,298 × 12 = ₹14,79,576/year ($17,826/year)
```

### The ROI Calculation

```
Revenue saved (one event, 45-min outage prevented):  ₹189,00,00,000
Annual infrastructure cost:                          ₹14,79,576
──────────────────────────────────────────────────────────────────
ROI (year 1):  ₹189 crore saved vs ₹14.8 lakh spent
               = 12,783% return on infrastructure investment

Even a single 10-minute outage prevented:
  ₹4.2 crore × 10 = ₹42 crore saved
  vs ₹1.23 lakh/month cost
  = 341× return in a single incident
```

**The redesigned infrastructure costs less per month than 30 seconds of downtime during a major event.**  
This is not a cost discussion — it is a risk management discussion.  
The question is not "can we afford this architecture?" — it is "can we afford not to build it before the World Cup Final?"

Additionally:
- **Brand damage** from 14.4M users seeing a broken app on a major event is immeasurable but significant (app store ratings, social media, churn)
- **Regulatory exposure** from failed payment states requires engineering reconciliation time (estimate: 200 engineering hours at ₹5,000/hour = ₹10 lakh)
- **Promo budget overrun** from race condition: estimated ₹50–216 crore in unintended discounts (from cascade analysis)
