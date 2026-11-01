# ADR-0001: Structure sentinel-core as a modular monolith with several roles

- **Status:** Accepted
- **Date:** 2026-11-01
- **Deciders:** Aadi007-git

## Context

Sentinel's core has five responsibilities: **ingesting** results, **evaluating** quorum and incidents,
**notifying**, **summarizing** with AI, and serving the **API**. A publisher and scheduled jobs
(rollups, partition upkeep) come on top.

They have different load profiles. The ingestor handles ~500 writes/s at design scale and must scale
with queue depth; the API is light and scales on CPU; the summarizer is idle most of the time.

The project is built and run by **one person**. Every extra image, pipeline and network hop costs
real time in CI/CD (Step 1.8), Helm (Step 2.4) and observability (Step 3.1).

The probe is **not** part of this decision: it runs on Lambda in three regions under different
constraints. The React dashboard is not part of it either: it's static files.

## Decision drivers

1. Each responsibility must **scale independently** (the ingestor on queue depth, the API on CPU).
2. A failure in one responsibility should **not crash the others**.
3. **Low operational overhead** for a one-person team: few pipelines, images and charts.
4. **Simple consistency:** opening an incident and recording its notification must happen atomically.
5. Keep a **clean path to splitting** later if the team or the load grows.

## Options considered

### Option A: Plain monolith (one process does everything)
- ✅ Simplest to build, run and debug.
- ❌ Can't scale the ingestor without also scaling the API.
- ❌ A crash or memory leak in one part takes down all of them.

### Option B: Microservices (five separately deployed services)
- ✅ Independent deployment and scaling; strong boundaries.
- ❌ 5 images, 5 pipelines, 5 charts: about 5× the CI/CD and Helm work for one person.
- ❌ Calls between parts go over the network and can fail; incident + notification would need a
  distributed pattern (a saga) instead of one database transaction.
- ❌ Solves an organizational problem (many teams) that Sentinel doesn't have.

### Option C: Modular monolith (one codebase and one image, run as several roles)
- ✅ Each role (`sentinel-core api`, `… ingestor`, `… evaluator`, …) is its own Kubernetes Deployment,
  so roles scale and fail independently.
- ✅ One image, one pipeline, one chart.
- ✅ Shared database, so opening an incident and writing its outbox event is **one transaction**.
- ❌ All roles release together: a change to the notifier also redeploys the API.
- ❌ Module boundaries are enforced by discipline and code review, not by the network, so they can
  erode if we're careless.

## Decision

**Option C: modular monolith.** It gives the independent scaling and fault isolation we need
(drivers 1–2) at a fraction of microservices' operational cost (driver 3), keeps incident
handling transactional (driver 4), and its clean module boundaries keep splitting possible (driver 5).

## Consequences

- **Positive:** one Dockerfile and one CI pipeline for the core; per-role Deployments with their
  own replica counts, resource limits and autoscalers; simple, atomic database transactions.
- **Negative:** every release redeploys all roles (mitigated by fast, automated deployments and
  canaries in Step 3.4). Modules must only talk through defined interfaces, never by reaching into
  each other's tables or internals.
- **Follow-ups:** the LLD (Step 1.2) defines module boundaries and interfaces; the Helm chart (Step
  2.4) creates one Deployment per role from the same image.

## Revisit when

- A **second team** needs to release part of the core on its own schedule.
- One role needs a **different runtime or language** (for example, GPU-based AI work).
- One role's **release rhythm or risk** differs so much that redeploying everything becomes painful.
