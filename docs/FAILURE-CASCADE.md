# FAILURE-CASCADE.md
## SwiftEats Monolith — World Cup Final Failure Cascade Analysis

**Prepared by:** SRE Team  
**Scenario:** India vs Pakistan World Cup Final, 8 PM IST  
**Promo:** 50% off all orders, push notification to 180 million users  
**System:** Single Node.js (t3.medium) + Single PostgreSQL (max_connections=100)

---

## Section 1: Traffic Simulation Math

### Notification Funnel

| Stage | Calculation | Result |
|---|---|---|
| Total users notified | Given | 180,000,000 |
| Click-through rate | 8% of 180M | 14,400,000 users |
| Concentrated spike window | Given | 60 seconds |
| Avg API calls per user in first 60s | App open + menu load + promo check + restaurant list = ~5 calls | 5 calls |
| **Peak RPS** | (14,400,000 × 5) / 60 | **1,200,000 RPS** |

> Even with 90% of requests bouncing at network edge (TCP handshake timeout, CDN miss, load balancer rejection) — the server still receives ~120,000 RPS.
> Even assuming only 1% of users succeed in connecting: **12,000 RPS** — still above the Node.js saturation threshold.

### Conservative Estimate (Used Throughout This Document)

For analysis, we use the **1% successful connection rate** scenario to reflect real-world TCP stack behavior under extreme load:

```
Peak effective RPS hitting Node.js = 12,000–15,000 RPS
```

This is the number that kills the monolith. Everything below is calculated against this figure.

---

## Section 2: Component Capacity Numbers

### 2.1 PostgreSQL Connection Pool

- **Hard limit:** `max_connections = 100`
- **Node.js default pg pool:** 10 connections per process (default `pg` library)
- **Effective available connections to app:** ~90 (subtract 10 for superuser/admin overhead)

**Pool Exhaustion Formula:**

```
Connections held at any moment =
  (non-payment RPS × avg_query_time_s) + (payment RPS × payment_hold_time_s)
```

Assumptions:
- 80% of requests are non-payment (browse, menu, search): avg query time = 0.02s (20ms)
- 20% of requests are payment: avg hold time = 1.1s (midpoint of 200ms–2000ms range)

```
At 500 RPS:
  non-payment connections = (500 × 0.80) × 0.02 = 8 connections
  payment connections     = (500 × 0.20) × 1.10 = 110 connections
  TOTAL = 118 connections > 100 ← POOL EXHAUSTED AT ~450 RPS
```

**Pool exhaustion RPS (payment-weighted):**

```
Solve: (RPS × 0.80 × 0.02) + (RPS × 0.20 × 1.10) = 100
        RPS × (0.016 + 0.22) = 100
        RPS × 0.236 = 100
        RPS = 423 RPS ← HARD LIMIT
```

> **The PostgreSQL connection pool exhausts at 423 RPS.** The system receives 12,000+ RPS. This is a 28× overload on the database alone.

### 2.2 Node.js Event Loop

- **Instance type:** t3.medium (2 vCPUs, 4GB RAM)
- **Single process — uses 1 vCPU effectively**
- **Benchmark saturation point:** ~12,000–15,000 RPS on t3.medium for I/O-bound Express workload
- **Behavior past saturation:** Event loop lag increases exponentially; callback queue grows unbounded

```
At 12,000 RPS:
  Event loop tick time = ~80ms (healthy = <10ms)
  P99 latency = 3,000ms+
  New connections start timing out

At 15,000 RPS:
  Callback queue depth: ~50,000 pending callbacks
  Memory consumption: heap grows ~200MB/s
  OOM kill within 60–90 seconds
```

### 2.3 Node.js Heap / Memory

- **Heap limit on t3.medium:** 4GB total RAM, Node.js default heap ~1.5GB (can be raised, but default configs apply)
- **Each queued request:** ~8–15KB in memory (headers, body buffer, active socket)

```
At 15,000 queued requests:
  Memory = 15,000 × 12KB = ~180MB for request buffers alone
  + response buffers + ORM objects + pg client pool objects
  OOM threshold: ~1,400MB heap consumed → process killed by OS
```

### 2.4 Synchronous Payment Call Amplification

- **Razorpay API latency:** 200ms (best case) to 2,000ms (worst case / under load)
- **During each payment call:** Node.js awaits the HTTP response synchronously in the request lifecycle
- **DB connection is held open** for the entire payment duration (connection not released until transaction commits)

```
At Razorpay 2000ms hold + 20% of traffic being payment:
  200 RPS payment traffic × 2.0s hold = 400 connection-seconds consumed per second
  = 400 connections needed (vs pool of 100)
  Pool exhausted by payment calls alone at 50 RPS payment traffic = 250 RPS total
```

> **Under degraded Razorpay conditions, pool exhausts at 250 RPS total throughput.**

### 2.5 Promo Code Race Condition

- **No distributed lock, no Redis, no atomic check-and-decrement**
- **Implementation:** `SELECT promo_uses WHERE code='WORLDCUP50'` → application logic → `UPDATE promo_uses SET count=count+1`
- **Race window:** Between SELECT and UPDATE, N concurrent requests all read the same count
- **Effect:** Promo budget can be overclaimed by 100–1,000× under spike load

```
At 1,000 concurrent promo-check requests:
  All 1,000 read count=0 (under limit)
  All 1,000 proceed to apply discount
  All 1,000 increment: count goes from 0 to 1,000
  Budget overrun: potentially ₹crores in unplanned discounts
```

### 2.6 Static Asset NIC Saturation

- **Static files (images, JS, CSS) served by Node.js process**
- **Average restaurant image:** 80KB; Average full page load: ~2MB static assets
- **t3.medium NIC:** 5 Gbps burst, but sustained ~1.56 Gbps (baseline bandwidth)

```
At 14,400,000 users loading app (2MB static each, within 60s):
  Data transfer = 14,400,000 × 2MB = 28.8 TB in 60 seconds
  Required bandwidth = 28.8TB / 60s = 480 GB/s = 3,840 Gbps

  t3.medium NIC capacity = 1.56 Gbps sustained
  Overload factor = 3,840 / 1.56 = 2,461×
```

> **The NIC is saturated instantaneously.** No request ever reaches Node.js if static assets are served from the same process/IP.

---

## Section 3: The Failure Cascade

### Failure 1: Static Asset NIC Saturation
**Severity:** CRITICAL  
**Triggers at:** T+0s, effectively 0 RPS to application (NIC saturated before app receives anything meaningful)

**What users see:**  
App loads spinner indefinitely. White screen. Images never appear. JS bundles fail to load — app is non-functional even before any API call is made.

**What happens technically:**  
The single IP's network interface is flooded with HTTP GET requests for `/static/images`, `/bundle.js`, `/app.css`. The kernel TCP receive buffer fills. New TCP SYN packets are dropped. Connection refused or timeout for all users — including those trying to place orders.

**Cascade:**  
→ Even users with cached static assets cannot reach the API because the same IP/port is saturated  
→ Triggers Failure 2 as the few connections that do get through pile up on Node.js

---

### Failure 2: PostgreSQL Connection Pool Exhaustion
**Severity:** CRITICAL  
**Triggers at:** 423 RPS (under normal Razorpay conditions) / 250 RPS (under degraded Razorpay)

**What users see:**  
`"Something went wrong. Please try again."` — every API call returns 500. Menu fails to load. Cannot add items to cart. Cannot checkout. Payment page hangs.

**What happens technically:**  
`pg` connection pool throws `Error: timeout exceeded when trying to connect`. All pending database queries queue in memory. Node.js request handlers never complete. Response callbacks stack up in the event loop. HTTP keep-alive connections pile up.

**The math:**
```
With 423 RPS arriving at the DB layer:
  Connection pool: 100/100 in use
  New requests: queued in Node.js memory
  Queue grows at: 423 requests/second
  After 10 seconds: 4,230 requests waiting for a DB connection
  Memory consumed: 4,230 × 12KB = ~50MB (manageable but growing)
  After 60 seconds: 25,380 requests queued → ~300MB → approaching OOM
```

**Cascade:**  
→ Node.js heap grows with queued callbacks  
→ Triggers Failure 3 (event loop saturation)  
→ Payment calls still holding connections → amplifies pool exhaustion

---

### Failure 3: Node.js Event Loop Saturation
**Severity:** CRITICAL  
**Triggers at:** 12,000–15,000 RPS to Node.js process

**What users see:**  
Total unresponsiveness. No HTTP response at all — not even an error page. Browser shows "ERR_CONNECTION_TIMED_OUT". Health check endpoint stops responding.

**What happens technically:**  
The Node.js event loop is single-threaded. When the callback queue depth exceeds the loop's processing capacity:
- `setImmediate` and `process.nextTick` callbacks pile up
- I/O callbacks (DB responses, HTTP responses) cannot be processed
- New incoming connections are accepted at TCP level but never processed at application level
- `libuv` thread pool (4 threads by default) saturates on DNS lookups and file I/O

```
Event loop lag progression:
  Healthy:      <5ms lag
  Degraded:     50–200ms lag (users notice slowness)
  Saturated:    500ms–2s lag (requests timing out)
  Critical:     >5s lag (process effectively frozen)
  OOM:          Process killed by OS at ~4GB heap
```

**Cascade:**  
→ Process OOM kill → complete downtime  
→ No graceful shutdown → in-flight payment transactions left in unknown state  
→ PostgreSQL connections not cleanly released → idle-in-transaction connections accumulate

---

### Failure 4: Synchronous Payment Call Amplification
**Severity:** HIGH  
**Triggers at:** ~250 RPS total (when Razorpay is under load and returning 2s responses)

**What users see:**  
Checkout hangs on "Processing payment..." for 30–60 seconds, then fails. Users retry → doubles the payment load → death spiral. Some users are charged but order not confirmed (partial transaction state).

**What happens technically:**  
Each payment request:
1. Opens DB transaction
2. Makes synchronous HTTP call to Razorpay (200–2000ms)
3. Holds DB connection for entire duration
4. Commits/rolls back only after Razorpay responds

When Razorpay itself is under load (it also serves many merchants during World Cup):
```
20 concurrent payment requests × 2000ms hold = 40 connection-seconds/second consumed
At 50 payment RPS: 50 × 2.0s = 100 connections held = pool exhausted by payments alone
Non-payment traffic: 0 connections available → all browse/menu requests fail
```

**The amplification loop:**
```
Payment slow → users retry → more payment requests → Razorpay more loaded → 
payments slower → more DB connections held → pool exhausted faster → 
non-payment requests fail → users rage-retry → RPS increases further
```

**Cascade:**  
→ DB pool exhaustion accelerates (Failure 2 worsens)  
→ Partial payment states create data integrity issues  
→ Manual reconciliation required post-incident (hours of engineering time)

---

### Failure 5: Promo Code Race Condition
**Severity:** HIGH  
**Triggers at:** >1 concurrent promo-check request (race exists from first concurrent user)

**What users see:**  
At first: Promo code works fine (overclaimed). Later: Either everyone gets discount (infinite overrun) or application crashes trying to handle inconsistent state.

**What happens technically:**  
```sql
-- Thread A reads:
SELECT uses FROM promos WHERE code='WORLDCUP50'; -- returns 0

-- Thread B reads (same millisecond):  
SELECT uses FROM promos WHERE code='WORLDCUP50'; -- returns 0

-- Both proceed (both see uses=0, limit=1,000,000)
-- Both update:
UPDATE promos SET uses = uses + 1 WHERE code='WORLDCUP50';

-- Result: uses=2, but 2 discounts granted when 1 was the intent
-- At 14,400,000 concurrent users: uses could reach 14,400,000
-- At ₹150 average discount per order: ₹2,160,000,000 (₹216 crore) in unplanned discounts
```

**No row-level locking** in the application code. No `SELECT FOR UPDATE`. No Redis atomic INCR.

**Cascade:**  
→ Financial loss from uncontrolled discount application  
→ If promo table rows lock under high write contention → additional DB latency → accelerates Failure 2  
→ Potential regulatory/audit issue if payments processed at incorrect amounts

---

## Section 4: The Incident Timeline

```
T+0s        Notification delivered to 180M users. Countdown begins.

T+0s–10s    Push notification services deliver to devices. 14.4M users see the
            notification. Finger hovering over phone. 

T+10s       First wave of users tap notification. App launch begins.
            DNS resolves to single IP. TCP handshakes start.

T+15s       Static asset requests flood the server.
            Node.js process begins serving bundle.js, CSS, images.
            NIC utilization: 40% and climbing.

T+20s       NIC utilization hits 100%. TCP SYN packets dropped.
            ~500,000 users cannot open the app at all.
            First "ERR_CONNECTION_TIMED_OUT" errors appear.

T+25s       2nd wave of users arrive. 8 million concurrent connection attempts.
            The few users who got through start making API calls.
            RPS at Node.js: ~3,000 RPS (only 1% of traffic penetrating NIC saturation).

T+30s       DB connection pool: 87/100 connections in use.
            Payment calls arriving: ~600/minute.
            Each payment holding DB connection for 200–2000ms.

T+35s       DB connection pool: 100/100 — EXHAUSTED.
            New API calls: queued in Node.js memory.
            First 500 errors returned to clients: "Something went wrong."

T+40s       Users see errors. Begin rage-retrying.
            RPS doubles from retry traffic: ~6,000 RPS.
            Node.js heap: 800MB and growing (queued callbacks).
            Event loop lag: 200ms.

T+45s       Promo code race condition fully active.
            ~50,000 concurrent promo checks reading same DB rows.
            Promo overclaim begins: 50,000 unintended discounts applied in this second.

T+50s       Event loop lag: 1,200ms.
            Health check endpoint stops responding (also blocked by event loop).
            Load balancer (none exists) — no failover possible.

T+55s       Node.js heap: 2.1GB.
            Payment callbacks from Razorpay arrive but cannot be processed.
            In-flight transactions: ~1,200 payments in unknown state.

T+60s       Node.js process OOM-killed by OS kernel.
            Server goes dark. All 14.4M users: complete failure.
            Razorpay receives no acknowledgment for 1,200 payments.

T+60s–5m   Engineers alerted via Pingdom/UptimeRobot (if configured).
            Manual SSH to server. Attempt to restart Node.js.
            Process restarts. Immediately receives backlog of retries.
            Crashes again within 15 seconds.

T+5m–15m   Engineers attempt to restart PostgreSQL. 
            Idle-in-transaction connections blocking.
            `SELECT pg_terminate_backend(pid)` run manually.
            Node.js restarted again. Crashes again — retry storm too large.

T+15m–30m  Decision made to block traffic at firewall level.
            Server accessible only to engineers.
            DB connections manually cleared. Promo table manually audited.
            Promo overclaim confirmed: est. 340,000 unintended discounts.

T+30m–45m  Engineering team discusses options:
            - Disable promo (business decision required)
            - Rate limit traffic (no infra to do this)
            - Roll back (nothing to roll back to — no versioning)
            Decision: Restart server with IP allow-list, gradually open traffic.

T+45m       Server back online with manual IP throttling via iptables.
            Order flow works for ~200 concurrent users.
            Promo code disabled by DB update.

T+1h        Traffic gradually opened. System stable at low load.
            Post-incident: 14.4M users saw failure. Promo window missed entirely.

T+1h–2h    Finance team calculates promo overclaim.
            Engineering writes incident notes.
            CTO and CEO briefed.
            Estimated revenue lost: 45 minutes × ₹4.2 crore/min = ₹189 crore.
            Plus: Brand damage, app store rating drop, social media storm.

T+2h        System declared "stable" at reduced capacity.
            Formal postmortem scheduled.
```

---

## Summary: Failure Trigger Points

| Failure | Severity | Trigger RPS | Time to Trigger |
|---|---|---|---|
| NIC Saturation (static assets) | CRITICAL | ~0 RPS (instantaneous) | T+15s |
| DB Pool Exhaustion | CRITICAL | 423 RPS | T+35s |
| Payment Call Amplification | HIGH | 250 RPS | T+40s |
| Event Loop Saturation | CRITICAL | 12,000–15,000 RPS | T+50s |
| Promo Race Condition | HIGH | >1 concurrent | T+45s |
| Process OOM / Total Failure | CRITICAL | 15,000 RPS / 4GB heap | T+60s |

**The system is fully dead within 60 seconds of notification delivery.**  
**Zero components survive the load. There is no graceful degradation.**
