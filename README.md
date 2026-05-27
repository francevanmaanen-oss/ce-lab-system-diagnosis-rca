# ce-lab-system-diagnosis-rca

## Investigation Summary

A production web application experienced severe latency degradation on January 20, 2024 between 14:00–15:00 UTC. Users reported slow page loads with some requests timing out entirely. This repository documents the full investigation, root cause analysis, and remediation proposals.

**Root Cause:** Database connection pool exhaustion caused by an unplanned 3x traffic surge from a marketing campaign launch.

**Outcome:** Incident resolved in 60 minutes. Latency returned to baseline after pool size was increased.

---

## Key Findings

| Signal | Normal | During Incident | Change |
|--------|--------|-----------------|--------|
| Request Rate | 500 req/min | 1,500 req/min | +200% |
| Error Rate | 0.1% | 5.0% | +4,900% |
| Latency (P95) | 300ms | 5,000ms | +1,567% |
| CPU Utilization | ~40% | ~45% | Normal |
| Memory Usage | ~65% | ~70% | Normal |
| DB Connection Pool | ~50% | **100%** | **Exhausted** |

**Critical finding:** CPU and memory remained healthy throughout the incident. The bottleneck was entirely in the database connection pool, which was capped at 20 connections — a limit sized for normal traffic, not a 3x surge.

---

## Timeline of Discovery

| Time (UTC) | Event |
|------------|-------|
| 14:00 | CloudWatch alarm fires: P95 latency > 2,000ms |
| 14:05 | On-call engineer paged via PagerDuty |
| 14:10 | Investigation begins — RED method applied |
| 14:15 | Elevated error rate (5%) and latency (5,000ms P95) confirmed |
| 14:20 | USE method applied to EC2, memory, and database resources |
| 14:25 | DB connection pool metric identified at 100% utilization |
| 14:30 | Root cause confirmed: pool exhausted (max 20 connections) |
| 14:35 | Mitigation applied: pool size increased to 50 |
| 14:45 | Latency begins recovering; errors drop below 1% |
| 15:00 | Latency back to baseline; incident closed |

---

## Repository Structure

```
ce-lab-system-diagnosis-rca/
├── README.md                          ← You are here
├── rca/
│   ├── root-cause-analysis.md        ← Full RCA document
│   ├── investigation-notes.md        ← Step-by-step investigation log
│   └── evidence/                     ← Supporting data and charts
├── analysis/
│   ├── red-method-analysis.md        ← Rate, Errors, Duration findings
│   ├── use-method-analysis.md        ← Utilization, Saturation, Errors per resource
│   └── correlation-analysis.md      ← Cross-signal pattern analysis
├── proposals/
│   ├── immediate-fix.md              ← Quick mitigation (applied during incident)
│   ├── long-term-fix.md              ← Permanent architectural solution
│   └── prevention.md                ← How to prevent recurrence
└── screenshots/
    └── ...                           ← Dashboard and metric screenshots
```

# Reflection Questions #

1. RED first, then USE?
RED tells you what's broken from the user's perspective — slow, erroring, high traffic. Once you know the symptom, USE helps you find which resource caused it. You start with "users are suffering" then ask "what infrastructure explains that?"
2. How did correlation nail the root cause?
CPU and memory were totally flat while latency exploded — that eliminated the obvious suspects. The only thing that spiked at the exact same moment as latency was the connection pool hitting 100%. Traffic surge → pool exhausted → latency spikes. Three signals, one timeline, one cause.
3. Immediate vs long-term fix?
Immediate is a band-aid — bump the pool from 20 to 50, stop the bleeding, get users back online. Long-term actually fixes the underlying fragility — dynamic scaling, RDS Proxy, load testing — so the same thing can't happen again at 5x traffic.
4. Why write an RCA after it's already fixed?
Because "fixed" just means it stopped hurting. Without an RCA you don't know why it broke, so you'll hit the same wall again. It also turns one team's painful experience into documented knowledge the whole org can learn from.
5. How do you pick P0/P1/P2?
Roughly: P0 = completely down, everyone affected, revenue bleeding fast. P1 = degraded but limping, significant users impacted. P2 = some users affected, workaround exists. This incident was P1 — service was slow and erroring but not dead, and only ~5k users hit it during a 1-hour window.