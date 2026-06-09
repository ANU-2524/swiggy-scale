# RUNBOOK.md
## SwiftEats Incident Response Runbook — World Cup Final

**Version:** 1.0  
**Last Updated:** Pre-event review  
**Owner:** SRE Team  
**Applicable To:** Any on-call engineer, including first-year engineers  
**Escalation:** #incident-response (Slack) → SRE Lead → CTO  

> **How to use this runbook:** Start at STEP 1. Follow every step in order. Do not skip steps. Every command here is safe to run. If in doubt about anything, call the on-call SRE lead before acting. The only forbidden action is modifying the database schema — see STEP 4.

---

## STEP 1 — DETECT: CloudWatch Alarms

The following alarms MUST be configured before the event. Each alarm is listed with: metric, threshold, alarm type, and what it means.

| # | Alarm Name | Metric | Threshold | Type | What It Means |
|---|---|---|---|---|---|
| 1 | `SwiftEats-DB-Connections-Warning` | `DatabaseConnections` (RDS) | > 150 for 2 min | **WARNING** | DB connection pool under pressure. Pre-failure state. Act now before pool exhausts. |
| 2 | `SwiftEats-DB-Connections-Critical` | `DatabaseConnections` (RDS) | > 185 for 1 min | **CRITICAL** | DB pool near exhaustion (200 max). Users getting 500 errors. Immediate action required. |
| 3 | `SwiftEats-EC2-CPU-High` | `CPUUtilization` (EC2 ASG) | > 70% for 3 min | **WARNING** | Application servers CPU saturating. Auto-scale should trigger. Verify it is scaling. |
| 4 | `SwiftEats-Redis-Memory-High` | `DatabaseMemoryUsagePercentage` (ElastiCache) | > 80% for 5 min | **WARNING** | Redis running low on memory. Cache evictions starting. DB load will increase. |
| 5 | `SwiftEats-ALB-5xx-Rate` | `HTTPCode_ELB_5XX_Count` (ALB) | > 1% of requests for 2 min | **CRITICAL** | Application returning errors at scale. Users impacted. Triage immediately. |
| 6 | `SwiftEats-ALB-Latency-P99` | `TargetResponseTime` P99 (ALB) | > 3000ms for 2 min | **WARNING** | P99 latency > 3s. Event loop likely saturating. Check Node.js instance count. |
| 7 | `SwiftEats-SQS-Queue-Depth` | `ApproximateNumberOfMessagesVisible` (SQS) | > 10,000 for 5 min | **WARNING** | Payment queue backing up. Payment workers under-scaled. Users waiting for payment confirmation. |
| 8 | `SwiftEats-EC2-HealthyHosts` | `HealthyHostCount` (ALB Target Group) | < 3 for 1 min | **CRITICAL** | Instances failing health checks and being removed from rotation. Capacity dropping. |

### Where to See These Alarms
1. AWS Console → CloudWatch → Alarms  
2. Slack: All alarms auto-post to `#cloudwatch-alerts`  
3. PagerDuty: CRITICAL alarms page on-call immediately  

---

## STEP 2 — TRIAGE: Decision Tree

> Target: Identify root cause within 30 seconds of alarm firing.

```
ALARM FIRES
│
├─► Is Alarm #5 (5xx rate > 1%) firing?
│   YES ──► Go to CHECK A below
│   NO  ──► Is Alarm #6 (P99 > 3s) firing?
│            YES ──► Go to CHECK B below
│            NO  ──► Monitor. Not yet user-impacting.
│
CHECK A: Open ALB → Target Groups → Healthy Hosts count
│
├─► Healthy Hosts < baseline? (e.g., 3 of 20)
│   YES ──► Instances crashing. Go to RESPOND 3.1 (Compute Saturation)
│   NO  ──► All instances healthy but returning 5xx?
│            ──► Go to CHECK C below
│
CHECK B: Open CloudWatch → SwiftEats-DB-Connections-Critical
│
├─► DB connections > 185?
│   YES ──► DB pool exhausted. Go to RESPOND 3.2 (DB Pool Exhaustion)
│   NO  ──► DB connections normal?
│            ──► Go to CHECK D below
│
CHECK C: Open RDS → DB Connections metric (last 5 min)
│
├─► DB connections > 185?
│   YES ──► DB is the bottleneck. Go to RESPOND 3.2
│   NO  ──► DB connections normal?
│            ──► Check Redis hit rate (ElastiCache → CacheHits vs CacheMisses)
│            ──► Cache miss rate > 50%? Go to RESPOND 3.4 (Redis Cache Miss Spike)
│            ──► Cache hit rate normal? → Escalate to SRE Lead. Unknown failure mode.
│
CHECK D: Open SQS Console → Payment Queue → Messages available
│
├─► Queue depth > 10,000?
│   YES ──► Payment workers under-scaled. Go to RESPOND 3.3 (Payment Queue Backup)
│   NO  ──► All metrics normal but latency high?
│            ──► Check EC2 CPU utilization. > 70%? → Auto-scale may be lagging.
│            ──► Manually trigger scale-out: Go to RESPOND 3.1
```

---

## STEP 3 — RESPOND: Component-Specific Action Steps

---

### 3.1 — Compute Saturation (Node.js Instances Crashing or CPU > 70%)

**Symptoms:** ALB healthy host count dropping, EC2 CPU alarms, P99 > 3s

**Step 1: Check current instance count**
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names SwiftEats-ASG \
  --query "AutoScalingGroups[0].{Desired:DesiredCapacity,Min:MinSize,Max:MaxSize,Running:Instances[?LifecycleState=='InService'] | length(@)}" \
  --output table \
  --region ap-south-1
```

**Step 2: Manually increase desired capacity (do not wait for auto-scale)**
```bash
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name SwiftEats-ASG \
  --desired-capacity 30 \
  --region ap-south-1
```
> Replace 30 with your target. Max cap is 40. Go to 30 first, then 40 if needed.

**Step 3: Watch instances come up (refresh every 30s)**
```bash
watch -n 30 'aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names SwiftEats-ASG \
  --query "AutoScalingGroups[0].Instances[*].{ID:InstanceId,State:LifecycleState,Health:HealthStatus}" \
  --output table \
  --region ap-south-1'
```

**Step 4: Verify in ALB**
- AWS Console → EC2 → Load Balancers → SwiftEats-ALB → Target Groups → SwiftEats-TG
- Watch "Healthy" count increase. New instances appear in ~3 minutes (boot + health check).

**What success looks like:**  
- Healthy host count ≥ 20  
- ALB P99 latency drops below 1,000ms within 5 minutes of new instances registering  
- 5xx rate drops below 0.1%  

**Who to notify:** Post in `#incident-response`: "Scaled Node.js from X to 30 instances. Monitoring recovery."  
**Escalate if:** 5xx rate does not drop within 10 minutes of new instances registering.

---

### 3.2 — DB Pool Exhaustion (RDS Connections > 185)

**Symptoms:** `SwiftEats-DB-Connections-Critical` alarm, 500 errors on all API calls, users cannot browse or checkout.

**Step 1: Check which queries are holding connections**
```sql
-- Run this in RDS Query Editor or psql session:
SELECT pid, state, wait_event_type, wait_event, query_start,
       now() - query_start AS duration, left(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC
LIMIT 20;
```

**Step 2: Kill long-running idle-in-transaction connections (safe to run)**
```sql
-- Kill connections idle in transaction for more than 30 seconds:
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - query_start > interval '30 seconds';
```

**Step 3: Kill connections idle for more than 2 minutes (safe — these are stale)**
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '2 minutes'
  AND pid != pg_backend_pid();
```

**Step 4: Check PgBouncer pool status**
```bash
# SSH to PgBouncer instance:
ssh ec2-user@<pgbouncer-private-ip>

# Connect to PgBouncer admin console:
psql -h 127.0.0.1 -p 6432 -U pgbouncer pgbouncer

# Run inside psql:
SHOW POOLS;
SHOW CLIENTS;
SHOW SERVERS;
```
> Look for `pool_mode`, `sv_active` (active server connections), `cl_waiting` (clients waiting). If `cl_waiting` > 100, PgBouncer itself is a bottleneck — increase `max_client_conn` in `/etc/pgbouncer/pgbouncer.ini`.

**Step 5: If connections remain maxed — temporarily reduce Node.js pool size**
```bash
# Update environment variable on running instances via SSM Parameter Store:
aws ssm put-parameter \
  --name "/swifteats/prod/DB_POOL_SIZE" \
  --value "5" \
  --type String \
  --overwrite \
  --region ap-south-1
# Then trigger rolling restart of Node.js instances:
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name SwiftEats-ASG \
  --preferences MinHealthyPercentage=50 \
  --region ap-south-1
```

**What success looks like:**  
- `DatabaseConnections` metric drops below 150 within 3 minutes of killing stale connections  
- 5xx error rate drops below 0.5%  
- P99 latency returns below 1,000ms  

**Who to notify:** Post in `#incident-response` and `#db-oncall`  
**Escalate if:** Connections stay above 185 after killing stale connections. Page DBA.

---

### 3.3 — Payment Queue Backup (SQS Depth > 10,000)

**Symptoms:** `SwiftEats-SQS-Queue-Depth` alarm, users see "Payment processing..." for > 5 minutes, order status stuck in PENDING.

**Step 1: Check queue depth and consumer count**
```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.ap-south-1.amazonaws.com/<account-id>/swifteats-payments \
  --attribute-names ApproximateNumberOfMessagesVisible,ApproximateNumberOfMessagesNotVisible \
  --region ap-south-1
```
- `ApproximateNumberOfMessagesVisible` = waiting to be processed  
- `ApproximateNumberOfMessagesNotVisible` = currently being processed by workers  

**Step 2: Scale up payment workers**
```bash
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name SwiftEats-PaymentWorkers-ASG \
  --desired-capacity 20 \
  --region ap-south-1
```

**Step 3: Check Dead Letter Queue for failed payments**
```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.ap-south-1.amazonaws.com/<account-id>/swifteats-payments-dlq \
  --attribute-names ApproximateNumberOfMessagesVisible \
  --region ap-south-1
```
> If DLQ has messages, payments are failing (Razorpay errors). Do NOT delete DLQ messages. Page payment team.

**Step 4: Verify Razorpay status**
- Check https://status.razorpay.com for any active incidents  
- If Razorpay is degraded: SQS messages will retry automatically (up to 3 times with exponential backoff). Do not interfere. Inform customer support to communicate delays.

**What success looks like:**  
- Queue depth dropping (not growing)  
- `ApproximateNumberOfMessagesNotVisible` increasing (workers processing)  
- Queue depth returns to < 1,000 within 15 minutes  

**Who to notify:** Post in `#incident-response` and `#payments-oncall`  
**Escalate if:** DLQ receiving messages and Razorpay is not showing an incident. Payment engineer must investigate.

---

### 3.4 — Redis Cache Miss Spike

**Symptoms:** ElastiCache `CacheMisses` metric spikes, DB read connections increase, latency degrades despite normal compute.

**Step 1: Check Redis memory and eviction stats**
```bash
# SSH to Redis diagnostic instance or use ElastiCache console:
# Check in CloudWatch: ElastiCache → SwiftEats-Redis → Evictions metric
# Non-zero evictions = Redis evicting keys due to memory pressure
```

**Step 2: Identify which keys are being missed most (via application logs)**
```bash
# On any Node.js instance:
ssh ec2-user@<app-instance-ip>
sudo journalctl -u swifteats -f | grep "cache miss"
```

**Step 3: If Redis memory > 80% — flush non-critical keys**
```bash
# Connect to Redis CLI:
redis-cli -h <elasticache-endpoint> -p 6379

# Check memory:
INFO memory

# Flush only menu cache (safe — menus auto-repopulate from DB):
redis-cli -h <elasticache-endpoint> -p 6379 --scan --pattern "menu:*" | xargs redis-cli DEL
```

> **WARNING:** Do NOT run `FLUSHALL` — this would clear all sessions and log out all users.

**Step 4: If Redis instance is unhealthy — promote replica**
- AWS Console → ElastiCache → SwiftEats-Redis → Actions → Failover Primary
- Failover completes in ~30–60 seconds. Brief connection errors expected.

**What success looks like:**  
- `CacheHits / (CacheHits + CacheMisses)` ratio returns above 90%  
- DB read replica connection count drops  
- P99 latency improves within 2 minutes  

**Who to notify:** Post in `#incident-response`

---

## STEP 4 — ROLLBACK

### When to Roll Back vs Scale Forward

| Situation | Action |
|---|---|
| Scaling fixed the issue | Scale forward. Do not roll back. Keep monitoring. |
| New code deployment caused the issue (e.g., post-deploy spike in errors) | Roll back application code immediately. |
| DB performance degraded after a schema migration | **NEVER roll back DB schema.** Roll back app code only. |
| Errors started 5–15 minutes after a deploy | Check deploy timestamp vs error timestamp. If correlated, roll back. |
| Errors are infrastructure-related (AWS service issue) | Scale forward / failover. Rolling back code won't help. |

### Rollback Trigger Criteria

Roll back if **all three** of the following are true:
1. 5xx error rate > 5% for more than 5 consecutive minutes
2. The error spike started within 15 minutes of a deployment
3. Scaling up did not reduce the error rate

### Rollback Command (Application Code Only)

```bash
# Find the previous stable task definition:
aws ecs describe-task-definition \
  --task-definition swifteats-app \
  --query "taskDefinition.taskDefinitionArn" \
  --region ap-south-1

# List recent task definitions to find previous version:
aws ecs list-task-definitions \
  --family-prefix swifteats-app \
  --sort DESC \
  --max-items 5 \
  --region ap-south-1

# Roll back to previous version (replace :42 with previous version number):
aws ecs update-service \
  --cluster swifteats-prod \
  --service swifteats-app \
  --task-definition swifteats-app:42 \
  --force-new-deployment \
  --region ap-south-1
```

> If using EC2 Auto-Scaling with AMIs instead of ECS, use the previous Launch Template version:
```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name SwiftEats-ASG \
  --launch-template LaunchTemplateName=swifteats-lt,Version='$Previous' \
  --region ap-south-1

aws autoscaling start-instance-refresh \
  --auto-scaling-group-name SwiftEats-ASG \
  --preferences MinHealthyPercentage=50 \
  --region ap-south-1
```

### ⚠️ CRITICAL WARNING: Never Roll Back the Database Schema
```
DO NOT:  Run any migration DOWN command
DO NOT:  DROP, ALTER, or TRUNCATE tables
DO NOT:  Remove columns that the previous app version did not have
DO NOT:  Touch pg_dump / pg_restore on a live production database

WHY:     Rolling back DB schema while the app is running will corrupt data.
         The previous app version will read garbage from columns it doesn't expect.
         This is unrecoverable without a full backup restore (hours of downtime).

IF:      A schema migration is causing errors, roll back ONLY the application code.
         The app code must be written to handle both old and new schema states.
         Contact the DBA team before any schema changes during an active incident.
```

---

## STEP 5 — POSTMORTEM TEMPLATE

> Fill this template within 48 hours of the incident. All sections are mandatory. If you don't know the answer, write "Unknown — action item to find out."

---

```markdown
# Postmortem: [Incident Title]
**Date:** [YYYY-MM-DD]  
**Duration:** [Start time IST] – [End time IST] ([X] minutes)  
**Severity:** [P0 / P1 / P2]  
**Postmortem Author:** [Your name]  
**Review Meeting:** [Scheduled date/time]

---

## 1. Incident Summary
[One paragraph. What happened, who was affected, what was the business impact. 
Write this as if the reader has never heard of the incident. 
Example: "On [date] at [time], SwiftEats experienced a complete service outage 
lasting [X] minutes, affecting all users attempting to browse or place orders. 
The root cause was [X]. Estimated [N] users impacted. Estimated revenue impact: ₹[X] crore."]

---

## 2. Timeline
[Fill every row. Times must be IST. Be precise — "around 8pm" is not acceptable.]

| Time (IST) | Event | Who Detected |
|---|---|---|
| HH:MM | Notification sent to users | Automated |
| HH:MM | First CloudWatch alarm fired | CloudWatch |
| HH:MM | On-call engineer paged | PagerDuty |
| HH:MM | Engineer began triage | [Name] |
| HH:MM | Root cause identified | [Name] |
| HH:MM | First remediation action taken | [Name] — describe action |
| HH:MM | Service partially restored | Monitoring |
| HH:MM | Service fully restored | Monitoring |
| HH:MM | Incident declared resolved | [Name] |

---

## 3. Root Cause
[Be specific. Not "the server was overloaded." Write the exact technical mechanism.]

**Primary root cause:** [Specific technical cause with exact failure point]

**Contributing factors:**
- [Factor 1: What made this worse than it needed to be]
- [Factor 2]
- [Factor 3]

**Why was this not caught before the incident?**
[What monitoring was missing? What test was not run? What assumption was wrong?]

---

## 4. Impact
[Fill every row. Estimated is acceptable — write the method of estimation.]

| Metric | Value | Source |
|---|---|---|
| Total downtime | [X] minutes | Monitoring |
| Users affected | [N] users | ALB access logs |
| Failed requests | [N] requests | ALB 5xx metrics |
| Revenue lost | ₹[X] crore | Finance estimate |
| SLA breach | Yes / No | SLA document |
| Promo overclaim (if applicable) | ₹[X] crore | DB audit |

---

## 5. What Worked
[Be honest. What parts of the response went well? Which tools, runbook steps, or 
on-call processes worked as designed?]
- [Example: "CloudWatch alarm fired within 2 minutes of first error."]
- [Example: "Runbook STEP 3.2 correctly identified DB pool exhaustion as root cause."]

---

## 6. What Did Not Work
[Be equally honest. What failed to detect the issue? What made the response slower?]
- [Example: "No alarm for PgBouncer pool saturation — added as Action Item."]
- [Example: "Runbook step X was ambiguous — [Name] spent 5 minutes looking up the command."]

---

## 7. Action Items
[Every action item must have an owner and a due date. "TBD" is not acceptable.]

| Action Item | Owner | Due Date | Status |
|---|---|---|---|
| [What needs to be done] | [Name] | [YYYY-MM-DD] | Open |
| Add CloudWatch alarm for PgBouncer pool saturation | [SRE Name] | [date] | Open |
| Load test at 150% of expected peak before next major event | [SRE Name] | [date] | Open |
| Update runbook step [X] to clarify [ambiguous part] | [Name] | [date] | Open |
| Configure pre-scaling schedule for all future promo events | [SRE Name] | [date] | Open |

---

## 8. Lessons Learned
[3–5 sentences. What would you do differently? What systemic change does this incident 
demand? This section is read by the CTO — write accordingly.]

---

**Postmortem Status:** [ ] Draft  [ ] In Review  [ ] Final  
**Approved by:** [SRE Lead] — [date]
```

---

## Quick Reference Card (Print and Keep at Desk)

```
SWIFTEATS INCIDENT QUICK REFERENCE
────────────────────────────────────────────────────
DETECT:    CloudWatch → Alarms → SwiftEats-*
SLACK:     #incident-response #cloudwatch-alerts
PAGERDUTY: Primary on-call → SRE Lead

5xx ALARM?         → Check DB Connections
                     > 185 → Kill stale connections (STEP 3.2)
                     < 185 → Check EC2 CPU
                              > 70% → Scale ASG (STEP 3.1)
                              < 70% → Check Redis CacheMisses (STEP 3.4)

SQS ALARM?         → Scale payment workers to 20 (STEP 3.3)
                     Check Razorpay status page

INSTANCE DOWN?     → aws autoscaling set-desired-capacity 30
ROLLBACK?          → ONLY app code. NEVER DB schema.

DBA PAGERDUTY:     #db-oncall
PAYMENTS:          #payments-oncall
CTO ESCALATION:    > 15 minutes unresolved P0
────────────────────────────────────────────────────
```
