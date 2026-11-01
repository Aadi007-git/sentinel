# Sentinel: high-level design (HLD)

| | |
|---|---|
| **Status** | Draft. Step 1.1 in progress |
| **Author** | Aadi007-git |
| **Last updated** | 2026-11-01 |
| **Related** | [capacity-estimates.md](capacity-estimates.md) · [roadmap](../roadmap.md) · ADRs in [../adr/](../adr/) |

## 1. Context and goals

Sentinel is a multi-region uptime monitoring service. Teams register the websites and APIs they
care about; Sentinel checks them every minute from three AWS regions, confirms real outages by
quorum, alerts the team, drafts an AI incident update, and publishes a public status page.

**Goals**
- Detect real outages quickly, without false alarms caused by one region's network problems.
- Keep the public status page available even when Sentinel's own core is down.
- Be operable by a small team: automated deployments, full observability, low running cost.

## 2. Product decisions

| # | Decision | Chosen | Why | Date |
|---|---|---|---|---|
| D1 | Who can use Sentinel | **Invite-only teams** (multi-tenant design, sign-up by invitation) | Real multi-tenancy without strangers pointing our probes at arbitrary sites | 2026-11-01 |
| D2 | Fastest check interval | **1 minute** | Industry default; matches EventBridge Scheduler's smallest interval, so one probe run = one round | 2026-11-01 |
| D3 | Outage confirmation rule | **2 of 3 regions failing, for 2 consecutive rounds** | One-off blips never page anyone; real outages still alerted within 3 minutes | 2026-11-01 |
| D4 | Raw check-result retention | **30 days raw**; hourly rollups for 1 year; incidents forever | Minute-level evidence for the current and previous month (uptime disputes); ~260 GB at design target; old days dropped by partition | 2026-11-01 |

## 3. Functional requirements

**Accounts and tenancy**
- **FR1** Users belong to a **team**; a team owns **projects**; a project owns monitors, notification
  channels and one public status page.
- **FR2** Sign-up is **by invitation only**: a team admin invites members by email.

**Monitoring**
- **FR3** Users create, edit, pause and delete **monitors**: URL, HTTP method, expected status code,
  optional keyword to find in the response body, interval (≥ 1 min) and timeout.
- **FR4** Each monitor is checked from **3 regions** at its interval. A check records: up/down,
  HTTP status, response time per stage (DNS → connect → TLS → first byte → total), error type,
  and SSL certificate expiry.
- **FR5** SSL certificates expiring within **14 days** trigger a warning.

**Incidents and alerting**
- **FR6** An outage is **confirmed** when **2 of 3 regions** report failure for **2 consecutive
  rounds**. Recovery is confirmed the same way (2 of 3 regions succeed for 2 rounds).
- **FR7** A confirmed outage **opens an incident**; confirmed recovery **resolves** it.
- **FR8** Incident open/resolve events notify the project's channels: **Slack, Discord, email**.
- **FR9** **AI drafts a plain-English incident update** from the evidence (which regions, which
  errors, latency before the failure). A human approves or edits it before it's published.

**Visibility**
- **FR10** Each project has a **public status page**: current status, 90-day uptime bars, incident history.
- **FR11** Team members see **history**: uptime %, response-time charts per region, past incidents.

**Self-monitoring**
- **FR12** Sentinel **monitors itself**, with an independent alert path that still works if
  Sentinel's core is down (a "dead man's switch").

## 4. Non-functional requirements

| # | Quality | Requirement (measurable) | Notes |
|---|---|---|---|
| NFR1 | Detection speed | **99% of confirmed outages are alerted within 3 minutes** of the outage starting | Follows from D2 + D3; see the worst-case maths in capacity-estimates.md |
| NFR2 | False alarms | A failure seen by **only one region never opens an incident** | Quorum (D3) |
| NFR3 | Availability: status page | **99.95%** per month (≈ 22 minutes of downtime allowed) | Must not share fate with the core: served from the CDN |
| NFR4 | Availability: probing + alerting | **99.9%** per month (≈ 43 minutes) | |
| NFR5 | Availability: dashboard + API | **99.5%** per month (≈ 3.6 hours) | Less critical than alerting |
| NFR6 | Freshness | The status page is **at most 60 seconds** behind reality | Publish every 30 s + 30 s CDN cache |
| NFR7 | Scale | Designed for **10,000 monitors × 3 regions × 1-minute interval**; operated at 20–50 monitors | Design for scale, run small |
| NFR8 | Durability | Once a probe hands a result to the queue, it is **never silently lost**: it is processed, or it lands in a dead-letter queue | |
| NFR9 | Tenant isolation | A team can **only ever see its own** monitors, incidents and settings | Enforced in the API and the data layer |
| NFR10 | Probe safety | Probes **cannot reach private or internal addresses** (cloud metadata service, VPC ranges) | SSRF protection, Step 3.5 |
| NFR11 | Cost | Learning mode **≤ $0.45/hour** while running; always-on showcase **≤ $30/month** | |
| NFR12 | Retention | Raw check results kept **30 days**; hourly rollups **1 year**; incidents and their evidence **indefinitely** | Decision D4 |

## 5. Non-goals

What Sentinel deliberately does **not** do, at least for now:
- Not an APM or logging platform: it checks from the outside and doesn't instrument customers' code.
- No multi-step browser checks (logging in, clicking through pages): HTTP checks only.
- No SMS or phone-call alerts (they cost per message).
- No billing or payments; no open public sign-up.
- No customer-hosted (on-premises) probes.

## 6. Capacity estimates

Full worksheet: [capacity-estimates.md](capacity-estimates.md). Headline numbers at the design target:

- **500 checks/s** on average (30,000 in one second if checks aren't spread across the minute)
- **43.2 million rows/day**, **8.64 GB/day** raw, **~260 GB** kept (30 days)
- Worst-case outage detection **140 s**, within the 180 s target (NFR1)
- Extremely **write-heavy**, so: batch writes, daily partitions, rollups for charts, packed queue messages

## 7. Architecture

*Next part of Step 1.1: components, data flow diagram, and the key decisions (ADR-0001, ADR-0002).*

## 8. Open questions

- Probe fan-out: at design scale, can one probe run per region finish ~10,000 checks within a minute,
  or do we split the work across several parallel runs? (Architecture section.)
