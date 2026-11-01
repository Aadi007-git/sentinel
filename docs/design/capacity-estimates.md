# Sentinel: capacity estimates (back-of-envelope)

**Goal of this page:** find out how big Sentinel gets at its design target, so we know which part
breaks first and what it costs. Round aggressively: we want the right *order of magnitude*,
not exact numbers. Formulas are in [the system-design cheatsheet](../cheatsheets/system-design.md).

## Assumptions

| Input | Value | Why |
|---|---|---|
| Monitors (design target) | **10,000** | NFR7 |
| Regions checking each monitor | **3** | us-east-1, eu-west-1, ap-south-1 |
| Interval | **60 s** | Decision D2 |
| Size of one stored check result | **200 bytes** (data + index overhead) | ~110 B row + ~90 B of index entries in Postgres |
| Size of one hourly rollup row | **100 bytes** | One row per monitor per hour: count, ups, p50/p95 latency |
| Probe timeout | **10 s** | |
| Processing time (queue → evaluator → notifier) | **~10 s** | |

## Worksheet: fill in the `____`

**A. Average checks per second**
`monitors × regions ÷ interval_seconds` = 10,000 × 3 ÷ 60 = **500 checks/s**

**B. Checks (rows) per day**
`A × 86,400` = **43.2 million rows/day**

**C. Raw storage per day**
`B × 200 bytes` = 8,640,000,000 bytes = **8.64 GB/day**

**D. Raw storage for different retention periods**
- 7 days = **~60 GB**
- 30 days = **~260 GB**
- 90 days = **~780 GB**

**E. Hourly rollups**
- Rollup rows per day = `monitors × 24` = **240,000**
- Rollup storage per year = 240,000 × 365 × 100 bytes = 8,760,000,000 bytes = **~8.8 GB/year**

**F. Probe invocations per month (Lambda)**
One invocation per region per minute: `regions × 60 × 24 × 30` = **129,600 invocations/month**
That's ~13% of Lambda's free tier of 1,000,000 requests/month, so the request count is effectively free.

**G. Worst-case detection time** (outage starts *just after* a check, so we wait a full interval before the next one)
- Wait for round 1 to fail: **60 s** (the next scheduled check is a full interval away)
- Wait for round 2 to fail: **60 s** (another full interval)
- Probe timeout on that last check: **10 s** (a hung server makes the probe wait the full timeout before calling it a failure)
- Processing: **10 s** (queue → evaluator → notifier)
- **Total: 140 s.** Meets NFR1 (≤ 180 s) with a **40 s safety margin**.

Round 1's timeout doesn't add to the total: it happens *while* we're already waiting for round 2.
Only the last check's timeout is on the critical path.

## Think about it (we'll discuss these)

1. **Average vs peak:** A says 500 checks/s *on average*. What happens if all 10,000 monitors are
   scheduled at exactly second :00 of each minute? How many checks hit in that one second, and how
   could we spread them out?
2. **Rollups vs raw:** compare C (per day) with E (per year). Why would charts covering 90 days read
   rollups instead of raw rows?
3. **The CDN effect:** during a big outage, 10,000 people refresh a status page. If the CDN caches
   the page for 30 seconds, roughly how many requests reach *our* servers per minute?

## Results

| Metric | Design target (10,000 monitors) | Real use (50 monitors) |
|---|---|---|
| Checks per second (average) | **500** | 2.5 |
| Checks in one second if nothing is spread out | **30,000** (60× average) | 150 |
| Rows per day | **43.2 million** | 216,000 |
| Raw storage per day | **8.64 GB** | ~43 MB |
| Raw storage kept (30 days, decision D4) | **~260 GB** | ~1.3 GB |
| Hourly rollups per year | **~8.8 GB** | ~44 MB |
| Probe invocations per month | 129,600 (1 per region per minute; ~13% of the free tier) | same |
| Worst-case detection time | **140 s** (target 180 s) | same |

## What the estimates tell us (design implications)

1. **Sentinel is extremely write-heavy:** ~500 writes/s, sustained, all day; humans read occasionally.
   The ingestor must **write in batches** (hundreds of rows per insert), not 500 single-row
   transactions per second.
2. **Partition `check_results` by day.** Retention (30 days) means *dropping* whole old partitions,
   which is instant, instead of `DELETE`-ing 43 million rows a day, which is slow and bloats the table.
3. **Charts read rollups.** Anything wider than about 2 days reads hourly rollups (~180× fewer rows);
   raw rows are for recent views and incident evidence.
4. **Spread checks across the minute** (`offset = hash(monitor_id) % 60`) so the load is a steady
   500/s instead of a 30,000-check spike.
5. **Pack many results into each queue message.** One result per SQS message at 500/s comes to
   ~390 million queue requests a month (send + receive + delete), **~$150/month**. Packing ~100
   results per message cuts that about 100×, to a few dollars. *The estimate surfaced a cost
   surprise: at scale, the queue would have cost more than the database storage.*
6. **The status page comes from the CDN,** so load on our servers doesn't depend on the number of viewers.
7. **Bottleneck to watch:** at design scale, each region must finish ~10,000 checks per minute, some
   of them waiting up to 10 s for timeouts. A single probe run per region may not finish in time,
   so we may need **fan-out** (several parallel probe runs per region). Decided in the architecture section.
