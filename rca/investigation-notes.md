# Investigation Notes

Raw investigation log documenting each step taken, commands run, and observations made during the incident on January 20, 2024.

---

## 14:05 UTC — Alert Received, Starting Investigation

Received PagerDuty page: `ALARM: HighLatency-Production — P95 ResponseTime > 2000ms`

First step: open the CloudWatch dashboard to get a broad picture before diving into any specific metric.

**Immediate observations:**
- ALB request count is significantly elevated
- P95 latency is trending sharply upward
- Something changed around 14:00 UTC

No deployments are listed in the deployment log for the past 4 hours. This rules out a code change as the initiating cause.

---

## 14:10 UTC — RED Method: Rate

Pulled request rate data from ALB metrics:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/prod-alb/abc123 \
  --start-time 2024-01-20T13:00:00Z \
  --end-time 2024-01-20T15:00:00Z \
  --period 300 \
  --statistics Sum
```

**Finding:** Request rate jumped from ~500/min to ~1,500/min starting at 14:00. That's a 3x increase happening within a single 5-minute window. This is unusual — organic traffic grows gradually. Something triggered this.

Checked Slack for context. Found a message from @marketing at 13:52: *"Campaign email going out now to 80k subscribers!"* — No engineering team tagged or consulted.

---

## 14:13 UTC — RED Method: Errors

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/prod-alb/abc123 \
  --start-time 2024-01-20T13:00:00Z \
  --end-time 2024-01-20T15:00:00Z \
  --period 300 \
  --statistics Sum
```

**Finding:** Error rate at 5.0% (up from 0.1% baseline). All errors are HTTP 504 — Gateway Timeout. 504s mean requests are reaching the ALB and getting forwarded, but the backend isn't responding in time. This is not a crash — the app is running but slow.

---

## 14:15 UTC — RED Method: Duration (Latency)

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/prod-alb/abc123 \
  --start-time 2024-01-20T13:00:00Z \
  --end-time 2024-01-20T15:00:00Z \
  --period 300 \
  --statistics p95
```

**Finding:** P95 at 5,100ms. That's 17x the normal 300ms baseline. The shape of the curve is notable — it's a sudden spike, not a gradual drift. This reinforces that something external (the traffic surge) triggered a saturation event.

RED summary so far:
- Rate: 3x elevated ← initiating cause
- Errors: 50x elevated ← symptom
- Duration: 17x elevated ← symptom

Moving to USE method to find the bottleneck.

---

## 14:20 UTC — USE Method: CPU and Memory

```bash
# CPU
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc123def456 \
  --start-time 2024-01-20T13:00:00Z \
  --end-time 2024-01-20T15:00:00Z \
  --period 300 \
  --statistics Average,Maximum

# Memory (requires CloudWatch Agent)
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_used_percent \
  --dimensions Name=InstanceId,Value=i-0abc123def456 \
  --start-time 2024-01-20T13:00:00Z \
  --end-time 2024-01-20T15:00:00Z \
  --period 300 \
  --statistics Average
```

**Finding:** CPU max 53%, memory max 71%. Both are well within normal ranges. No saturation. These are not the bottleneck.

This narrows it significantly — if CPU and memory are healthy, the bottleneck must be in I/O, a downstream service, or a connection pool.

---

## 14:22 UTC — Checking Application Logs

Queried CloudWatch Logs for errors:

```bash
aws logs filter-log-events \
  --log-group-name /aws/ec2/app/application.log \
  --start-time 1705755600000 \
  --filter-pattern "ERROR"
```

**Key log entry found (14:12 UTC):**
```
ERROR [app.db] Could not acquire DB connection after 30s timeout
  pool_size=20, checked_out=20, overflow=0, wait_queue=34
  SQLAlchemy error: TimeoutError: QueuePool limit of size 20 overflow 0 reached,
  connection timed out, timeout 30
```

**This is the smoking gun.** The application is explicitly logging that the connection pool is full (20/20) with 34 requests queuing. All latency is waiting for a DB connection — not executing any query at all.

---

## 14:25 UTC — USE Method: DB Connection Pool

Queried the custom connection pool metric:

```bash
aws cloudwatch get-metric-statistics \
  --namespace Application \
  --metric-name DBConnectionPoolUsage \
  --start-time 2024-01-20T13:00:00Z \
  --end-time 2024-01-20T15:00:00Z \
  --period 300 \
  --statistics Average,Maximum
```

**Finding:** Pool at 100% from 14:10 to 14:35. Wait queue peaked at 58. No alarm was configured on this metric.

---

## 14:30 UTC — Root Cause Confirmed

Root cause is definitively **database connection pool exhaustion**.

Evidence chain:
1. Traffic 3x higher than normal due to marketing campaign
2. Connection pool maxed at 20 connections (all in use)
3. 58 requests queuing simultaneously
4. Each queued request waits up to 30s before timing out with a 504
5. CPU, memory, disk, and network all healthy

**Decision:** Increase pool size immediately as mitigation.

---

## 14:35 UTC — Mitigation Applied

Updated `config/database.py`:

```python
DB_CONNECTION_POOL_SIZE = 50  # was 20
DB_CONNECTION_MAX_OVERFLOW = 10
```

Restarted application service:

```bash
sudo systemctl restart webapp
```

Monitored CloudWatch for recovery.

---

## 14:43–15:00 UTC — Recovery Confirmed

- 14:43: Wait queue cleared to 0
- 14:48: P95 latency at 480ms
- 14:52: Error rate at 0.3%
- 15:00: P95 latency at 380ms — incident closed

**Post-incident note:** The `DBConnectionPoolUsage` metric was already being published to CloudWatch. A simple threshold alarm at 80% would have given ~10 minutes of warning before the pool fully exhausted. This will be the first action item.
