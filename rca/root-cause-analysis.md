# Root Cause Analysis: High Latency Incident
**Incident ID:** INC-2024-0120-001  
**Date:** January 20, 2024  
**Author:** France van Maanen
**Status:** Resolved  
**Severity:** P1

---

## Executive Summary

On January 20, 2024 between 14:00 and 15:00 UTC, the production web application experienced a severe latency degradation. API response times at the P95 level rose from a baseline of 300ms to over 5,000ms. Approximately 5,000 users were affected, with a 5% request error rate generating HTTP 504 Gateway Timeout responses.

The root cause was **database connection pool exhaustion**. The connection pool was configured with a maximum of 20 connections — sized for normal traffic. A marketing email campaign launched at approximately 13:55 UTC drove a 3x spike in request volume (500 → 1,500 requests/min), overwhelming the pool. Once all 20 connections were checked out, subsequent requests queued indefinitely until timing out.

The incident was mitigated at 14:35 UTC by increasing the pool size to 50 connections. Full service recovery was confirmed by 15:00 UTC.

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
| 13:55 | Marketing team sends email campaign to ~80,000 subscribers |
| 14:00 | Request rate begins rising; CloudWatch alarm fires (P95 latency > 2,000ms) |
| 14:05 | On-call engineer paged via PagerDuty |
| 14:10 | Connection pool reaches 100% utilization (20/20); wait queue forms |
| 14:10 | Application logs begin showing `TimeoutError: QueuePool limit reached` |
| 14:10 | Investigation begins — RED method applied to ALB metrics |
| 14:15 | Confirmed: error rate 2.9%, P95 latency 2,800ms |
| 14:20 | USE method applied to EC2 CPU, memory, network, disk — all healthy |
| 14:25 | DB connection pool metric identified at 100% utilization with 34-request queue |
| 14:30 | Root cause confirmed: pool exhausted (max 20), 58 requests queuing |
| 14:35 | **Mitigation applied:** `DB_CONNECTION_POOL_SIZE` increased from 20 → 50; service restarted |
| 14:43 | Wait queue clears; latency begins recovering |
| 14:50 | P95 latency at 380ms; error rate < 0.5% |
| 15:00 | Incident closed; all metrics at baseline |

---

## Root Cause

**Problem:** The database connection pool was exhausted, causing all incoming requests to queue or time out.

### Why It Happened — 5 Whys

1. **Why did users experience high latency?**  
   Requests were queuing for database connections, taking 30+ seconds before receiving a timeout error.

2. **Why were requests queuing for connections?**  
   The connection pool was fully exhausted — all 20 connections were in use simultaneously.

3. **Why were all 20 connections in use?**  
   Incoming request volume tripled (500 → 1,500 req/min), requiring far more than 20 concurrent DB connections.

4. **Why was the pool only sized for 20 connections?**  
   The pool was sized based on normal traffic load. No capacity planning was done for traffic spikes.

5. **Why was there no capacity planning for this spike?**  
   The marketing campaign was not communicated to the engineering team. No process existed requiring engineering sign-off before campaigns.

### Proximate Cause
Database connection pool exhaustion due to a sudden 3x traffic surge.

### Contributing Causes
- No monitoring or alerting on connection pool utilization
- No auto-scaling or overflow configuration on the connection pool
- No inter-team process to flag traffic-driving campaigns to engineering
- No load testing had been performed at 3x traffic levels

---

## Evidence

| Evidence | Finding |
|----------|---------|
| ALB `RequestCount` metric | 3x traffic spike beginning at 14:00 UTC |
| ALB `HTTPCode_Target_5XX_Count` | 504 errors correlate exactly with pool exhaustion |
| ALB `TargetResponseTime` P95 | 300ms → 5,100ms, recovering sharply after fix |
| Custom `DBConnectionPoolUsage` metric | 100% utilization from 14:10 to 14:35 |
| Application logs (`TimeoutError`) | First error at 14:10, matching pool saturation time |
| EC2 CPU + Memory metrics | Both flat/normal throughout — rules out resource exhaustion |
| Sharp recovery at 14:35 | Latency dropped to 380ms within 8 minutes of pool increase |

See `evidence/` directory for charts and screenshots.

---

## Impact

| Category | Detail |
|----------|--------|
| Users Affected | ~5,000 unique users |
| Duration | 60 minutes (14:00–15:00 UTC) |
| Error Rate at Peak | 5.0% (approximately 450 failed requests) |
| Latency at Peak (P95) | 5,100ms (baseline: 300ms) |
| Estimated Revenue Impact | ~$2,500 (based on average order value and conversion assumptions) |
| SLA Breach | Yes — P95 latency SLA is 1,000ms |

**Severity:** P1 — Service degraded but not completely unavailable.

---

## Resolution

### Immediate Fix (Applied at 14:35 UTC)

The database connection pool maximum was increased from 20 to 50:

```python
# config/database.py — BEFORE
DB_CONNECTION_POOL_SIZE = 20
DB_CONNECTION_TIMEOUT = 30

# config/database.py — AFTER
DB_CONNECTION_POOL_SIZE = 50
DB_CONNECTION_MAX_OVERFLOW = 10
DB_CONNECTION_TIMEOUT = 30
```

The application was restarted to apply the new pool configuration. Within 8 minutes, the wait queue drained and latency returned to baseline.

### Verification

- P95 latency: 5,100ms → 380ms (within 8 minutes of fix)
- Error rate: 5.0% → 0.3% (within 10 minutes of fix)
- DB connection pool utilization: 100% → 56% (healthy headroom)
- Wait queue depth: 58 → 0

---

## Lessons Learned

### What Went Well ✅

- **Alerting worked:** CloudWatch alarm triggered within 1 minute of degradation onset
- **Fast response:** On-call engineer acknowledged PagerDuty within 5 minutes
- **Structured methodology:** RED → USE method led directly to root cause in under 20 minutes
- **Clear monitoring data:** Custom `DBConnectionPoolUsage` metric was already being published (but had no alarm)
- **Fast mitigation:** Only 5 minutes from root cause identification to fix deployment

### What Went Wrong ❌

- **No alarm on connection pool:** Metric existed but had no threshold alert configured
- **No capacity planning:** Pool size was never reviewed against potential traffic scenarios
- **No auto-scaling:** Pool size was static with no ability to respond to traffic changes
- **Communication gap:** Marketing campaign not communicated to engineering before launch
- **No load testing:** System had never been tested above 1.2x normal traffic
- **No runbook:** On-call engineer had no documented procedure for connection pool issues

---

## Action Items

### Immediate (Week 1)

| # | Action | Owner | Due | Status |
|---|--------|-------|-----|--------|
| 1 | Increase connection pool to 50 (already done) | Engineering | Jan 20 | ✅ Done |
| 2 | Add CloudWatch alarm: pool utilization > 80% | Sarah K. | Jan 25 | 🔲 Open |
| 3 | Document connection pool sizing rationale | Mike T. | Jan 22 | 🔲 Open |
| 4 | Write runbook: connection pool exhaustion response | Alex R. | Jan 30 | 🔲 Open |

### Short-term (Month 1)

| # | Action | Owner | Due | Status |
|---|--------|-------|-----|--------|
| 5 | Implement dynamic connection pool with min/max bounds | Mike T. | Feb 5 | 🔲 Open |
| 6 | Load test at 5x normal traffic | Full team | Feb 10 | 🔲 Open |
| 7 | Establish marketing ↔ engineering campaign notification process | Manager | Feb 15 | 🔲 Open |

### Long-term (Quarter 1)

| # | Action | Owner | Due | Status |
|---|--------|-------|-----|--------|
| 8 | Quarterly capacity planning review process | Manager | Mar 1 | 🔲 Open |
| 9 | Add connection pool panel to main ops dashboard | Sarah K. | Feb 5 | 🔲 Open |
| 10 | Evaluate PgBouncer or RDS Proxy for connection pooling at infrastructure level | Mike T. | Mar 15 | 🔲 Open |

---

## Prevention

### Monitoring
- Alert on DB connection pool utilization > 80%
- Add connection wait queue depth as a secondary metric
- Include pool stats in weekly ops review

### Auto-scaling
- Configure dynamic pool sizing (min 20, max 100) based on traffic load
- Set overflow connections for burst handling

### Process
- Require engineering sign-off 48 hours before any marketing campaign expected to drive > 1.5x traffic
- Add "expected traffic impact" field to campaign launch checklist

### Testing
- Load test at 3x and 5x normal traffic quarterly
- Include connection pool behavior in load test success criteria

---

## References

- CloudWatch Dashboard: [link-to-dashboard]
- Incident Slack Thread: #incidents > Jan 20 2024
- Application logs (CloudWatch Logs): `/aws/ec2/app/application.log`
- Related runbooks: [link-to-runbook]
- Previous similar incidents: None on record
