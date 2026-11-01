# Git cheatsheet

> Grows as we go. Step 1.3 adds branches, pull requests, conventional commits and hooks.

## One-time setup

| Command | What it does |
|---|---|
| `git config --global user.name "Name"` | Name stamped on your commits |
| `git config --global user.email "you@example.com"` | Email stamped on your commits (use the one on your GitHub account) |
| `git config --global init.defaultBranch main` | New repos start on `main` |
| `git config --global core.autocrlf input` | Store LF line endings in Git even if a Windows editor writes CRLF |
| `git config --global --list` | Show all your global settings |

## The everyday loop

```
working folder ──git add──> staging area ──git commit──> local repo ──git push──> GitHub
 (your edits)              (what goes in the                (history on        (shared
                            next commit)                     your machine)       copy)
```

| Command | What it does |
|---|---|
| `git init` | Turn the current folder into a Git repo (creates `.git/`) |
| `git status` | What's changed, what's staged, what's untracked. **Run it constantly** |
| `git status -sb` | Short status; also shows whether you're ahead of or behind the remote |
| `git add <file>` / `git add .` | Stage changes for the next commit |
| `git commit -m "type: message"` | Save staged changes as a commit in *local* history |
| `git log --oneline` | Compact history; decorations like `(HEAD -> main, origin/main)` show where branches point |
| `git diff` / `git diff --staged` | Show unstaged / staged changes line by line |

## Remotes (GitHub)

| Command | What it does |
|---|---|
| `git remote -v` | List remotes (`origin` = the default name for "the GitHub copy") |
| `git push -u origin main` | Push `main` to `origin` **and** remember it as the upstream (`-u`); after that, plain `git push` works |
| `git branch -vv` | Show each branch and the remote branch it tracks, if any |
| `gh repo create NAME --public --source=. --remote=origin --push` | Create a GitHub repo from the current folder, add the remote and push |
| `ssh -T git@github.com` | Test SSH authentication to GitHub |

## .gitignore

| Pattern | Matches |
|---|---|
| `.env` | A file or folder named `.env`, anywhere |
| `.env.*` | `.env.local`, `.env.prod`, … |
| `!.env.example` | **Exception**: do NOT ignore this file, even though `.env.*` matches it |
| `*.pem` | Any file ending in `.pem` |
| `.venv/` | A folder named `.venv` (the trailing `/` = folders only) |
| `.vscode/*` + `!.vscode/extensions.json` | Ignore a folder's contents except one file |

| Command | What it does |
|---|---|
| `git check-ignore -v <file>` | Show which `.gitignore` rule ignores a file (prints nothing if it isn't ignored) |
| `git status --ignored` | Also list the ignored files |

## Gotchas

- **Committed ≠ pushed.** `git commit` only saves locally. If `git log` shows `(HEAD -> main)` but no
  `origin/main`, nothing is on GitHub yet. *(This happened in Step 1.0; fixed with `git push -u origin main`.)*
- **`.gitignore` only affects untracked files.** If a file was already committed, adding it to
  `.gitignore` does nothing; you'd need `git rm --cached <file>` too. That's why `.gitignore` comes **first**.
- **A secret pushed to a public repo is compromised**, even if you delete it a minute later: bots scan
  GitHub in seconds and history keeps it. The fix is to **rotate** (replace) the secret, not just delete the commit.
- **Comments in `.gitignore` must be on their own line.** `node_modules/  # deps` silently breaks the rule.
- **Git doesn't track empty folders.** Use a placeholder file such as `.gitkeep`.

## Conventional commits (preview, full lesson in Step 1.3)

`type: short description in the imperative mood`

| Type | Use for |
|---|---|
| `feat:` | New feature |
| `fix:` | Bug fix |
| `docs:` | Documentation only |
| `chore:` | Maintenance (setup, config, tidying) |
| `ci:` | CI pipeline changes |
| `refactor:` / `test:` | Code restructuring / tests |
