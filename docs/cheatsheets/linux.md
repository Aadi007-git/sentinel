# Linux shell cheatsheet

> Grows as we go. Every command here is one we actually used, or will use soon in Sentinel.

## Where am I, and what's here?

| Command | What it does |
|---|---|
| `pwd` | Print the folder you're in |
| `ls -la` | List all files, including hidden ones (`.name`), with details |
| `cd ~/projects/sentinel` | Go to a folder (`~` = your home folder, `/home/algomill`) |
| `cd -` | Jump back to the previous folder |
| `tree -a -I .git` | Show folders as a tree, including hidden files, skipping `.git` |

## Files and folders

| Command | What it does |
|---|---|
| `mkdir -p a/b/c` | Create folders, including parents; no error if they already exist |
| `touch file` | Create an empty file (or update its timestamp) |
| `cat file` | Print a file |
| `less file` | Read a long file page by page (Space = next, `/word` = search, `q` = quit) |
| `cp src dst` / `mv src dst` | Copy / move or rename |
| `rm file` | Delete a file. **No recycle bin: it's gone** |
| `echo "text" > file` | Write text to a file (**overwrites** it) |
| `echo "text" >> file` | Append text to the end of a file |

## Combining commands (shell operators)

| Syntax | Meaning | Example |
|---|---|---|
| `a \| b` | **Pipe**: a's output becomes b's input | `gh --version \| head -1` |
| `a && b` | Run b **only if** a succeeded | `sudo apt update && sudo apt install -y jq` |
| `a \|\| b` | Run b **only if** a failed | `cd dir \|\| echo "no such dir"` |
| `a ; b` | Run b after a, regardless | `pwd; ls` |
| `> /dev/null` | Throw away the output | `... \| sudo tee file > /dev/null` |
| `$(command)` | Run a command and paste its output in place | `echo "arch is $(dpkg --print-architecture)"` |
| `{a,b,c}` | **Brace expansion**: becomes three words | `mkdir -p docs/{design,adr,notes}` |
| `\` at line end | Continue the command on the next line | long `curl` commands |

## Packages (apt, Ubuntu's package manager)

| Command | What it does |
|---|---|
| `sudo apt update` | Refresh the list of available packages (doesn't install anything) |
| `sudo apt upgrade -y` | Upgrade installed packages (`-y` = answer yes automatically) |
| `sudo apt install -y jq tree` | Install packages |
| `apt list --installed \| grep jq` | Check whether something is installed |
| `/etc/apt/sources.list.d/` | Where extra repositories are registered |
| `/etc/apt/keyrings/` | Where repository signing keys live (how apt verifies packages) |

## Users, permissions, environment

| Command | What it does |
|---|---|
| `sudo <cmd>` | Run a command as administrator (root) |
| `whoami` | Which user am I? |
| `chmod go+r file` | Give **g**roup and **o**thers **r**ead permission |
| `echo $HOME` / `echo $PATH` | Print an environment variable |
| `command -v uv` | Show where a command lives (or nothing if it isn't installed) |
| `source ~/.bashrc` | Reload shell settings in the current terminal (or just open a new tab) |

`PATH` is the list of folders the shell searches for commands. A newly installed tool "isn't found"
until its folder is on `PATH`: that's why we opened a new tab after installing `uv`.

## Downloading and remote access

| Command | What it does |
|---|---|
| `curl -fsSL URL` | Download quietly; fail on HTTP errors; follow redirects |
| `curl -LsSf URL -o file` | Download to a file instead of the screen |
| `ssh-keygen -t ed25519 -C "label"` | Create an SSH key pair (private key `~/.ssh/id_ed25519`, public key `id_ed25519.pub`) |
| `ssh -T git@github.com` | Test that GitHub accepts your SSH key |

## Gotchas

- **`sudo echo "x" > /etc/file` fails** with "Permission denied". `sudo` applies to `echo`, but the
  `>` redirect is done by *your* unprivileged shell. Use `echo "x" | sudo tee /etc/file > /dev/null`.
  That's why the GitHub CLI install used `tee`.
- **`curl ... | sh` runs code you haven't read.** Download it, skim it with `less`, then run it.
- **Work in `~/`, not `/mnt/c` or `/mnt/d`.** Windows drives mounted in WSL are much slower for Git
  and Docker, and can mess up file permissions.
- **Line endings:** Windows uses CRLF (`\r\n`), Linux uses LF (`\n`). A shell script saved with CRLF
  fails with odd errors like `$'\r': command not found`. `git config --global core.autocrlf input` helps.
- **`rm` has no undo.** Pause before `rm -r`, and never run `rm -rf` with a variable that might be empty.
- **Passphrase prompt on every push?** An `ssh-agent` remembers it for the session. We'll set that up
  when it gets annoying.
