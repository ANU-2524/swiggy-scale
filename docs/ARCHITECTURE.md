# ARCHITECTURE.md
## SwiftEats — System Architecture: Monolith vs Redesigned Multi-Tier

---

## Section 1: Current Architecture (The Monolith)

```
                        INTERNET
                           │
                    180M users
                    14.4M spike
                           │
                     ┌─────▼─────┐
                     │  Single   │ ← ⚠️  No load balancer.
                     │   IP /    │      Single point of failure.
                     │   DNS     │      One IP absorbs all traffic.
                     └─────┬─────┘
                           │
              ┌────────────▼────────────┐
              │   Node.js Express       │ ← ⚠️  Failure 3: Event loop saturates
              │   Single Process        │      at 12,000–15,000 RPS.
              │   t3.medium             │      Single-threaded. One CPU used.
              │   1 vCPU, 4GB RAM       │ ← ⚠️  Failure 6: OOM kill at 4GB heap.
              │                         │      Process dies at T+60s.
              │   + Serves static files │ ← ⚠️  Failure 5: NIC saturates at T+15s.
              │     (images, JS, CSS)   │      2,461× overload on NIC.
              │                         │
              │   + Synchronous         │ ← ⚠️  Failure 4: Each payment holds
              │     Razorpay calls      │      DB connection 200–2000ms.
              │     inline in request   │      Pool exhausted at 250 RPS.
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │   PostgreSQL            │ ← ⚠️  Failure 1: Pool exhausted at
              │   Single Instance       │      423 RPS. Hard limit: 100 connections.
              │   max_connections = 100 │
              │   No replicas           │ ← ⚠️  Single DB handles ALL reads and
              │   No connection pooler  │      writes. Read replicas = 0.
              │                         │
              │   Promo table:          │ ← ⚠️  Failure 2: Race condition.
              │   No row locking        │      SELECT → UPDATE without
              │   No atomic ops         │      FOR UPDATE or Redis INCR.
              └─────────────────────────┘

              No Redis. No CDN. No queue. No autoscaling. No replicas.
```

**Hard limits of this system:**
- Max sustainable RPS: ~423 (DB bottleneck)
- Max burst before OOM: ~15,000 RPS (Node.js bottleneck)
- Time to total failure under World Cup load: **60 seconds**
- Estimated users served before failure: **~25,000 out of 14,400,000**

---

## Section 2: Redesigned Architecture

```
                              INTERNET
                                 │
                         180M push notification
                         14.4M users tapping
                                 │
              ┌──────────────────▼──────────────────────┐
              │            CloudFront CDN                │
              │         (AWS Global Edge Network)        │
              │                                          │
              │  Caches: All static assets               │
              │    - /static/* images, icons  TTL: 24h   │
              │    - /bundle.js, /app.css     TTL: 1y    │
              │    - /manifest.json           TTL: 1h    │
              │    - Restaurant menu JSONs    TTL: 60s   │
              │                                          │
              │  Cache Hit Rate Target: 95%+             │
              │  Only 5% of requests reach origin        │
              │  720,000 users → ~36,000 to origin       │
              └──────────────────┬──────────────────────┘
                                 │ ~36,000 users (cache miss + API calls)
                                 │
              ┌──────────────────▼──────────────────────┐
              │     Application Load Balancer (ALB)      │
              │                                          │
              │  - SSL/TLS termination (offload crypto)  │
              │  - Health checks every 10s               │
              │  - Sticky sessions (off — stateless API) │
              │  - Rate limiting: 100 req/s per IP       │
              │  - WAF rules: block scraping, DDoS       │
              │  - Round-robin to healthy Node instances │
              └──────────────────┬──────────────────────┘
                                 │
              ┌──────────────────▼──────────────────────┐
              │         Auto-Scaling Group               │
              │      Node.js EC2 Instances (t3.large)    │
              │                                          │
              │  Baseline:   4 instances  (normal day)   │
              │  Pre-scaled: 20 instances (event day)    │
              │  Max:        40 instances (auto-scale)   │
              │                                          │
              │  Scale-out trigger: CPU > 60% for 60s   │
              │  Scale-in trigger:  CPU < 20% for 300s  │
              │                                          │
              │  Each instance:                          │
              │  - Stateless (no local session storage)  │
              │  - Does NOT serve static files           │
              │  - Does NOT make sync payment calls      │
              │  - Reads: → Redis first, then DB replica │
              │  - Writes: → PostgreSQL primary only     │
              │  - Payments: → SQS queue (async)         │
              └──────┬───────────────────┬───────────────┘
                     │                   │
          ┌──────────▼──────┐   ┌────────▼──────────────┐
          │   Redis Cluster  │   │  SQS Payment Queue    │
          │  (ElastiCache)   │   │                       │
          │                  │   │  - Decouples payment  │
          │  Caches:         │   │    from request cycle │
          │  - Session data  │   │  - FIFO queue         │
          │    TTL: 30min    │   │  - Visibility timeout:│
          │  - Menu JSON     │   │    30s (retry-safe)   │
          │    TTL: 60s      │   │  - Dead letter queue  │
          │  - Restaurant    │   │    after 3 retries    │
          │    list TTL: 30s │   │  - Processes ~500     │
          │  - Promo check   │   │    payments/sec       │
          │    TTL: 5s       │   └────────┬──────────────┘
          │                  │            │
          │  Promo Race Fix: │   ┌────────▼──────────────┐
          │  INCR promo_uses │   │  Payment Worker       │
          │  atomic in Redis │   │  Service              │
          │  (no SELECT/     │   │  (EC2 Auto-Scale)     │
          │   UPDATE race)   │   │                       │
          └──────────────────┘   │  - Reads from SQS     │
                                 │  - Calls Razorpay API │
          ┌──────────────────┐   │  - Writes result to   │
          │    PgBouncer     │   │    DB (success/fail)  │
          │  (Connection     │   │  - Updates order      │
          │   Pooler)        │   │    status             │
          │                  │   │  - Notifies user via  │
          │  - Pool mode:    │   │    WebSocket / push   │
          │    Transaction   │   └───────────────────────┘
          │  - Pool size:    │
          │    20 per app    │            │
          │    instance      │            │
          │  - Max server    │            │
          │    connections:  │            │
          │    400 (4× 100)  │            │
          └────────┬─────────┘            │
                   │                      │
        ┌──────────▼──────────────────────▼───────────┐
        │          PostgreSQL Cluster                  │
        │                                              │
        │  Primary (r6g.large)                         │
        │  - All writes (orders, users, payments)      │
        │  - max_connections = 200                     │
        │  - PgBouncer fronts it: effective 400 pool   │
        │                                              │
        │  Read Replica 1 (r6g.large)  ──────────────►│
        │  - Menu queries                              │
        │  - Restaurant listings                       │
        │  - Search queries                            │
        │                                              │
        │  Read Replica 2 (r6g.large)  ──────────────►│
        │  - User profile reads                        │
        │  - Order history reads                       │
        │  - Analytics queries                         │
        └──────────────────────────────────────────────┘
```

### Traffic Flow Summary

```
Request Type          Path
──────────────────────────────────────────────────────────────────
Static assets         CloudFront edge → user (95% cache hit)
Menu JSON             CloudFront (60s TTL) → Redis (60s TTL) → DB replica
Restaurant list       CloudFront (30s TTL) → Redis (30s TTL) → DB replica
User session check    Redis (30min TTL) → DB primary if miss
Promo code check      Redis INCR (atomic) → no DB roundtrip
Order placement       Node.js → DB primary write
Payment initiation    Node.js → SQS enqueue (instant return to user)
Payment processing    Payment worker → Razorpay → DB write → push notification
```

---

## Section 3: Component Justification Table

| Component | Failure It Prevents | How It Prevents It |
|---|---|---|
| **CloudFront CDN** | Failure 5: NIC Saturation | Serves all static assets (images, JS, CSS) from 450+ edge PoPs worldwide. Node.js never receives a single image request. Offloads 95%+ of raw bandwidth. TTL-based caching means even menu data is served from edge. |
| **Application Load Balancer (ALB)** | Failure 3: Single-point-of-failure / no failover | Distributes traffic across all healthy Node.js instances. Performs health checks every 10s — unhealthy instances removed in <30s. Rate limiting (100 req/s per IP) prevents any single user or bot from monopolizing connections. SSL termination offloads CPU-intensive crypto from app servers. |
| **Auto-Scaling Group (Node.js)** | Failure 3: Event loop saturation | Horizontal scaling means load is distributed. 20 instances pre-scaled for the event each handle ~600 RPS — well under the 12,000 RPS single-process limit. Scale-out trigger at 60% CPU adds capacity before saturation. |
| **Redis Cluster (ElastiCache)** | Failure 1: DB pool exhaustion (read pressure) | Caches all read-heavy data (menus, restaurant lists, sessions). Reduces DB read traffic by ~80%. With 95% cache hit rate, DB receives only write traffic and cache-miss reads — manageable within pool limits. |
| **Redis INCR for Promo** | Failure 2: Promo code race condition | Redis `INCR` is atomic and single-threaded. `INCR promo_uses` is an all-or-nothing operation — no race window between read and write. Promo budget enforced exactly, regardless of concurrency. |
| **PgBouncer (Connection Pooler)** | Failure 1: DB pool exhaustion | PgBouncer in transaction pooling mode multiplexes many app connections onto fewer DB connections. 20 app instances × 20 PgBouncer connections = 400 app-side connections → multiplexed to 200 actual DB connections. Eliminates pool exhaustion under normal load. |
| **PostgreSQL Read Replicas (×2)** | Failure 1: DB pool exhaustion (write contention) | Offloads all read traffic to replicas. Primary handles only writes. Reduces primary connection usage by ~80%. Each replica independently scalable. Provides read redundancy — replica failure doesn't cause data loss. |
| **SQS Payment Queue** | Failure 4: Synchronous payment call amplification | Decouples payment from the request lifecycle entirely. Node.js enqueues payment intent and returns HTTP 202 immediately. DB connection released in <5ms. Razorpay latency (200–2000ms) no longer holds DB connections or event loop. |
| **Payment Worker Service** | Failure 4: Synchronous payment call amplification | Dedicated service reads from SQS and calls Razorpay. Independently scalable. Slow Razorpay responses only affect payment workers — not the main API. Dead letter queue handles Razorpay failures without losing payment data. |
| **PostgreSQL Primary (r6g.large)** | Failure 1: DB capacity | r6g.large has 2 vCPUs, 16GB RAM — far more headroom than t3.medium. max_connections raised to 200. With PgBouncer, effective pool is 400+. |

---

## Architecture Decision Records (ADR)

### ADR-001: Async Payment via SQS over Sync HTTP
**Decision:** All payment calls go to SQS queue, processed by separate payment workers.  
**Rationale:** The synchronous payment model was the single biggest contributor to DB pool exhaustion. A ₹1,000 crore scale event cannot have payment latency dictate DB connection availability for 14M users. SQS provides guaranteed delivery, retry logic, and complete decoupling.  
**Trade-off:** Payment confirmation is now async — users see "Payment processing" instead of instant confirmation. Acceptable UX for 99% of users. Mitigated with WebSocket/push notification for confirmation.

### ADR-002: Redis INCR for Promo Code over DB SELECT/UPDATE
**Decision:** Promo redemption counter maintained exclusively in Redis using atomic INCR.  
**Rationale:** PostgreSQL row-level locking under 14M concurrent requests causes lock contention that amplifies pool exhaustion. Redis single-threaded INCR has O(1) complexity and zero race window. Promo budget overrun from the race condition could cost ₹216 crore in a single minute.  
**Trade-off:** Redis is not a system of record — DB is eventually synced. Acceptable because promo counts are eventually consistent; exact real-time count is not a business requirement.

### ADR-003: Pre-scale to 20 Instances Before Event
**Decision:** Scheduled scale-out to 20 instances at T-30 minutes before notification send.  
**Rationale:** Auto-scaling reaction time is 3–5 minutes (CloudWatch alarm → scale action → instance boot → health check). At 14M users in 60 seconds, 3 minutes of under-capacity = 3 minutes of failures. Pre-scaling eliminates the reaction gap for a known event.
