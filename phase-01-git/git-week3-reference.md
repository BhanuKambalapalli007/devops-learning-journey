# Git complete reference — Week 3
**Bhanu Prakash · DevOps Learning Journey · Phase 01**

---

## Table of contents

1. [GitFlow workflow](#1-gitflow-workflow)
2. [Trunk-based development](#2-trunk-based-development)
3. [git stash](#3-git-stash)
4. [git cherry-pick](#4-git-cherry-pick)
5. [Git tags and semantic versioning](#5-git-tags-and-semantic-versioning)
6. [Git hooks](#6-git-hooks)
7. [GitHub Actions — CI/CD pipelines](#7-github-actions--cicd-pipelines)
8. [git bisect — binary search for bugs](#8-git-bisect--binary-search-for-bugs)
9. [git reflog — recover anything](#9-git-reflog--recover-anything)
10. [Choosing the right workflow](#10-choosing-the-right-workflow)
11. [Quick reference card](#11-quick-reference-card)

---

## 1. GitFlow workflow

### Overview

GitFlow is a branching model by Vincent Driessen (2010) that defines
strict rules for how branches are created, used, and merged. Built for
teams shipping versioned software on a release schedule.

### The 5 branch types

```
feature/ ─────────────────────○
             ↗                   ↘
develop:  ○──────────────────────○──────────────────○
                              ↓    ↑           ↑↑
release/:               ○──────○    hotfix: ○──○
                                 ↘            ↓
main:     ○──────────────────────○────────────○
                                v1.0         v1.1
```

| Branch | Branches from | Merges into | Purpose |
|---|---|---|---|
| `main` | — | — | Production only. Tagged releases. |
| `develop` | `main` | — | Integration. Next release lives here. |
| `feature/*` | `develop` | `develop` | One branch per feature. |
| `release/*` | `develop` | `main` + `develop` | Release prep. Bug fixes only. |
| `hotfix/*` | `main` | `main` + `develop` | Urgent production patches. |

### Critical rules

```
feature/* → NEVER touches main directly
release/* → ONLY bug fixes, no new features
hotfix/*  → ONLY branch that starts from main
Every main merge MUST get a version tag
develop and main MUST always be kept in sync after merges
```

### Core GitFlow commands

```bash
# Setup
git checkout -b develop main
git push -u origin develop

# Feature lifecycle
git checkout -b feature/xyz develop
# ... commits ...
git switch develop
git merge --no-ff feature/xyz -m "feat: merge xyz"
git branch -d feature/xyz

# Release lifecycle
git checkout -b release/v1.0 develop
# ... bug fixes only ...
git switch main
git merge --no-ff release/v1.0 -m "release: merge v1.0"
git tag -a v1.0.0 -m "Release v1.0.0"
git switch develop
git merge --no-ff release/v1.0 -m "release: sync v1.0 to develop"
git branch -d release/v1.0
git push origin main develop v1.0.0

# Hotfix lifecycle
git checkout -b hotfix/fix-critical main
# ... fix ...
git switch main
git merge --no-ff hotfix/fix-critical -m "hotfix: critical fix"
git tag -a v1.0.1 -m "Release v1.0.1"
git switch develop
git merge --no-ff hotfix/fix-critical -m "hotfix: sync fix to develop"
git branch -d hotfix/fix-critical
git push origin main develop v1.0.1
```

### When to use GitFlow

```
✅ Use for:                          ❌ Avoid for:
────────────────────────────────     ────────────────────────────────
Versioned software (v1, v2, v3)      Web apps deploying many times/day
Mobile apps (App Store releases)     Small teams (1-3 engineers)
Libraries (npm / pip packages)       Continuous deployment workflows
Enterprise with multiple versions    Teams using trunk-based dev
```

---

## 2. Trunk-based development

### Core concept

Everyone integrates to `main` (the trunk) frequently — multiple times
per day. Branches are short-lived, measured in hours not days. Every
commit to main triggers CI and is potentially deployable.

```
GITFLOW:   feature─────days─────►develop►release►main
TRUNK:     main●─●─●─●─●─●─●─●─●─●─●─● (branch exists for hours)
```

### Rules of trunk-based development

```
1. main is always green — CI must pass at all times
2. Branches must be merged within 1-2 days maximum
3. Every commit to main triggers the CI pipeline
4. Broken CI is the highest priority to fix — blocks everyone
5. Never push untested code to main
6. If you need more than 2 days, use a feature flag
```

### Feature flags — the enabler

```python
# Incomplete feature ships to production safely behind a flag
def process_payment():
    if feature_flag("new-payment-system"):
        new_payment_logic()    # only when flag is ON
    else:
        old_payment_logic()    # production traffic goes here

# Flag is turned ON only when fully ready and tested
# Can be turned OFF instantly if issues arise
```

### Trunk-based daily workflow

```bash
git switch main && git pull       # always start from latest
git checkout -b docs/small-fix    # branch if needed
# ... focused work, complete in hours ...
git add . && git commit -m "type: description"
git fetch origin && git rebase origin/main    # stay current
git switch main
git merge --no-ff docs/small-fix -m "type: merge description"
git push origin main                          # CI triggers
git branch -d docs/small-fix                  # immediate cleanup
```

### When to use trunk-based

```
✅ Use for:                          ❌ Avoid for:
────────────────────────────────     ──────────────────────────────
SaaS web apps                        Products with strict versioning
Microservices                        Teams without CI/CD pipelines
Teams with mature CI/CD              Teams without feature flag system
Continuous deployment goals          Junior teams needing more structure
```

---

## 3. git stash

### What it does

Saves uncommitted changes temporarily so you can switch context
without committing incomplete work.

```
Working directory (unfinished)
    ↓  git stash push -m "message"
Stash stack (saved safely)
    ↓  git stash pop
Working directory (restored)
```

### All stash commands

```bash
# Save changes with a descriptive message
git stash push -m "wip: working on monitoring config"

# Save including untracked (new) files
git stash push --include-untracked -m "wip: includes new files"

# List all stashes
git stash list
# stash@{0}: On main: wip: working on monitoring config
# stash@{1}: On feature/xyz: wip: half-done login

# See what is in the most recent stash
git stash show
git stash show -p                   # full diff

# Restore most recent stash and remove from list
git stash pop

# Restore but keep in the stash list
git stash apply

# Restore a specific stash by index
git stash apply stash@{2}

# Delete a specific stash
git stash drop stash@{0}

# Delete ALL stashes
git stash clear
```

### Stash best practices

```
✅ Always add a descriptive message — git stash push -m "..."
✅ Don't let stashes accumulate — pop or drop them promptly
✅ Use --include-untracked when you've created new files
⚠️  Stashes are LOCAL only — not pushed to GitHub
⚠️  Stash conflicts can occur if branch changed significantly
```

---

## 4. git cherry-pick

### What it does

Applies a specific commit from another branch onto your current branch
without merging the entire branch.

```
develop:  ●──●──●──SECURITY-FIX──●──●──●
main:     ●──●──●──────────────────────────
                              ↑
              git cherry-pick SECURITY-FIX hash
main:     ●──●──●──SECURITY-FIX' (new hash, same changes)
```

### When to use cherry-pick

```
✅ Applying a critical security fix to an older release
✅ Moving a single commit accidentally made on wrong branch
✅ Picking specific features from an abandoned branch
✅ Backporting a bug fix to a maintenance branch

❌ Do NOT use as a substitute for proper merging
❌ Avoid picking merge commits (complex, use -m 1 if needed)
```

### All cherry-pick commands

```bash
# Apply one specific commit to current branch
git cherry-pick <commit-hash>

# Apply without auto-committing — review staged changes first
git cherry-pick --no-commit <commit-hash>

# Apply a range of commits (A exclusive, B inclusive)
git cherry-pick a3f9c12..b532acb

# Cherry-pick a merge commit (specify which parent to use)
git cherry-pick -m 1 <merge-commit-hash>

# Abort if a conflict is too complex
git cherry-pick --abort

# Continue after resolving conflict
git cherry-pick --continue
```

---

## 5. Git tags and semantic versioning

### Tag types

```bash
# Annotated (recommended for releases)
# Stores: tag name, tagger, date, GPG-signable message
git tag -a v1.0.0 -m "Release v1.0.0 — initial release"

# Lightweight (just a pointer — no metadata)
# Use for temporary local markers only
git tag v1.0.0-draft
```

### Semantic versioning — the industry standard

```
v  MAJOR . MINOR . PATCH
   ─────   ─────   ─────
     │       │       └── Bug fixes, no new features (hotfix)
     │       └────────── New features, backward compatible
     └────────────────── Breaking changes, incompatible API

Examples:
v1.0.0  →  initial release
v1.0.1  →  patch (hotfix)
v1.1.0  →  minor (new feature)
v2.0.0  →  major (breaking change)
```

### All tag commands

```bash
git tag -a v1.0.0 -m "message"       # create at HEAD
git tag -a v0.1.0 a3f9c12 -m "msg"   # tag a past commit
git tag                               # list all tags
git tag -l "v1.*"                     # list matching pattern
git show v1.0.0                       # view tag details
git push origin v1.0.0               # push one tag
git push origin --tags               # push all tags
git tag -d v1.0.0-draft              # delete locally
git push origin --delete v1.0.0-draft # delete from remote
git checkout v1.0.0                   # view code at that tag
git checkout -b fix/from-v1 v1.0.0   # branch from a tag
```

---

## 6. Git hooks

### What are hooks?

Scripts in `.git/hooks/` that run automatically at specific Git events.
If a hook exits with non-zero, the Git operation is blocked.

```bash
ls .git/hooks/    # shows available hooks (*.sample = disabled)
# Remove .sample extension to activate a hook
chmod +x .git/hooks/pre-commit    # must be executable
```

### Most useful hooks for DevOps

| Hook | Triggers when | Common use |
|---|---|---|
| `pre-commit` | Before commit is created | Lint, format, secret scan |
| `commit-msg` | After message is written | Enforce message format |
| `pre-push` | Before git push | Run full test suite |
| `post-merge` | After merge completes | Install dependencies |
| `pre-rebase` | Before rebase starts | Warn about public branches |

### Example: commit-msg hook

```bash
#!/bin/bash
# Enforce Conventional Commits format
COMMIT_MSG=$(cat "$1")
VALID_TYPES="feat|fix|docs|chore|refactor|ci|test|style|perf"

if ! echo "$COMMIT_MSG" | grep -qE "^($VALID_TYPES)(\(.+\))?: .{1,72}$"; then
  echo "❌ Invalid format. Required: type: description"
  exit 1
fi
exit 0
```

### Example: pre-commit hook

```bash
#!/bin/bash
# Block TODO/FIXME in staged files
TODOS=$(git diff --cached --name-only | xargs grep -l "TODO\|FIXME" 2>/dev/null)
if [ -n "$TODOS" ]; then
  echo "❌ TODO/FIXME found in: $TODOS"
  exit 1
fi
exit 0
```

### The team problem — Husky solves it

```bash
# .git/hooks/ is never committed — teammates don't get your hooks
# Husky stores hooks in /.husky/ which IS committed

npm install husky --save-dev
npx husky init
# Creates .husky/pre-commit — committed to the repo
# All teammates get the hooks automatically on npm install
```

---

## 7. GitHub Actions — CI/CD pipelines

### Core concepts

```
WORKFLOW  → the whole automation file (.github/workflows/ci.yml)
TRIGGER   → when it runs (push, pull_request, schedule, manual)
JOB       → group of steps running on one machine
RUNNER    → the machine (ubuntu-latest, windows-latest, macos-latest)
STEP      → one task: either uses: (marketplace action) or run: (shell)
```

### Basic workflow structure

```yaml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5        # check out code
      - run: echo "Run your checks here"  # shell command
```

### Context variables

| Variable | Value |
|---|---|
| `github.actor` | Username who triggered the run |
| `github.ref_name` | Branch or tag name |
| `github.sha` | Full commit hash |
| `github.event_name` | push, pull_request, etc. |
| `github.repository` | owner/repo-name |

### Trigger types

```yaml
on:
  push:                          # on every push
    branches: [ main ]
    tags: [ 'v*' ]               # also trigger on version tags

  pull_request:                  # on PR open/update
    branches: [ main ]

  schedule:                      # cron schedule
    - cron: '0 9 * * 1'         # every Monday at 9am UTC

  workflow_dispatch:             # manual trigger button on GitHub
```

### Multi-job pipeline

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: echo "Run linter"

  test:
    runs-on: ubuntu-latest
    needs: lint          # only runs if lint passes
    steps:
      - uses: actions/checkout@v5
      - run: echo "Run tests"

  deploy:
    runs-on: ubuntu-latest
    needs: test          # only runs if test passes
    if: github.ref == 'refs/heads/main'   # only on main
    steps:
      - uses: actions/checkout@v5
      - run: echo "Deploy to production"
```

### Free tier limits

```
Public repos:   Unlimited minutes (free)
Private repos:  2,000 minutes/month free
Runner specs:   2-core CPU, 7GB RAM, 14GB SSD
Retention:      Logs kept 90 days, artifacts 90 days
```

### Adding a status badge to README

```markdown
![CI](https://github.com/USERNAME/REPO/actions/workflows/ci.yml/badge.svg)
```

---

## 8. git bisect — binary search for bugs

### What it does

Performs a binary search through commit history to find exactly which
commit introduced a bug. Far faster than manual investigation.

```
47 commits to check = maximum 6 bisect steps (log₂47 ≈ 6)
```

### How binary search works in bisect

```
All commits: [C1  C2  C3  C4  C5  C6  C7  C8  C9  C10]
              ↑good                              ↑bad

Step 1: Check midpoint C5 → bad
        [C1  C2  C3  C4  C5]
                        ↑bad

Step 2: Check C3 → good
        [C3  C4  C5]
                 ↑bad

Step 3: Check C4 → bad → FOUND
        C4 is the first bad commit
```

### All bisect commands

```bash
git bisect start                  # begin the search
git bisect bad                    # current commit is broken
git bisect good v1.0.0            # this commit definitely worked
# ... Git checks out midpoint, you test ...
git bisect bad                    # bug exists here
git bisect good                   # bug does not exist here
# ... repeat until Git announces the bad commit ...
git bisect reset                  # return HEAD to original position

# Automated bisect with a test script
git bisect start
git bisect bad HEAD
git bisect good <good-hash>
git bisect run bash your-test.sh  # script must exit 0=good, 1=bad
git bisect reset

# View the bisect log
git bisect log

# Skip a commit (e.g. broken for unrelated reasons)
git bisect skip
```

---

## 9. git reflog — recover anything

### What it does

Records every movement of HEAD — commits, resets, rebases, checkouts,
merges. Git keeps this data for 30 days. Nothing is truly lost.

```bash
git reflog            # full HEAD history
git reflog | head -20 # last 20 entries
```

Output:
```
af4e25b HEAD@{0}: commit: docs: add Kubernetes overview
3ee58cd HEAD@{1}: commit: ci: fix TODO false positive
4c76e67 HEAD@{2}: commit: hotfix: merge README fix
```

### Recover from git reset --hard

```bash
# You accidentally destroyed work with hard reset
git reset --hard HEAD~3

# Find the lost commits in reflog
git reflog | head -10
# HEAD@{1}: commit: your last important commit

# Restore to before the reset
git reset --hard HEAD@{1}
# Or use the specific hash: git reset --hard af4e25b
```

### Recover a deleted branch

```bash
# Branch deleted without merging
git branch -D feature/important-work

# Find the last commit on that branch in reflog
git reflog | grep "important-work"
# HEAD@{3}: commit: feat: important work (was on feature/important-work)

# Recreate the branch from that commit
git checkout -b feature/important-work HEAD@{3}
```

### Recover from a bad rebase

```bash
# Rebase went wrong — history is a mess
git rebase --abort    # try abort first

# If rebase already finished and looks wrong:
git reflog | head -10
# Find entry just before rebase: "HEAD@{N}: checkout: moving from..."
git reset --hard HEAD@{N}    # restore to pre-rebase state
```

### Time-based reflog recovery

```bash
git reset --hard HEAD@{1.hour.ago}     # one hour ago
git reset --hard HEAD@{yesterday}      # yesterday
git reset --hard HEAD@{2.weeks.ago}    # two weeks ago
```

---

## 10. Choosing the right workflow

### Decision framework

```
Are you shipping versioned software (v1, v2)?
├── YES → GitFlow
└── NO  ↓

Do you deploy multiple times per day?
├── YES → Trunk-based development
└── NO  ↓

Is your team larger than 10 engineers?
├── YES → GitFlow or scaled trunk-based
└── NO  → Trunk-based (simpler)
```

### Side-by-side comparison

| Factor | GitFlow | Trunk-based |
|---|---|---|
| Branch lifespan | Days to weeks | Hours to 1-2 days |
| Release frequency | Scheduled (weekly/monthly) | Continuous |
| Merge commits | Many | Minimal |
| CI requirement | Recommended | Mandatory |
| Feature flags | Optional | Required |
| Best for | Versioned products | Web services |
| Examples | Mobile apps, libraries | Netflix, Google, Spotify |

---

## 11. Quick reference card

### GitFlow

```bash
git checkout -b feature/xyz develop    # start feature
git merge --no-ff feature/xyz          # merge feature to develop
git checkout -b release/v1.0 develop   # start release
git tag -a v1.0.0 -m "msg"            # tag after release merge
git checkout -b hotfix/fix main        # start hotfix from main
```

### Trunk-based

```bash
git switch main && git pull            # always start current
git checkout -b fix/small-thing        # short-lived branch
git rebase origin/main                 # keep current before merge
git push origin main                   # push and trigger CI
```

### Stash

```bash
git stash push -m "wip: description"  # save with message
git stash list                         # see all stashes
git stash pop                          # restore and remove
git stash apply stash@{2}             # restore specific stash
```

### Cherry-pick

```bash
git cherry-pick <hash>                 # apply one commit
git cherry-pick --no-commit <hash>     # apply without committing
git cherry-pick a3f..b5c              # apply a range
git cherry-pick --abort                # cancel on conflict
```

### Tags

```bash
git tag -a v1.0.0 -m "Release"        # create annotated tag
git push origin v1.0.0                # push tag
git push origin --tags                # push all tags
git tag -d v1.0.0                     # delete locally
```

### Git bisect

```bash
git bisect start                       # start search
git bisect bad                         # mark current as broken
git bisect good <hash>                 # mark known good commit
git bisect run bash test.sh            # automated search
git bisect reset                       # end and restore HEAD
```

### Git reflog

```bash
git reflog                             # all HEAD movements
git reset --hard HEAD@{1}             # restore N steps ago
git reset --hard HEAD@{1.hour.ago}    # restore by time
git checkout -b rescue HEAD@{3}       # create branch from reflog
```

### GitHub Actions

```bash
# File location
.github/workflows/ci.yml

# Key YAML fields
on: push / pull_request / schedule / workflow_dispatch
runs-on: ubuntu-latest
uses: actions/checkout@v5
run: bash my-script.sh
needs: previous-job-name          # job dependency
if: github.ref == 'refs/heads/main' # conditional step
```

---

## Appendix: Week 3 skills summary

```
Day 15  ✅  GitFlow — 5 branch types, full lifecycle with tags
Day 16  ✅  Trunk-based development — short branches, feature flags
Day 17  ✅  git stash, cherry-pick, tags — prior org experience
Day 18  ✅  Git hooks — prior org experience
Day 19  ✅  GitHub Actions — first CI pipeline, live and green
Day 20  ✅  git bisect — binary search debugging
Day 20  ✅  git reflog — recover from any disaster
Day 21  ✅  Final portfolio project
```

**Phase 01 Git & Version Control — COMPLETE.**

---

*Document version: Week 3 complete — Days 15–21 covered*
*Previous: [git-week2-reference.md](git-week2-reference.md)*
*Next: Phase 02 — Docker & Containerisation*
