# Prevention — How to Avoid Recurrence

This document covers the monitoring, process, and organizational changes needed to prevent a recurrence of connection pool exhaustion — and similar capacity-related incidents.

---

## 1. Monitoring: Know Before It Breaks

The `DBConnectionPoolUsage` metric already existed during this incident but had **no alarm configured**. Adding a simple threshold alert would have given approximately 10 minutes of warning.

### CloudWatch Alarms to Add

```bash
# Alarm: Pool approaching saturation (warning)
aws cloudwatch put-metric-alarm \
  --alarm-name "DB-ConnectionPool-High" \
  --alarm-description "DB connection pool > 80% utilized" \
  --namespace Application \
  --metric-name DBConnectionPoolUsage \
  --statistic Average \
  --period 60 \
  --threshold 80 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:ops-alerts \
  --ok-actions arn:aws:sns:us-east-1:123456789:ops-alerts

# Alarm: Pool near-exhaustion (critical)
aws cloudwatch put-metric-alarm \
  --alarm-name "DB-ConnectionPool-Critical" \
  --alarm-description "DB connection pool > 95% utilized — imminent exhaustion" \
  --namespace Application \
  --metric-name DBConnectionPoolUsage \
  --statistic Maximum \
  --period 60 \
  --threshold 95 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:pagerduty-critical
```

### Dashboard Panel

Add a "Connection Pool Health" section to the main ops dashboard with:
- Pool utilization % (gauge, 0–100%)
- Wait queue depth (line chart)
- Active connections (line chart)
- Pool size over time (to track when auto-scaling is working)

---

## 2. Capacity Planning: Know Your Limits Before You Hit Them

A quarterly capacity review should answer:

- What is the current peak traffic level?
- At what traffic level would each major resource (pool, CPU, memory) saturate?
- What is the expected traffic for the next quarter (organic growth + planned campaigns)?
- Do current limits support 3x peak traffic? If not, adjust.

### Capacity Planning Template

```markdown
## Quarterly Capacity Review — Q1 2024

### Current Baselines
- Normal traffic: 500 req/min
- Peak traffic (past 90 days): 620 req/min
- P95 latency at peak: 420ms

### Resource Headroom at Current Peak
| Resource | Limit | Peak Usage | Headroom |
|----------|-------|------------|----------|
| DB Connection Pool | 50 | 28 | 44% |
| EC2 CPU | 100% | 53% | 47% |
| EC2 Memory | 100% | 71% | 29% |
| RDS IOPS | 3,000 | 820 | 73% |

### Traffic Projections
- Q1 organic growth: +10% (~550 req/min expected)
- Planned campaigns: 3 email sends to 80k subscribers (+3x spikes)
- Required headroom for campaigns: pool must support 1,500 req/min

### Action Items
- [ ] Load test at 1,500 req/min before next campaign
- [ ] Verify pool auto-scaling works correctly up to 100 connections
```

---

## 3. Process: Engineering Must Know What's Coming

### Marketing ↔ Engineering Campaign Checklist

Before any marketing activity expected to drive > 1.3x normal traffic, the marketing team must file a **Campaign Engineering Review** at least **48 hours before launch**:

```markdown
## Campaign Engineering Review Request

**Campaign name:** Spring Sale Email
**Launch date/time:** Feb 15, 2024 at 10:00 AM EST
**Expected audience reach:** 80,000 subscribers
**Expected traffic multiplier:** ~3x for 30–60 minutes post-send
**Links to similar past campaigns:** [link]

**Engineering review:**
- [ ] Current resource headroom supports 3x traffic
- [ ] Load test completed at target traffic level
- [ ] On-call engineer notified of launch window
- [ ] Rollback plan documented if issues arise
**Reviewed by:** [Engineer name]
**Approved:** Yes / No
```

This process prevents the communication gap that directly contributed to this incident.

---

## 4. Load Testing: Prove It Works Before Users Find Out It Doesn't

### Regular Load Testing Schedule

| Frequency | Test | Threshold |
|-----------|------|-----------|
| Monthly | Baseline: 1x traffic for 30 min | P95 < 400ms, errors < 0.5% |
| Quarterly | Spike: 3x traffic for 15 min | P95 < 800ms, errors < 1% |
| Before campaigns | Campaign simulation: 3–5x for 10 min | P95 < 1,000ms, errors < 2% |
| After major infra changes | Full regression at 3x | All SLAs met |

### Load Test Script (Locust)

```python
# load_tests/locustfile.py
from locust import HttpUser, task, between

class WebAppUser(HttpUser):
    wait_time = between(1, 3)

    @task(5)
    def home_page(self):
        self.client.get("/")

    @task(3)
    def product_listing(self):
        self.client.get("/products")

    @task(2)
    def product_detail(self):
        self.client.get("/products/123")

    @task(1)
    def user_profile(self):
        self.client.get("/profile")
```

```bash
# Run campaign simulation (3x traffic for 10 minutes)
locust --headless \
  --host https://staging.example.com \
  -u 1500 --spawn-rate 100 \
  -t 10m \
  --html load_test_report.html
```

---

## 5. Runbook: Make the Fix Fast Next Time

When the next connection pool alarm fires, the on-call engineer should have a clear playbook rather than needing to rediscover the solution:

```markdown
## Runbook: DB Connection Pool High Utilization

**Alarm:** DB-ConnectionPool-High (> 80%) or DB-ConnectionPool-Critical (> 95%)

### Step 1: Assess (2 minutes)
- Check current pool utilization in CloudWatch dashboard
- Check request rate — is there a traffic spike?
- Check application logs for `TimeoutError: QueuePool limit`

### Step 2: Mitigate (5 minutes)
If pool utilization > 95% and requests are queuing:

1. Open `config/database.py`
2. Increase `DB_CONNECTION_POOL_SIZE` to `current_value + 30`
3. Restart service: `sudo systemctl restart webapp`
4. Watch CloudWatch for pool utilization to drop below 80%

### Step 3: Verify (5 minutes)
- Pool utilization < 80%
- Wait queue depth = 0
- P95 latency returning toward baseline
- Error rate dropping

### Step 4: Follow Up
- File incident report
- Identify traffic source (campaign? bot? organic?)
- Schedule capacity planning review if traffic is trending up
```

---

## Summary of Prevention Measures

| Category | Action | Priority | Owner |
|----------|--------|----------|-------|
| Monitoring | CloudWatch alarm on pool > 80% | 🔴 P1 | Sarah K. |
| Monitoring | Add pool panel to ops dashboard | 🟡 P2 | Sarah K. |
| Capacity | Quarterly capacity review process | 🟡 P2 | Manager |
| Process | Marketing campaign review checklist | 🔴 P1 | Manager |
| Testing | Monthly + pre-campaign load tests | 🟡 P2 | Engineering |
| Documentation | Connection pool runbook | 🔴 P1 | Alex R. |
| Infrastructure | RDS Proxy for connection multiplexing | 🟢 P3 | Mike T. |
