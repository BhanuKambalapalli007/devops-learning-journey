# Git complete reference — Week 1
**Bhanu Prakash · DevOps Learning Journey · Phase 01**

---

## Table of contents

1. [Core concepts](#1-core-concepts)
2. [Initial setup](#2-initial-setup)
3. [Repository basics](#3-repository-basics)
4. [Staging and committing](#4-staging-and-committing)
5. [Reading changes with git diff](#5-reading-changes-with-git-diff)
6. [The .gitignore file](#6-the-gitignore-file)
7. [Undoing changes](#7-undoing-changes)
8. [Remote repositories and GitHub](#8-remote-repositories-and-github)
9. [History recovery tools](#9-history-recovery-tools)
10. [Professional workflow](#10-professional-workflow)
11. [Commit message conventions](#11-commit-message-conventions)
12. [Quick reference card](#12-quick-reference-card)

---

## 1. Core concepts

### 1.1 What is Git?

Git is a **distributed version control system**. It tracks every change made
to files, stores that history permanently, and allows multiple engineers to
work on the same codebase simultaneously without overwriting each other's work.

In DevOps, Git is the starting point for everything:
- Every CI/CD pipeline is triggered by a `git push`
- Every deployment begins with a commit
- Every infrastructure change (Terraform, Kubernetes) lives in a Git repo

### 1.2 The 3-tree model

This is the most important mental model in Git. At any moment, Git manages
your project across three distinct areas:

```
┌─────────────────────┐       ┌─────────────────────┐       ┌─────────────────────┐
│                     │       │                     │       │                     │
│  Working directory  │──────▶│   Staging area      │──────▶│    Repository       │
│                     │git add│   (Index)           │git    │    (.git folder)    │
│  Files you edit     │       │   Changes selected  │commit │    Saved history    │
│                     │       │                     │       │                     │
└─────────────────────┘       └─────────────────────┘       └─────────────────────┘
         ▲                                                             │
         └─────────────────── git restore ◀───────────────────────────┘
                                (undo changes)
```

| Area | Description | How to reach it |
|---|---|---|
| Working directory | Files on your disk that you edit | Just edit any file |
| Staging area (Index) | Changes selected for the next commit | `git add <file>` |
| Repository | Permanent snapshot history in `.git/` | `git commit` |

### 1.3 Key terminology

| Term | Meaning |
|---|---|
| **Repository (repo)** | A project folder tracked by Git, containing a `.git/` directory |
| **Commit** | A permanent snapshot of staged changes with a message and unique hash |
| **Hash (SHA)** | A 40-character ID unique to every commit, e.g. `a3f9c12` |
| **HEAD** | A pointer to the commit you are currently on (usually the latest) |
| **Branch** | A movable pointer to a line of commits |
| **Remote** | Another copy of the repository (e.g. on GitHub) |
| **Origin** | The conventional name for the primary remote |
| **Upstream** | The remote branch that a local branch tracks |
| **Untracked** | A file Git sees but has never been told to manage |
| **Staged** | A change moved to the staging area, ready to be committed |
| **Modified** | A tracked file that has been changed but not yet staged |
| **Clean working tree** | No uncommitted or unstaged changes exist — ideal state |

---

## 2. Initial setup

### 2.1 Install Git

```bash
# Ubuntu / Debian / WSL
sudo apt update && sudo apt install git -y

# Verify
git --version
```

### 2.2 Global configuration (one-time setup)

These settings are stored in `~/.gitconfig` and apply to every repository
on your machine.

```bash
# Set your identity — every commit is stamped with this
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# Set default branch name to modern standard
git config --global init.defaultBranch main

# Set your preferred editor (vim, nano, or code)
git config --global core.editor "vim"

# Verify all settings
git config --list
```

### 2.3 SSH key authentication for GitHub

GitHub requires SSH keys instead of passwords (removed in 2021).

```bash
# Generate a new SSH key
ssh-keygen -t ed25519 -C "your@email.com"
# Press Enter three times (accept defaults, no passphrase)

# View your PUBLIC key — this is what you give to GitHub
cat ~/.ssh/id_ed25519.pub
# Starts with: ssh-ed25519 AAAA...

# Test GitHub connection after adding the key on GitHub
ssh -T git@github.com
# Expected: Hi YourUsername! You've successfully authenticated...
```

**Critical rule:** `~/.ssh/id_ed25519` is your **private key** — never share
it. `~/.ssh/id_ed25519.pub` is your **public key** — safe to share with any
server or service.

```
Private key  (~/.ssh/id_ed25519)      = Your house key. Never leaves your machine.
Public key   (~/.ssh/id_ed25519.pub)  = Your house lock. Install it everywhere.
```

---

## 3. Repository basics

### 3.1 Creating a repository

```bash
# Initialise a new repository in the current folder
git init
# Creates a hidden .git/ folder — this IS the repository

# Clone an existing repository from GitHub
git clone git@github.com:Username/repo-name.git
# Downloads full history and configures remote automatically
```

### 3.2 Checking status

```bash
# Full status — verbose, human-readable
git status

# Short status — compact, great for quick checks
git status -s
git status --short
```

**Short status output guide:**

```
MM app.py        ← Left M (green) = staged | Right M (red) = also unstaged
 M config.py     ← Only right column = unstaged changes only
M  utils.py      ← Only left column = staged, nothing extra unstaged
?? newfile.py    ← Untracked — Git has never seen this file
A  added.py      ← New file that is staged
D  deleted.py    ← File deleted and staged
R  old → new.py  ← File renamed (with git mv)
```

### 3.3 Viewing commit history

```bash
# Compact one-line view — use this daily
git log --oneline

# Full detail — author, date, full hash, message
git log

# With branch/tag/remote labels shown
git log --oneline --decorate

# Visual branch graph (useful in Week 2)
git log --oneline --decorate --graph --all

# Show last N commits only
git log --oneline -5

# Show what files changed in each commit
git log --oneline --stat
```

---

## 4. Staging and committing

### 4.1 The git add variants

```bash
# Stage a specific file
git add filename.md

# Stage all changes in current directory and subdirectories
git add .

# Stage all changes including deletions everywhere in the repo
git add -A

# Stage all files of a specific type
git add *.md

# Interactive partial staging (most powerful — see below)
git add -p filename.md
git add --patch filename.md
```

### 4.2 Partial staging with `git add -p`

`git add -p` breaks your changes into "hunks" and asks you what to stage
piece by piece. This lets one editing session produce multiple focused commits.

```bash
git add -p filename.md
```

**Hunk prompt options:**

| Key | Action |
|---|---|
| `y` | Stage this hunk |
| `n` | Skip this hunk (leave unstaged) |
| `s` | Split hunk into smaller pieces |
| `e` | Manually edit the hunk in vim |
| `q` | Quit — stop asking, don't stage remaining hunks |
| `?` | Show all available options |

**When `s` fails (cannot split):** Use `e` to open vim and manually delete
the `+` lines you don't want staged. Git keeps deleted lines as unstaged
changes automatically. Save with `:wq`.

**Result of partial staging** — `git status -s` shows `MM`:
```
MM linux-notes.md
││
│└── Red M  = unstaged changes still in working directory
└─── Green M = staged changes ready to commit
```

### 4.3 git commit

```bash
# Commit staged changes with a message
git commit -m "type: description"

# Open editor to write a longer commit message
git commit

# Stage ALL tracked files and commit in one step
# (does not include untracked files)
git commit -a -m "type: description"

# Fix the most recent commit message (before pushing)
git commit --amend -m "corrected message"

# Add more staged changes to the most recent commit (before pushing)
git add forgotten-file.md
git commit --amend --no-edit
```

**Important:** `--amend` rewrites the commit with a new hash. Only safe on
commits that have **not yet been pushed** to a shared remote branch.

---

## 5. Reading changes with git diff

### 5.1 The three essential variants

```bash
# What have I changed but NOT staged yet?
# (working directory vs staging area)
git diff

# What am I ABOUT to commit?
# (staging area vs last commit) ← run this before every commit
git diff --staged
git diff --cached    # same thing, older alias

# Everything changed since the last commit?
# (working directory + staging vs last commit)
git diff HEAD
```

### 5.2 Comparing commits

```bash
# Compare last two commits
git diff HEAD~1 HEAD

# Compare specific commits by hash
git diff a3f9c12 b532acb

# Compare commits using relative notation
git diff HEAD~3 HEAD~1   # three commits ago vs one commit ago

# What changed in a specific file between commits
git diff HEAD~1 HEAD -- filename.md
```

### 5.3 Reading diff output

```diff
diff --git a/README.md b/README.md
index 7b488ab..c2c19e3 100644
--- a/README.md          ← old version (last commit / staging area)
+++ b/README.md          ← new version (staging area / working dir)
@@ -11,3 +11,6 @@       ← line numbers: old file @ line 11 | new file @ line 11
 unchanged line          ← no symbol = context line, unchanged
-removed line            ← red minus = line that was deleted
+added line              ← green plus = line that was added
```

**`@@ -11,3 +11,6 @@` means:**
- Old file: starts at line 11, shows 3 lines
- New file: starts at line 11, shows 6 lines (3 more added)

---

## 6. The .gitignore file

### 6.1 Purpose

`.gitignore` tells Git to permanently ignore certain files and folders.
Files listed here will never appear in `git status` or be committed.

**Critical rule:** Never commit `.env` files, API keys, passwords, private
keys, or secrets into Git — even in a private repository.

### 6.2 Common patterns

```gitignore
# OS files
.DS_Store
Thumbs.db

# Editor files
.vscode/
.idea/
*.swp
*.swo

# Python
__pycache__/
*.py[cod]
.venv/
venv/

# Environment and secrets — NEVER commit these
.env
.env.*
*.key
*.pem
secrets.yaml

# Logs
*.log
logs/

# Build outputs
dist/
build/
*.zip
*.tar.gz

# Terraform (Phase 05)
.terraform/
*.tfstate
*.tfstate.backup

# Docker (Phase 02)
*.tar
```

### 6.3 Pattern syntax cheatsheet

| Pattern | Matches |
|---|---|
| `*.log` | Any file ending in `.log` anywhere in project |
| `logs/` | The entire `logs/` directory |
| `/CHANGELOG` | Only the `CHANGELOG` file in the root, not subdirectories |
| `doc/*.txt` | `.txt` files in `doc/` but not `doc/sub/` |
| `**/temp` | `temp` folder at any depth in the project |
| `!important.log` | Exception — do NOT ignore this specific file |

### 6.4 Verify what is being ignored

```bash
# Show all ignored files explicitly
git status --ignored

# Check if a specific file would be ignored
git check-ignore -v filename.log
```

---

## 7. Undoing changes

This is the most critical section. Using the wrong undo command can make
things worse. Match the command to the situation:

### Decision guide

```
What is the situation?
│
├── "I edited a file and want to throw away the change (never staged)"
│   └── git restore <file>                    ⚠️  Irreversible locally
│
├── "I staged the wrong file, want to unstage it"
│   └── git restore --staged <file>           ✅ Always safe
│
├── "My last commit has the wrong message (not pushed yet)"
│   └── git commit --amend -m "new message"   ⚠️  Only before pushing
│
├── "I committed too early, want to add more (not pushed yet)"
│   └── git reset --soft HEAD~1               ⚠️  Only before pushing
│
├── "I need to undo N local commits entirely (not pushed yet)"
│   └── git reset --hard HEAD~N               🔴 Destroys changes permanently
│
└── "I need to undo a commit that is already on a shared branch"
    └── git revert <hash>                     ✅ Always safe — preserves history
```

### 7.1 git restore — discard working directory changes

```bash
# Discard all changes to one file — go back to last committed version
git restore filename.md

# Discard all unstaged changes in the entire repo
git restore .

# Restore a file to a specific commit's version
git restore --source=HEAD~2 filename.md
```

⚠️ **Irreversible:** The change was never committed, so Git has no record of
it. Once restored, that edit is gone permanently.

### 7.2 git restore --staged — unstage a file

```bash
# Move a file from staging area back to working directory
# The change is NOT lost — just unstaged
git restore --staged filename.md

# Unstage everything at once
git restore --staged .
```

✅ **Always safe:** This never loses any work — it only moves changes between
areas.

### 7.3 git reset — undo commits

`git reset` moves the HEAD pointer backward, uncommitting changes. Three modes:

```bash
# ── SOFT RESET ──────────────────────────────────────────────────────
# Undo last commit. Changes go BACK to the staging area.
# Use when: committed too early, want to add more before recommitting.
git reset --soft HEAD~1

# ── MIXED RESET (default) ────────────────────────────────────────────
# Undo last commit. Changes go back to working directory (unstaged).
# Use when: want to re-stage selectively before recommitting.
git reset HEAD~1
git reset --mixed HEAD~1   # explicit form

# ── HARD RESET ───────────────────────────────────────────────────────
# Undo commit AND permanently discard all changes.
# Use when: that commit was completely wrong and must be gone entirely.
git reset --hard HEAD~1

# Undo multiple commits (replace 1 with any number)
git reset --soft HEAD~3    # undo last 3 commits, keep changes staged
git reset --hard HEAD~3    # undo last 3 commits, discard all changes
```

**The golden rule of git reset:**
> Never reset commits that have already been pushed to a shared remote branch.
> Your teammates still have those commits — your histories will diverge.
> Reset is only safe on commits that exist ONLY on your local machine.

### 7.4 git revert — safe undo for shared history

```bash
# Create a new commit that undoes a specific commit
# The original commit stays in history — a new "undo" commit is added on top
git revert <commit-hash>

# Revert the most recent commit
git revert HEAD

# Revert without opening vim for the message
git revert HEAD --no-edit
```

✅ **Always safe on shared branches:** History is never rewritten. Every
change, including the undo, is recorded in the audit trail — required in
production systems.

```
Before revert:   A ← B ← C (broken)
After revert:    A ← B ← C (broken) ← D (reverts C)
```

---

## 8. Remote repositories and GitHub

### 8.1 Managing remotes

```bash
# Add a remote called 'origin' (the standard name)
git remote add origin git@github.com:Username/repo-name.git

# View all configured remotes
git remote -v
# origin  git@github.com:... (fetch)
# origin  git@github.com:... (push)

# Rename a remote
git remote rename old-name new-name

# Remove a remote
git remote remove origin

# Change the URL of an existing remote
git remote set-url origin git@github.com:Username/new-repo.git
```

### 8.2 Push, pull, fetch

```bash
# First push — -u sets origin/main as the default upstream branch
git push -u origin main

# All subsequent pushes (after -u is set)
git push

# Push a specific branch
git push origin branch-name

# Force push after rebase (use --force-with-lease, not --force)
git push --force-with-lease origin main

# Download + merge remote changes into current branch
git pull

# Download remote changes WITHOUT merging (inspect first)
git fetch
git diff main origin/main    # see what's different
git merge origin/main        # merge when ready

# git pull = git fetch + git merge in one step
```

**`--force-with-lease` vs `--force`:**
```
git push --force              = "Overwrite GitHub no matter what"
git push --force-with-lease   = "Overwrite only if GitHub still matches
                                 what I last fetched" — protects teammates
```
Always use `--force-with-lease` when a force push is unavoidable (e.g. after
interactive rebase).

### 8.3 git clone

```bash
# Download a complete repository including all history
git clone git@github.com:Username/repo-name.git

# Clone into a specific folder name
git clone git@github.com:Username/repo-name.git my-folder

# Clone only the latest commit (no full history — faster for large repos)
git clone --depth=1 git@github.com:Username/repo-name.git
```

`git clone` is used **once** to create a fresh local copy.
`git pull` is used **repeatedly** to update an existing local copy.

### 8.4 Checking sync status

```bash
# See if local and remote are on the same commit
git log --oneline --decorate
# f918d88 (HEAD -> main, origin/main) ← both same = in sync
# f918d88 (HEAD -> main)              ← remote missing = need to push

# How many commits ahead/behind the remote
git status
# "Your branch is ahead of 'origin/main' by 2 commits"
# "Your branch is behind 'origin/main' by 1 commit"
```

---

## 9. History recovery tools

### 9.1 git reflog — the black box recorder

`git reflog` records every movement of HEAD — including after destructive
operations like `git reset --hard`. Git keeps this data for ~30 days.

```bash
# View the full HEAD movement history
git reflog

# View last 20 entries
git reflog | head -20
```

Output format:
```
a665a99 HEAD@{0}: commit: chore: add .gitignore
d5a5742 HEAD@{1}: commit: docs: add Git commands
8dc2e18 HEAD@{2}: commit: docs: add filesystem commands
78e34e0 HEAD@{3}: reset: moving to HEAD~1
```

**Recovering a lost commit after hard reset:**
```bash
# 1. Find the lost commit hash in reflog
git reflog | head -20

# 2. Restore that exact state
git reset --hard <lost-commit-hash>
```

### 9.2 git rebase -i — rewrite history

Interactive rebase lets you rewrite, reorder, squash, or delete commits.

```bash
# Rewrite the last 3 commits
git rebase -i HEAD~3

# Rewrite from the very first commit (including initial commit)
git rebase -i --root
```

Vim opens with a todo list:
```
pick a3f9c12 feat: initialise repository
pick b532acb docs: add README
pick c0cdbd3 docs: add Linux notes
```

**Available actions** (change `pick` to the action you want):

| Action | Shortcut | What it does |
|---|---|---|
| `pick` | `p` | Keep this commit as-is |
| `reword` | `r` | Keep the commit, change only the message |
| `edit` | `e` | Pause here — amend files or message |
| `squash` | `s` | Merge into the previous commit, combine messages |
| `fixup` | `f` | Merge into previous commit, discard this message |
| `drop` | `d` | Delete this commit entirely |

**Common workflow — fix multiple commit messages:**
```bash
git rebase -i --root
# Change 'pick' to 'r' on commits you want to rename
# Save and close vim (:wq)
# Vim opens once per 'r' commit — edit message, :wq each time

# After rebase — force push (hashes changed)
git push --force-with-lease origin main
```

---

## 10. Professional workflow

### 10.1 Feature branch workflow

This is the industry standard. Main branch is always clean and deployable.
All work happens on feature branches.

```bash
# 1. Always start from an updated main
git checkout main
git pull

# 2. Create a feature branch
git checkout -b feature/your-feature-name

# 3. Do your work — multiple commits on this branch
git add .
git commit -m "feat: add new feature"

# 4. Push branch to GitHub
git push -u origin feature/your-feature-name

# 5. Create a Pull Request on GitHub — get code review

# 6. After approval, merge to main and clean up
git checkout main
git merge feature/your-feature-name
git push origin main
git branch -d feature/your-feature-name          # delete locally
git push origin --delete feature/your-feature-name  # delete on GitHub
```

### 10.2 Daily workflow loop

```
1. git pull                     ← Start of day — get latest from team
2. git checkout -b feature/xyz  ← Create branch for your task
3. [edit files]
4. git diff                     ← Check what changed (unstaged)
5. git add -p <file>            ← Stage selectively
6. git diff --staged            ← Review exactly what will be committed
7. git commit -m "type: desc"   ← Commit with good message
8. [repeat 3–7 as needed]
9. git push                     ← Send to GitHub
10. Open Pull Request           ← Request code review
```

### 10.3 File management with git mv

Always use `git mv` to move or rename tracked files, never the regular `mv`:

```bash
# Correct — Git tracks the rename, file history is preserved
git mv old-name.md new-name.md
git mv filename.md folder/filename.md

# Wrong — looks like delete + new untracked file to Git
mv old-name.md new-name.md   # ← never do this for tracked files
```

---

## 11. Commit message conventions

### 11.1 The Conventional Commits format

```
<type>: <short description in imperative mood>

[optional body — explain WHY, not WHAT — max 72 chars per line]
```

**Types:**

| Type | When to use |
|---|---|
| `feat` | A new feature or file added |
| `fix` | A bug or error fixed |
| `docs` | Documentation only changes |
| `chore` | Tooling, setup, config, dependencies |
| `refactor` | Code change with no new feature or bug fix |
| `ci` | CI/CD pipeline configuration changes |
| `test` | Adding or updating tests |
| `style` | Formatting, whitespace — no logic change |

### 11.2 Rules

```bash
# ✅ Correct examples
git commit -m "feat: add Dockerfile for app container"
git commit -m "fix: correct shebang syntax in deploy script"
git commit -m "docs: add networking commands to Linux notes"
git commit -m "chore: add .gitignore for Terraform state files"

# ❌ Wrong examples
git commit -m "Fixed stuff"          # vague, past tense
git commit -m "Add Feature"          # Title Case
git commit -m "WIP"                  # not descriptive
git commit -m "Bhanu::changes"       # non-standard format
git commit -m "updated the readme file to include the new section"  # too long
```

**Rules summary:**
- Lowercase type prefix always (`feat:` not `Feat:`)
- Imperative mood — write as a command (`add` not `added` or `adding`)
- Maximum 50 characters for the subject line
- No period at the end of the subject
- Your name does not belong in the message — git config already stamps it

---

## 12. Quick reference card

### Essential commands — daily use

```bash
git status -s                    # check current state (short)
git log --oneline                # view commit history (compact)
git diff                         # see unstaged changes
git diff --staged                # see staged changes (before committing)

git add <file>                   # stage a file
git add -p <file>                # stage parts of a file interactively
git commit -m "type: msg"        # commit staged changes

git push                         # send commits to GitHub
git pull                         # get latest from GitHub
```

### Undo commands — rescue kit

```bash
git restore <file>               # discard unstaged changes (irreversible)
git restore --staged <file>      # unstage a file (safe)
git commit --amend -m "msg"      # fix last commit message (before push only)
git reset --soft HEAD~1          # undo last commit, keep staged
git reset --mixed HEAD~1         # undo last commit, keep unstaged
git reset --hard HEAD~1          # undo last commit, discard changes (danger)
git revert <hash>                # safe undo on shared branches
```

### Remote commands

```bash
git remote -v                    # show remotes
git remote add origin <url>      # add a remote
git push -u origin main          # first push (sets upstream)
git push --force-with-lease      # force push after rebase (safer)
git clone <url>                  # download a repository
```

### Recovery commands

```bash
git reflog                       # history of all HEAD movements
git rebase -i HEAD~N             # rewrite last N commits interactively
git rebase -i --root             # rewrite entire history from first commit
```

### git log variations

```bash
git log --oneline                # compact history
git log --oneline --decorate     # show branch and remote labels
git log --oneline --graph --all  # visual branch tree
git log --oneline -5             # last 5 commits only
git log --oneline --stat         # show files changed per commit
```

---

## Appendix: files in this repository

```
devops-learning-journey/
│
├── README.md                          ← Project overview and roadmap
├── .gitignore                         ← Patterns Git ignores
│
├── phase-01-git/
│   ├── README.md                      ← Phase summary and commands table
│   └── notes/
│       ├── linux-notes.md             ← Linux commands reference
│       ├── bash-notes.md              ← Bash scripting notes
│       └── python-notes.md            ← Python DevOps library notes
│
├── phase-02-docker/
│   └── README.md                      ← Placeholder — Phase 02
│
├── phase-03-cicd/
│   └── README.md                      ← Placeholder — Phase 03
│
└── phase-04-aws/
    └── README.md                      ← Placeholder — Phase 04
```

---

*Document version: Week 1 complete — 21 days total, Days 1–7 covered*
*Next: Week 2 — Branching, merging, pull requests, and team collaboration*
