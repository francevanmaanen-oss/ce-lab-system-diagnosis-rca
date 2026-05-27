# Correlation Analysis

Cross-signal pattern analysis to confirm the causal chain between the traffic surge, connection pool exhaustion, and user-facing latency.

---

## Pattern 1: Traffic Surge Precedes All Degradation

The request rate began rising at **14:00 UTC**, approximately **10 minutes before** latency and error signals became severe. This ordering confirms that the traffic surge was the **initiating cause**, not a symptom of something else.

If latency had risen first, we might suspect a slow query, memory leak, or runaway process causing requests to pile up. Instead, the sequence is:

> Traffic ↑ → Pool exhausted → Wait queue grows → Latency ↑ → Timeouts → Errors ↑

---

## Pattern 2: DB Pool Exhaustion Is the Inflection Point

Latency remained near-normal until the connection pool hit 100% utilization at **14:10 UTC**. The moment the pool was full and a wait queue formed, latency and errors began compounding rapidly.

| Metric | 14:05 (pool 80%) | 14:10 (pool 100%) | Change |
|--------|-----------------|-------------------|--------|
| Latency P95 | 480ms | 2,800ms | +483% |
| Error Rate | 0.8% | 2.9% | +263% |
| Wait Queue | 0 | 12 | New |

This is a classic **queuing theory** inflection: once a resource hits 100%, wait times grow non-linearly. Every new request must now wait for an existing one to finish before it can even begin.

---

## Pattern 3: CPU and Memory Are Uncorrelated

CPU and memory showed no meaningful correlation with the latency spike. Both metrics drifted upward slightly (as expected with more traffic) but never approached saturation thresholds.

This is critical evidence: it **rules out** common alternative explanations:
-  Not a memory leak (memory stable, no OOM events)
-  Not CPU-bound computation (CPU well under 50%)
-  Not a disk I/O issue (disk reads flat)
-  Must be a connection/I-O wait issue in application layer

---

## Pattern 4: Sharp Recovery After Fix Confirms Root Cause

After the pool size was increased from 20 → 50 at **14:35 UTC**, latency dropped from 5,100ms to under 400ms within **8 minutes** — without any other changes.

If the root cause had been something else (e.g., a slow database query or memory pressure), simply enlarging the pool would not have produced such an immediate recovery. The speed and completeness of recovery is the strongest single piece of evidence confirming pool exhaustion as the root cause.

---

## Causal Chain Summary

```
Marketing email campaign sent at ~13:55 UTC
        │
        ▼
Request rate increases 3x (500 → 1,500 req/min)
        │
        ▼
DB connection pool reaches max capacity (20/20) at 14:10
        │
        ▼
New requests queue waiting for a free connection
        │
        ▼
Queue depth grows → average wait time grows non-linearly
        │
        ▼
Requests exceed 30s timeout threshold → HTTP 504 errors
        │
        ▼
Users experience high latency and failed page loads
        │
        ▼
[FIX at 14:35: Pool increased to 50]
        │
        ▼
Queue drains → latency returns to baseline
```

---

## What the Data Rules Out

| Hypothesis | Evidence Against |
|-----------|-----------------|
| Application bug / code regression | Incident started cleanly at traffic surge, not before |
| Database query performance degradation | DB query time was normal; only wait time for connection was high |
| EC2 instance under-provisioning | CPU and memory both healthy throughout |
| Network bottleneck | Network utilization < 3% of instance capacity |
| DDoS or malicious traffic | Traffic was clean web requests; source was marketing campaign |
