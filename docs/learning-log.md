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

**Went wrong → fixed**
- The first commit was never pushed: `git log` showed `(HEAD -> main)` but no `origin/main`, and
  `git branch -vv` showed no upstream. Fix: `git push -u origin main`.

**Learned (in my own words)**
- …

**Notes / cheatsheets written**
- [notes/devops-fundamentals.md](notes/devops-fundamentals.md)
- [cheatsheets/linux.md](cheatsheets/linux.md), [cheatsheets/git.md](cheatsheets/git.md)

**Next:** Step 1.1, System design I: HLD (requirements + capacity estimates).
