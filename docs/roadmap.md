# Sentinel roadmap

Sentinel is a **multi-region uptime monitoring service**, built step by step as a hands-on DevOps,
cloud and system-design project. Progress is tracked with the checkboxes below; the day-by-day
story is in [learning-log.md](learning-log.md).

## What Sentinel does

Users add websites or APIs to watch. **Probes in 3 AWS regions** (us-east-1, eu-west-1, ap-south-1)
check each one every minute, recording status, response time by stage (DNS → connect → TLS → first
byte) and SSL certificate expiry.

An outage is **confirmed only when 2 of 3 regions agree** (a quorum), so one region's network glitch
never pages anyone. Sentinel then opens an incident, alerts by Slack / Discord / email, updates a
**public status page**, and has **AI draft a plain-English incident update** that a human approves.

Sentinel also **watches itself** with an independent heartbeat (a "dead man's switch").

## Architecture (first draft, refined in Step 1.1)

```
 ┌───────────── Probe regions: AWS Lambda, outside any VPC ──────────────┐
 │  us-east-1              eu-west-1              ap-south-1             │
 │  scheduler ─> probe     scheduler ─> probe     scheduler ─> probe     │
 └──────┬───────────────────────┬────────────────────────┬──────────────┘
        │ reads the monitor list from a snapshot file in S3
        └──────────── check results (batched) ───────────┘
                                │  sent across regions
                                v
 Core region  [SQS results queue] ──(messages that keep failing)──> [dead-letter queue]
 (EKS)                          │
                ┌───────────────v────────────────────────────┐
                │ sentinel-core: one image, several roles    │
                │  ingestor   → saves check results           │──> [RDS Postgres]
                │  evaluator  → 2-of-3 quorum, open/close     │     results partitioned by day,
                │               incidents                     │     hourly rollups
                │  notifier   → Slack / Discord / email       │
                │  summarizer → AI incident update (LLM)      │
                │  api        → monitors, incidents, auth     │
                └───────────────┬────────────────────────────┘
                                │ publishes status JSON every 30s
                                v
                     [S3 + CloudFront CDN] ──> public status pages
                     (still served even if the core is down)
 Dashboard: [web: React] ──> api
 Who watches the watcher: external heartbeat + Sentinel monitoring its own endpoints
```

**AI model:** pluggable. A fake model in tests and CI, Ollama locally, the Claude API in AWS.

**Design target vs real use:** designed for 10,000 monitors × 3 regions × 1 check/min =
**500 checks/s, ~43M rows/day**; actually run with 20–50 monitors.

## How the work is split

- **I design**: HLD and LLD documents, diagrams and ADRs.
- **Claude implements the application code** (Python and React) from my design and walks me through it.
- **I write every DevOps and infrastructure file**: Dockerfiles, Compose, CI/CD, Kubernetes, Helm,
  Terraform, Argo CD, dashboards, alerts, policies.
- **Every step:** concept → design → build → review → verify → break it → checkpoint questions →
  note in `docs/notes/` + cheatsheet in `docs/cheatsheets/`.

---

## Phase 1: Design, build, containerize, CI (laptop + GitHub)

Goal: a written design; the whole system runs with `docker compose up`; every PR is linted, tested
and security-scanned; every merge produces versioned, scanned images.

- [x] **1.0 Orientation + setup**: DevOps, CALMS, DORA metrics, Linux shell, WSL, `gh`, `uv`, SSH, repo
- [ ] **1.1 System design I: HLD**: requirements, capacity estimates, components, ADR-0001, ADR-0002
- [ ] **1.2 System design II: LLD**: API design, data model, class design + patterns, sequence and state diagrams
- [ ] **1.3 Git like a team**: trunk-based development, PRs, conventional commits, branch protection, pre-commit hooks
- [ ] **1.4 The app (12-factor)**: probe, core and web implemented from the LLD; run without Docker
- [ ] **1.5 Docker fundamentals**: multi-stage, non-root, slim images; Lambda container image
- [ ] **1.6 Docker Compose**: full system with 3 simulated probe "regions"
- [ ] **1.7 Testing**: quorum state-machine tests, contract tests, Testcontainers
- [ ] **1.8 CI with GitHub Actions**: lint → test → build → push to GHCR, path filters, caching
- [ ] **1.9 DevSecOps**: gitleaks, Trivy, Semgrep, SBOM, Dependabot
- [ ] **1.10 AI in the dev loop**: AI PR reviewer, LLM evals in CI

**Phase 1 demo:** take a target down in Compose → 2 of 3 regions agree → incident → AI update →
Discord alert; a PR with a vulnerable package is blocked by CI.

## Phase 2: Kubernetes, multi-region AWS with Terraform, GitOps

Goal: `terraform apply` builds the platform from nothing; merge → dev automatically; prod via PR;
no secrets in Git; status page on the CDN.

- [ ] **2.0 System design III**: failure domains, quorum, delivery guarantees, idempotency, CAP/PACELC, static stability
- [ ] **2.1 Kubernetes core** on a local kind cluster
- [ ] **2.2 State + jobs**: StatefulSets, migration Job, rollup and partition CronJobs
- [ ] **2.3 Traffic in**: Gateway API + Envoy Gateway
- [ ] **2.4 Helm**: the `sentinel` chart with dev/prod values
- [ ] **2.5 AWS + Terraform foundations**: account hardening, budgets, VPC, EKS, RDS, Terraform CI
- [ ] **2.6 Multi-region probes**: Lambda ×3 regions, ECR replication, EventBridge Scheduler, SQS + DLQ
- [ ] **2.7 GitOps with Argo CD**: `sentinel-gitops` repo, ApplicationSet, promotion by PR
- [ ] **2.8 Secrets + identity**: GitHub OIDC, External Secrets + Secrets Manager, EKS Pod Identity
- [ ] **2.9 Status pages at the edge**: S3 + CloudFront, optional custom domain

**Cost control:** local kind cluster until 2.4. On AWS (~$0.30–0.45/hour while running), we
`terraform destroy` after every session; budget alarms at $10 / $25 / $50.

## Phase 3: Operate it like production

Goal: full visibility from probe to alert, autoscaling, resistance to abuse and bad releases,
survives a region failure, and runs live 24/7.

- [ ] **3.0 System design IV**: timeouts, retries + jitter, circuit breaker, bulkhead, outbox, locking
- [ ] **3.1 Observability**: OpenTelemetry → Prometheus, Loki, Tempo, Grafana; one trace from Lambda to alert
- [ ] **3.2 SLOs + who watches the watcher**: burn-rate alerts, dead man's switch, runbooks
- [ ] **3.3 Autoscaling**: KEDA on SQS depth, HPA, Karpenter, k6 load test
- [ ] **3.4 Progressive delivery**: Argo Rollouts canary; Lambda alias canary
- [ ] **3.5 Security hardening**: Cosign, Kyverno, NetworkPolicies, RBAC, SSRF protection, rate limiting
- [ ] **3.6 AI in operations**: LLM evals, cost tracking, fallback; read-only Kubernetes MCP server with guardrails
- [ ] **3.7 Chaos + DR game day**: Chaos Mesh, region outage drill, RDS restore, postmortem
- [ ] **3.8 FinOps + showcase launch**: OpenCost; always-on low-cost deployment; portfolio README

**Phase 3 demo:** load test → autoscaling; a region goes dark → no false alarm; a real outage →
alert within 2 minutes + AI update; a bad canary → automatic rollback; the core goes down → the
status page stays up.

---

## System-design concepts and where they appear

| Concept | Where in Sentinel | Step |
|---|---|---|
| Requirements, back-of-envelope estimation | Sizing 500 checks/s, storage, retention | 1.1 |
| HLD, sync vs async, modular monolith vs microservices | Overall architecture, ADR-0001 | 1.1 |
| Queue vs Kafka vs direct HTTP | Probe → core results path, ADR-0002 | 1.1 |
| REST API design, idempotency keys | Monitor and incident APIs | 1.2 |
| Data modelling, indexes, partitioning | monitors, check_results, incidents, outbox | 1.2 |
| SOLID + design patterns (Strategy, Factory, Repository, State, Adapter) | Notifiers, check types, LLM providers, incident lifecycle | 1.2 |
| Sequence and state diagrams | Check → incident → alert; UP/DOWN/UNKNOWN | 1.2 |
| Failure domains, quorum | 3 probe regions, 2-of-3 rule | 2.0 |
| Delivery guarantees, idempotency, DLQs | SQS results, dedup by `check_id` | 2.0 |
| CAP / PACELC | Status page may be 30s stale | 2.0 |
| Horizontal scaling, L4/L7 load balancing | core pods behind Gateway API | 2.1–2.3 |
| DB scaling: partitioning, rollups, pooling | check_results by day | 2.2 |
| Caching, CDN, static stability | Status pages on S3 + CloudFront | 2.9 |
| Reliability patterns | Probe concurrency, notifier retries, LLM fallback | 3.0 |
| Outbox, locking, leader election | Reliable alerts, one evaluator per monitor | 3.0 |
| Rate limiting, SSRF | Public API, probe URL safety | 3.5 |
| DR: RTO / RPO | Restore drill, region outage drill | 3.7 |
| Cost-aware design | Lambda vs containers, NAT, showcase tier | 3.8 |
