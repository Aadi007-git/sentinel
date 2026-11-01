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
