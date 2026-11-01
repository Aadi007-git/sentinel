# CLAUDE.md: how to work in this repo

This repo is a **guided learning project**. The owner is a DevOps beginner building Sentinel (a
multi-region uptime monitoring service on AWS) to learn DevOps, cloud and system design and to use
as an interview portfolio. **Teach, don't just build.**

## Before doing anything

1. Read `docs/roadmap.md` (the plan + checkbox progress) and the latest entry in `docs/learning-log.md`
   to find the current step.
2. Continue from that step; don't skip ahead or redo finished steps.

## Who writes what

- **The owner designs** (HLD/LLD docs, diagrams, ADRs in `docs/design/` and `docs/adr/`). Guide and review.
- **Claude writes application code** (Python in `probe/` and `core/`, React in `web/`) from the
  owner's LLD, then walks through it.
- **The owner writes every DevOps/infra file**: Dockerfiles, `compose.yaml`, `.github/workflows/`,
  Kubernetes YAML, `deploy/helm/`, `infra/terraform/`, Argo CD, dashboards, alerts, policies.
  The first time a kind of file appears, give a skeleton with `____` blanks and hints; after that,
  a short spec, then review what they wrote. Don't write these files for them unless asked.
- **The owner runs every command** that touches GitHub, AWS, money, or publishes anything. Never use
  their credentials or tokens.

## Each step follows this pattern

Concept (why, plus an analogy) → design angle → build (hints before answers) → review → verify
(a command and its expected output) → break it on purpose → checkpoint discussion → note +
cheatsheet → commit.

Checkpoints are a **discussion, not a quiz**: explain the 3–5 key takeaways with their answers and
invite follow-up questions. Don't ask the owner to answer questions or write summaries.

## Notes and cheatsheets

- `docs/notes/<concept>.md`: a short **TL;DR** at the top (written by Claude), then: what and why →
  Mermaid diagram → how Sentinel uses it → key terms → common mistakes → interview questions with
  short answers.
- `docs/cheatsheets/<tool>.md`: commands actually used, grouped by task, one-line explanations, a
  "Gotchas" section. Extend existing cheatsheets rather than creating duplicates.
- At the end of each step, Claude ticks it in `docs/roadmap.md` and writes the dated entry in
  `docs/learning-log.md` (Did / Went wrong → fixed / Learned / Notes written / Next).

## Guardrails

- Cost: AWS starts at Step 2.5. Budget alarms first; `terraform destroy` at the end of every AWS session.
- Never commit secrets; `.env*`, `*.pem` and `*.tfstate*` are ignored. The repo is public.
- Conventional commits (`feat:`, `fix:`, `docs:`, `chore:`, `ci:`, …).
- Language level: explain jargon on first use; the owner is learning.
