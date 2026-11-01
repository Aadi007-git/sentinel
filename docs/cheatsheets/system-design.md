# System design cheatsheet

> Grows as we go. Step 1.1: the interview framework, requirements, estimation. Later: patterns,
> distributed-systems concepts, reliability.

## The framework (interviews and real design docs)

| # | Stage | Questions to answer | Time in a 45-min interview |
|---|---|---|---|
| 1 | **Requirements** | What must it do? How well (latency, availability, scale)? What's out of scope? | ~5 min |
| 2 | **Estimates** | How many requests/s? How much data per day? Read-heavy or write-heavy? | ~5 min |
| 3 | **High-level design** | Main components and how data flows between them | ~10 min |
| 4 | **Deep dives** | The 1–2 hardest parts (for Sentinel: quorum, the ingestion path) | ~15 min |
| 5 | **Trade-offs + bottlenecks** | What breaks first at 10× scale? What did you trade away, and why? | ~10 min |

## Requirements

| Functional | Non-functional |
|---|---|
| *What* it does: verbs, features | *How well*: availability, latency, throughput, durability, consistency, security, cost |
| "A user can add a monitor" | "99% of outages alerted within 3 minutes" |
| Drives APIs and screens | Drives **architecture** |

- Make NFRs **measurable**: a number, a percentile, a time window.
- Write **non-goals** explicitly; scope creep kills designs.

## Availability: the "nines"

| Availability | Downtime per year | Per month | Per day |
|---|---|---|---|
| 99% ("two nines") | 3.65 days | 7.2 hours | 14.4 min |
| 99.5% | 1.83 days | 3.6 hours | 7.2 min |
| 99.9% ("three nines") | 8.76 hours | 43.8 min | 1.44 min |
| 99.95% | 4.38 hours | 21.9 min | 43 s |
| 99.99% ("four nines") | 52.6 min | 4.4 min | 8.6 s |

Each extra nine costs roughly 10× more effort. **Components in series multiply**: two 99.9% services
that both must work give 99.9% × 99.9% ≈ **99.8%**.

## Estimation toolkit

**Handy numbers**

| Fact | Value | Rounded for mental maths |
|---|---|---|
| Seconds in a day | 86,400 | **~100,000 (10⁵)** |
| Seconds in a month | ~2.6 million | **~2.5 × 10⁶** |
| 1 request/s sustained | 86,400/day, ~2.6M/month | |
| 1 million requests/day | ~12 requests/s | |
| 1 KB / 1 MB / 1 GB / 1 TB | 10³ / 10⁶ / 10⁹ / 10¹² bytes | |

**Formulas**

| You want | Formula |
|---|---|
| Requests (or events) per second | `items × frequency per item ÷ seconds in the period` |
| Rows per day | `events per second × 86,400` |
| Storage per day | `rows per day × bytes per row` |
| Storage over retention | `storage per day × days kept` (+ indexes and replicas if not already counted) |
| Peak load | `average × peak factor` (often 2–10×; or worse if everything is scheduled at the same moment) |

**Rules**
- Round hard: 86,400 → 100,000 is fine. The order of magnitude is what drives the design.
- **Average ≠ peak.** Systems fall over at peak. Ask "what if everything happens at once?"
- Convert to units you can feel: "8 GB/day" means something; "43,200,000 × 200" doesn't.
- Always ask: **read-heavy or write-heavy?** Sentinel is extremely write-heavy (every check is a
  write; humans read occasionally), and that shapes the database design.

## Latency numbers every engineer should know (approximate)

| Operation | Time |
|---|---|
| Read from memory (RAM) | ~100 ns |
| Read 1 MB sequentially from memory | ~10 µs |
| Read from an SSD | ~100 µs |
| Round trip within one data centre | ~0.5 ms |
| Read 1 MB sequentially from an SSD | ~1 ms |
| Round trip within one AWS region (between availability zones) | ~1–2 ms |
| Round trip US East ↔ Europe | ~70–90 ms |
| Round trip US East ↔ Mumbai | ~180–220 ms |

The lesson: **network hops between regions cost about a million times more than memory reads.**
Avoid chatty cross-region calls. That's why Sentinel's probes send results **in batches**.

## Communication: sync or async?

| Use **sync** (request/response) when… | Use **async** (queue/events) when… |
|---|---|
| The caller needs the answer **now** (a user clicked "Save") | The work can happen **later**; no reply is needed |
| The operation is fast and the callee is reliable | The receiver may be slow, down or scaling |
| | You need to absorb spikes or fan out work |

Async's price: **at-least-once delivery → idempotent consumers**, no ordering guarantee, harder debugging.

## Queue vs stream vs event bus

| | Queue (SQS) | Stream / log (Kafka, Kinesis) | Event bus (EventBridge) |
|---|---|---|---|
| Model | Work items, each taken by one consumer, then deleted | Append-only log; many consumer groups each read everything | Routes each event to matching targets |
| Ordering | No (standard) / yes (FIFO) | Yes, per partition | No |
| Replay | No | Yes (within retention) | Limited (archive) |
| Ops cost | None (serverless) | Brokers/shards to run and pay for | None |
| Pick when | Background jobs, buffering, ingestion | Many consumers, replay, ordering, very high throughput | Routing events between services and accounts |

## Architecture shapes

| Shape | Choose when |
|---|---|
| Monolith | Small app, one team, uniform load |
| **Modular monolith** (one image, many roles) | One team, parts need independent scaling or isolation |
| Microservices | Many teams who need independent releases; clear, stable domain boundaries |

## Patterns introduced so far

| Pattern | One-liner | In Sentinel |
|---|---|---|
| **Queue-based load levelling** | A queue absorbs spikes; consumers work at a steady rate | Results queue |
| **Dead-letter queue** | Park messages that keep failing | Results DLQ |
| **Idempotent consumer** | Duplicates are harmless | Dedup by `check_id` |
| **Fan-out / fan-in** | Split work into parallel shards / gather results | Dispatcher → workers |
| **Stable offset spreading** | `hash(id) % period` flattens spikes | Checks spread across the minute |
| **Transactional outbox** | Write the state change and the "send this" event in one transaction | Incidents → notifier (details in Step 3.0) |
| **Static stability** | Keep serving from a precomputed snapshot when dependencies fail | Status page on S3 + CDN |
| **Dead man's switch** | Alert when an expected heartbeat *stops* | External heartbeat |

## Writing an ADR

Context → decision drivers → options (pros/cons) → decision → consequences (including the costs you
accept) → when to revisit. One decision per ADR; never edit an accepted one; supersede it instead.
Template: [docs/adr/README.md](../adr/README.md).

## Unit conversion: bytes → GB in your head

Group the digits in threes from the right; each group of three zeros is one unit step:

```
8,640,000,000 bytes   →   8,640,000 KB   →   8,640 MB   →   8.64 GB
                (÷1,000)           (÷1,000)        (÷1,000)
```

Quick check: **10 digits ≈ GB, 13 digits ≈ TB.** If a result has more digits than seems
reasonable, you've probably left it in bytes.

## Gotchas

- **Forgetting to convert units.** `rows × bytes per row` gives *bytes*. Convert before labelling
  it GB (Step 1.1: 8,640,000,000 is 8.64 GB, not 8.64 billion GB).
- **Multiplying the wrong count.** Check which quantity each formula needs (raw rows vs rollup rows).
- Designing boxes before requirements: you'll optimize for the wrong thing.
- Forgetting **index overhead** (often +50–100% on top of raw row size) and **replicas** (×2 or ×3).
- Planning for average load only.
- Unmeasurable NFRs ("fast", "scalable", "highly available") can't be designed for or tested.
