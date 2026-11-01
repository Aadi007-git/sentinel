# Learning log

One entry per session: what we did, what I learned, what went wrong, what's next.
The plan and progress checklist live in [roadmap.md](roadmap.md).

---

## 2026-11-01: Step 1.0 Orientation + setup

**Did**
- Learned what DevOps is: the wall of confusion, CALMS, the delivery loop, DORA metrics.
- Installed `jq`, `tree`, `gh` (from GitHub's signed apt repository) and `uv` (read the installer
  before running it) in WSL Ubuntu 26.04.
- Configured Git (`init.defaultBranch main`, `core.autocrlf input`).
- Created an ed25519 SSH key and authenticated `gh` and Git to GitHub (`Aadi007-git`).
- Created `~/projects/sentinel` with the `docs/` skeleton and a `.gitignore`; first commit `d82827d`.
- Pushed to GitHub (`Aadi007-git/sentinel`, public) as 3 commits, each one a single logical change.

**Went wrong → fixed**
- The first commit was never pushed: `git log` showed `(HEAD -> main)` but no `origin/main`, and
  `git branch -vv` showed no upstream. Fix: `git push -u origin main`. Habit: after pushing, check
  `git log --oneline` for `origin/main`.
- `.gitignore` had `*.tfstate.backup`, which misses Terraform's timestamped backups
  (`terraform.tfstate.<timestamp>.backup`). Changed it to `*.tfstate.*` and proved it with
  `git check-ignore -v` (commit `190675b`).

**Learned**
- DevOps is a way of working, not a tool or a job title: the team that builds a service also runs
  it, and automation removes the handoffs between them.
- Speed and stability aren't a trade-off: small, frequent changes are easier to review, test and
  roll back, which is why elite teams score well on all four DORA metrics.
- Supply-chain thinking starts on day one: apt checks packages against a signing key, and an
  installer script should be read before it runs (no blind `curl | sh`).
- Git only ignores files that aren't tracked yet, so `.gitignore` comes before the first commit;
  and a secret pushed to a public repo must be rotated, not just deleted.
- Committed is not the same as pushed: `origin/main` in `git log` is the proof a commit reached GitHub.

**Notes / cheatsheets written**
- [notes/devops-fundamentals.md](notes/devops-fundamentals.md)
- [cheatsheets/linux.md](cheatsheets/linux.md), [cheatsheets/git.md](cheatsheets/git.md)

**Next:** Step 1.1, System design I: HLD (requirements + capacity estimates).

---

## 2026-11-01: Step 1.1 System design I: HLD (part 1: requirements + estimates)

**Did**
- Learned what HLD vs LLD is, how design docs work, and the 5-stage system-design framework.
- Made 4 product decisions: **D1** invite-only teams (multi-tenant), **D2** 1-minute interval,
  **D3** 2 of 3 regions failing for 2 consecutive rounds, **D4** 30 days of raw data + 1 year of rollups.
- Wrote 12 functional and 12 measurable non-functional requirements in `docs/design/HLD.md`.
- Completed the capacity-estimates worksheet: 500 checks/s, 43.2M rows/day, 8.64 GB/day,
  ~260 GB retained, 140 s worst-case detection.

**Went wrong → fixed**
- Labelled byte counts as GB (wrote 8,640,000,000 "GB" for 8.64 GB). Fix: group digits in threes;
  10 digits ≈ GB.
- Rollup storage used the raw row count (43.2M) instead of rollup rows (240,000): 1.6 TB instead
  of 8.8 GB.
- Didn't know how to build up the worst-case detection time. Walked through it as a timeline:
  60 + 60 + 10 + 10 = 140 s.

**Learned**
- Estimates decide the architecture: write-heavy at 500/s means batch writes, daily partitions,
  rollups, and dropping partitions instead of deleting rows.
- Average ≠ peak: synchronized schedules create spikes (30,000 checks in one second); spreading
  with `hash(id) % 60` flattens them. The same idea as jitter on retries.
- Pre-aggregation (rollups) trades detail for cheap reads: ~360× less storage per year.
- A CDN makes origin load independent of viewer count, which is critical for a status page during an outage.
- Estimates surface cost surprises: one result per queue message would cost ~$150/month in SQS
  requests at design scale; packing results per message cuts that ~100×.
- Worst-case timeline reasoning proves whether an NFR is achievable before building anything.

**Notes / cheatsheets written**
- [design/HLD.md](design/HLD.md) (sections 1–6), [design/capacity-estimates.md](design/capacity-estimates.md)
- [cheatsheets/system-design.md](cheatsheets/system-design.md)

**Next:** Step 1.1 part 2: architecture components + data-flow diagram, ADR-0001 (modular monolith
vs microservices), ADR-0002 (how results travel from probe to core).
