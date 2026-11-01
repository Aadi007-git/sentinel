# ADR-0003: Spread and fan out probe checks with a dispatcher and a delay queue

- **Status:** Accepted
- **Date:** 2026-11-01
- **Deciders:** Aadi007-git

## Context

At design scale each probe region must run **~10,000 checks per minute** (capacity estimates).
Two problems follow:

1. **Load spreading:** if every check fires at second :00, that's a 30,000-check spike across the
   regions, 60× the average. Checks must be spread across the minute.
2. **Fan-out:** a single Lambda run is limited to **1,024 open file descriptors** (roughly, open
   connections). With checks that can take up to 10 s to time out, the worst case needs ~1,700 at
   once. One run per region **cannot** reach the design target.

EventBridge Scheduler's smallest interval is **1 minute**, so it can't trigger runs at sub-minute offsets.

## Decision drivers

1. Reach the **10,000-monitor design target** (NFR7).
2. **Spread load** evenly across the minute.
3. **Cost:** pay for real work, not for sleeping; stay near the free tier at real scale (NFR11).
4. **No servers or VPCs** to manage in the probe regions.
5. Keep detection within budget (NFR1).

## Options considered

### Option A: One run per region per minute (it sleeps until each monitor's offset)
- ✅ Simplest: one function, no extra queue.
- ❌ Hits the 1,024-connection limit before the design target.
- ❌ Billed for ~60 s of mostly sleeping every minute: ~$10–25/month always-on across 3 regions.

### Option B: Dispatcher + delay queue + parallel workers
- ✅ The dispatcher runs for ~1 s per minute; workers are billed only while checking.
- ✅ Scales by **adding shards**: each worker handles ≤ 200 monitors, far below the connection limit.
- ✅ SQS **per-message delay** (`DelaySeconds`, up to 15 minutes) gives sub-minute offsets without
  any scheduler support.
- ❌ More moving parts: one extra queue and one extra function per region.
- ❌ The dispatch hop adds ~1 s of latency (within NFR1's margin).

### Option C: Always-on containers in each region (ECS Fargate) with their own scheduler loop
- ✅ Natural, fine-grained spreading; no cold starts.
- ❌ ~$30/month for 3 regions even when idle; needs a VPC and public IPs in every region.
- ❌ Gives up the serverless model the project is meant to demonstrate.

## Decision

**Option B: dispatcher + delay queue + parallel workers**, in each probe region:

1. **EventBridge Scheduler** triggers the **dispatcher** every minute.
2. The dispatcher reads the config snapshot and puts each monitor in one of **6 buckets of 10 seconds**
   using `offset = hash(monitor_id) % 60`, rounded down to the bucket. The hash is stable, so a
   monitor keeps the same slot every minute.
3. Each bucket is split into **shards of ≤ 200 monitors**, each sent to the regional **jobs queue**
   with `DelaySeconds` = the bucket's offset (0, 10, 20, 30, 40, 50).
4. **Worker** Lambdas, triggered by the jobs queue, run one shard each with a concurrency limit,
   spread across their 10 s window, and send results packed ~100 per message to the core's results queue.

These parameters (6 buckets, 200 monitors per shard) are starting values, to be tuned with real
measurements.

## Consequences

- **Positive:** reaches the design target; load is spread evenly; at real scale (~50 monitors) it's
  1 dispatcher run plus ≤ 6 worker runs per region per minute, about 900,000 invocations a month in
  total, inside the 1M free tier.
- **Negative (accepted):** two functions and one queue per region to deploy and monitor; one more
  hop to trace (Step 3.1 carries the trace context through the queue).
- **Follow-ups:**
  - The worker must handle **SSRF safety** and **timeouts** (Step 3.5 / LLD).
  - Metrics: shard run time, checks per shard, jobs-queue age (Step 3.1).
  - The probe-region Terraform module creates scheduler + dispatcher + jobs queue + worker together (Step 2.6).

## Revisit when

- Sub-minute intervals (e.g., 30 s) are needed. The dispatcher could then enqueue two rounds per run.
- Per-invocation costs at very large scale outweigh an always-on container fleet.
