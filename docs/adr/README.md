# Architecture Decision Records (ADRs)

An ADR is a short document that captures **one important decision**: the context, the options we
considered, what we chose, and the consequences we accept. Code shows *what* was built; ADRs explain
*why*, so that six months from now, or in an interview, nobody has to guess.

## Rules

- **One decision per ADR**, numbered in order (`0001-…`, `0002-…`).
- **Accepted ADRs are not edited.** If we change our minds, we write a new ADR that **supersedes** the
  old one and mark the old one `Superseded by ADR-00XX`. History stays honest.
- Keep each one short: if it takes more than a page or two, it's probably two decisions.

## Index

| ADR | Title | Status | Date |
|---|---|---|---|
| [0001](0001-modular-monolith-core.md) | Structure sentinel-core as a modular monolith with several roles | Accepted | 2026-11-01 |
| [0002](0002-sqs-for-check-results.md) | Use SQS (with a dead-letter queue) to move check results from probes to the core | Accepted | 2026-11-01 |
| [0003](0003-probe-fan-out-with-delay-queue.md) | Spread and fan out probe checks with a dispatcher and a delay queue | Accepted | 2026-11-01 |

## Template

```markdown
# ADR-00XX: <decision, phrased as an action>

- **Status:** Proposed | Accepted | Superseded by ADR-00YY
- **Date:** YYYY-MM-DD
- **Deciders:** <names>

## Context
What problem are we solving? What constraints and requirements apply (link to the NFRs)?

## Decision drivers
The 3–5 things that matter most for this choice.

## Options considered
### Option A: <name>
Pros / cons.
### Option B: <name>
Pros / cons.

## Decision
What we chose, in one or two sentences, and the main reason.

## Consequences
- Positive: …
- Negative (the costs we accept): …
- Follow-ups: …

## Revisit when
The signals that would make us reconsider.
```
