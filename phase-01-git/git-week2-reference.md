# Git complete reference — Week 2
**Bhanu Prakash · DevOps Learning Journey · Phase 01**

---

## Table of contents

1. [Branches — core concepts](#1-branches--core-concepts)
2. [The HEAD pointer](#2-the-head-pointer)
3. [Branch commands](#3-branch-commands)
4. [Merging](#4-merging)
5. [Merge conflicts](#5-merge-conflicts)
6. [git rebase](#6-git-rebase)
7. [Remote branch operations](#7-remote-branch-operations)
8. [GitHub Pull Requests](#8-github-pull-requests)
9. [GitFlow workflow](#9-gitflow-workflow)
10. [Professional team workflow](#10-professional-team-workflow)
11. [Quick reference card](#11-quick-reference-card)

---

## 1. Branches — core concepts

### What is a branch?

A branch is a **lightweight movable pointer to a commit**. It is not a
copy of your code. Creating a branch costs almost nothing — it is a
41-byte file containing a single commit hash stored in `.git/refs/heads/`.

```bash
# Proof — a branch is just a file containing a hash
cat .git/refs/heads/main
# 58f02a7c1a2b3d4e5f6a7b8c9d0e1f2a3b4c5d6
```

When you make a new commit on a branch, the pointer moves forward
automatically to point at the new commit.

### Why branches matter in DevOps

Every CI/CD pipeline is built around branches:

```
Developer pushes to feature branch  →  pipeline runs tests
PR merged to main                   →  deployment triggered
Tag pushed to main                  →  release published
```

`main` is always clean and deployable. All work happens on branches.

### Branch naming conventions

| Prefix | Purpose | Example |
|---|---|---|
| `feature/` | New functionality | `feature/add-docker-setup` |
| `fix/` | Bug fix | `fix/correct-login-timeout` |
| `hotfix/` | Urgent production fix | `hotfix/patch-security-vuln` |
| `release/` | Release preparation | `release/v2.1.0` |
| `docs/` | Documentation only | `docs/update-api-guide` |
| `chore/` | Maintenance / tooling | `chore/upgrade-dependencies` |

---

## 2. The HEAD pointer

`HEAD` is Git's "you are here" marker. It always points to your current
location in the repository.

### Normal state — HEAD points to a branch

```
HEAD → feature/my-branch → latest commit on that branch
```

When you make a new commit, the branch pointer moves forward and HEAD
stays pointing to the branch (so HEAD also moves forward automatically).

### After switching branches

```bash
git switch main
# HEAD → main → latest commit on main
```

HEAD now points to `main`. Your working directory instantly updates to
match `main`'s state.

### Detached HEAD state

Occurs when HEAD points directly to a commit instead of a branch:

```bash
git checkout a3f9c12    # checking out a specific commit hash
# HEAD → a3f9c12 (no branch)
```

Any commits made in this state are at risk of being lost. To save work
done in detached HEAD, create a branch immediately:

```bash
git checkout -b rescue-branch    # saves the detached commits
```

### Relative references

```bash
HEAD~1    # one commit before HEAD
HEAD~2    # two commits before HEAD
HEAD~3    # three commits before HEAD
HEAD^     # same as HEAD~1 (parent of HEAD)
HEAD^2    # second parent of a merge commit
```

---

## 3. Branch commands

### Creating branches

```bash
# Create a branch without switching (stays on current branch)
git branch feature/xyz

# Create AND switch in one command — classic syntax
git checkout -b feature/xyz

# Create AND switch — modern syntax (Git 2.23+)
git switch -c feature/xyz

# Create a branch from a specific commit or other branch
git checkout -b feature/xyz main       # from main
git checkout -b feature/xyz a3f9c12    # from a specific commit
```

### Switching branches

```bash
git checkout main          # classic syntax
git switch main            # modern syntax (preferred)
git switch -               # switch back to the previous branch
```

### Listing branches

```bash
git branch              # list local branches (* = current)
git branch -v           # show last commit on each branch
git branch -a           # list local + remote branches
git branch -r           # list remote branches only
git branch --merged     # branches already merged into current
git branch --no-merged  # branches NOT yet merged
```

### Deleting branches

```bash
git branch -d feature/xyz     # delete if merged — safe
git branch -D feature/xyz     # force delete even if not merged

# Delete a remote branch
git push origin --delete feature/xyz

# Clean up stale remote references locally
git remote prune origin
git fetch --prune             # fetch and prune in one step
```

### Renaming branches

```bash
git branch -m new-name           # rename current branch
git branch -m old-name new-name  # rename any branch
```

---

## 4. Merging

### Fast-forward merge

Occurs when the target branch has **not moved** since the feature branch
was created. Git simply moves the target pointer forward — no new commit.

```
Before:                         After (fast-forward):
main  → C3                      main  → C4 (= feature)
                C4 ← feature
```

```bash
git switch main
git merge feature/xyz
# Output: "Fast-forward"
# No merge commit — history stays linear
```

### 3-way merge

Occurs when **both** branches have new commits since the split point.
Git uses 3 commits (common ancestor + both tips) to create a merge commit.

```
Before:                         After (3-way):
main  → C3 → C5                 main  → M (merge commit)
         \                               / \
          C4 ← feature               C5   C4
```

```bash
git switch main
git merge feature/xyz
# Vim opens for merge commit message
# Output: "Merge made by the 'ort' strategy."
```

### The `--no-ff` flag

Forces a merge commit even when fast-forward is possible. Preserves
the record of when a feature was merged.

```bash
git merge --no-ff feature/xyz -m "feat: merge feature xyz"
```

GitHub's **"Create a merge commit"** button uses `--no-ff`.

### Merge strategies comparison

| Strategy | Command | History shape | Merge commit |
|---|---|---|---|
| Fast-forward | `git merge` (auto) | Linear | No |
| 3-way | `git merge` (auto) | Diverge + converge | Yes |
| No fast-forward | `git merge --no-ff` | Diverge + converge | Always |
| Squash | `git merge --squash` | Linear | No (manual commit) |

### Aborting a merge

```bash
git merge --abort
# Returns everything to pre-merge state
# Only works while merge is paused (MERGE_HEAD exists)
```

---

## 5. Merge conflicts

### What causes a conflict

A conflict occurs when **two branches edit the same line** of the same
file differently. Git cannot decide which version is correct and stops
to ask you.

```
Branch A changes line 7:   status = "complete"
Branch B changes line 7:   status = "in progress"
← Git cannot choose — human must decide
```

### Conflict markers

Git writes conflict markers directly into the affected file:

```
<<<<<<< HEAD
your current branch version of the line
=======
incoming branch version of the line
>>>>>>> feature/branch-name
```

| Marker | Meaning |
|---|---|
| `<<<<<<< HEAD` | Start of YOUR current branch version |
| `=======` | Divider between the two versions |
| `>>>>>>> branch` | End of the INCOMING branch version |

### Resolving a conflict

```bash
# 1. See which files have conflicts
git status -s
# UU filename.md    ← UU = unmerged on both sides

# 2. Open the conflicted file and edit it
vim filename.md
# Delete the three marker lines
# Edit to keep the correct final version
# Save :wq

# 3. Verify all markers are removed
grep -n "<<<<<<\|=======\|>>>>>>" filename.md
# (empty output = all markers removed)

# 4. Stage the resolved file
git add filename.md

# 5. Complete the merge
git commit -m "fix: resolve merge conflict in filename.md"
```

### Conflict prevention habits

```bash
# Pull before you start — always
git switch main && git pull
git checkout -b feature/new-work

# Keep branches short-lived (days not weeks)
# Communicate: if someone is editing config.yaml, stay out of it
# Small focused commits = fewer conflict surfaces
```

### Using git status during a conflict

```bash
git status
# Both modified:   filename.md     ← conflict here
# Changes to be committed:         ← already resolved files
```

---

## 6. git rebase

### What rebase does

Rebase **replays your commits on top of another branch**, rewriting
their hashes to have a new parent. The result is a perfectly linear
history with no merge commits.

```
Before rebase:                  After rebase:
main:    C1─C2─C3─C5            main:    C1─C2─C3─C5─C4'─C6'
                 \                                        (new hashes)
feature:          C4─C6
```

### The golden rule of rebase

> **Never rebase commits that have already been pushed to a shared**
> **remote branch.**

Rebase rewrites commit hashes. Teammates who have your old commits
will have a diverged history. Force-pushing to fix it is disruptive.
Rebase is only safe on local-only commits.

```
✅ Safe:    commits that only exist on YOUR local machine
❌ Unsafe:  commits already pushed to origin/main or shared branches
```

### Regular rebase — keep feature branch current

```bash
# Update your feature branch with latest main
git switch feature/xyz
git rebase main

# Now feature is replayed on top of latest main
# A subsequent merge will be a clean fast-forward
git switch main
git merge feature/xyz    # Fast-forward — linear history
```

### Interactive rebase — rewrite history

```bash
# Rewrite last N commits
git rebase -i HEAD~N

# Rewrite from the very first commit
git rebase -i --root
```

Vim opens with a todo list of commits oldest-first. Change `pick` to
an action keyword:

| Action | Short | What it does |
|---|---|---|
| `pick` | `p` | Keep commit as-is |
| `reword` | `r` | Keep commit, edit the message |
| `edit` | `e` | Pause to amend files or message |
| `squash` | `s` | Merge into previous commit, combine messages |
| `fixup` | `f` | Merge into previous commit, discard this message |
| `drop` | `d` | Delete this commit entirely |

### Squash — combine multiple commits into one

```bash
git rebase -i HEAD~3
# Change:
# pick  first commit
# s     second commit    ← squash into first
# s     third commit     ← squash into first

# Vim opens again — write the combined message
# Save :wq → 3 commits become 1
```

### Force push after rebase

Rebase rewrites hashes, so regular `git push` is rejected after rebasing:

```bash
# ✅ Safer — refuses if remote changed since last fetch
git push --force-with-lease origin branch-name

# ❌ Dangerous — overwrites remote regardless
git push --force origin branch-name
```

Always use `--force-with-lease`. It protects teammates' work in case
someone pushed while you were rebasing.

### Merge vs Rebase — decision guide

```
Use MERGE when:                     Use REBASE when:
──────────────────────────────      ────────────────────────────────
Merging a completed feature PR      Updating a local feature branch
Working on a shared branch          Local commits not yet pushed
Preserving exact history            Cleaning history before a PR
After a code review approval        Squashing WIP commits
```

---

## 7. Remote branch operations

### Tracking branches

A tracking branch is a local branch linked to a remote branch. After
`git push -u origin branch`, Git knows where to push/pull automatically.

```bash
# See tracking relationships for all branches
git branch -vv
# * main     58f02a7 [origin/main] docs: add reference
#   feature  a3f9c12 [origin/feature] work in progress
```

### Push, fetch, pull for branches

```bash
# Push a branch and set tracking (first time)
git push -u origin feature/xyz

# Push after tracking is set
git push

# Download all remote branch updates (no merge)
git fetch

# Download and merge current branch's upstream
git pull

# Download and rebase instead of merge
git pull --rebase

# See difference between local and remote before pulling
git fetch
git diff main origin/main
```

### Checking out a remote branch

```bash
# Create a local branch tracking a remote branch
git checkout -b feature/xyz origin/feature/xyz

# Modern syntax
git switch feature/xyz    # Git auto-tracks if remote exists
```

### Sync status

```bash
git log --oneline --decorate
# 58f02a7 (HEAD -> main, origin/main) ← same = in sync
# 58f02a7 (HEAD -> main)              ← origin missing = push needed
```

---

## 8. GitHub Pull Requests

### What a PR is

A Pull Request is a GitHub feature (not Git) that wraps a branch push
into a formal review request. It shows the diff, hosts comments, tracks
approvals, and provides the merge button.

```
Local branch → git push → GitHub branch → Open PR → Review → Merge
```

### The PR lifecycle

```
1. git checkout -b feature/xyz          Create branch
2. [make commits]
3. git push -u origin feature/xyz       Push branch
4. Open PR on GitHub                    Request review
   └── Title, description, reviewers
5. Code review                          Feedback loop
   └── Comments on specific lines
   └── Approve / Request changes
6. Address feedback → git push          Update PR
7. PR approved → Merge                  Ship it
8. git switch main && git pull          Sync locally
9. git branch -d feature/xyz           Clean up
```

### PR description template

```markdown
## What this PR does
[Brief description of the change]

## Why
[Motivation — link to issue if applicable]

## Changes
- file1.md — [what changed]
- file2.md — [what changed]

## Checklist
- [ ] Tests pass
- [ ] No sensitive information included
- [ ] Documentation updated
```

### GitHub merge strategies

| Button | What it does | Best for |
|---|---|---|
| **Create a merge commit** | 3-way merge, preserves branch | Default — shows feature history |
| **Squash and merge** | All PR commits → one commit | Clean main history |
| **Rebase and merge** | Replays commits linearly | Linear history purists |

Most modern teams use **Squash and merge** so `main` has one commit
per feature regardless of how many WIP commits were made on the branch.

### Forks

A fork is your own copy of someone else's repository on GitHub.
Used for open source contributions.

```bash
# After forking on GitHub, clone your fork
git clone git@github.com:YourUsername/repo-name.git

# Add the original as 'upstream'
git remote add upstream https://github.com/OriginalOwner/repo-name.git

# Keep your fork in sync with the original
git fetch upstream
git merge upstream/main

# Make changes, push to your fork, open PR against original
```

### Branch protection rules

```
GitHub → Repo → Settings → Branches → Add rule
─────────────────────────────────────────────────
Branch name pattern:  main
✅ Require a pull request before merging
✅ Require approvals: 1
✅ Require status checks to pass (CI must pass)
✅ Include administrators
```

With these rules: nobody — not even the repo owner — can push directly
to main. Everything goes through a PR.

---

## 9. GitFlow workflow

### Overview

GitFlow defines 5 branch types with strict rules for how they interact.
Created by Vincent Driessen in 2010 for versioned software releases.

```
feature/ ──────────────────○
            ↗                ↘
develop:  ○────────────────────○──────────────────○
                            ↓    ↑           ↑↑
release/:               ○──────○    hotfix: ○─○
                                ↘            ↓
main:     ○────────────────────────○──────────○
                                  v1.0       v1.1
```

### The 5 branch types

| Branch | Branches from | Merges into | Purpose |
|---|---|---|---|
| `main` | — | — | Production code only. Tagged releases. |
| `develop` | `main` | — | Integration branch. Next release lives here. |
| `feature/*` | `develop` | `develop` | One branch per feature. |
| `release/*` | `develop` | `main` + `develop` | Release prep. Bug fixes only. |
| `hotfix/*` | `main` | `main` + `develop` | Urgent production patches. |

### GitFlow commands

```bash
# Setup
git checkout -b develop main
git push -u origin develop

# Start a feature
git checkout -b feature/xyz develop
# ... work, commit ...
git switch develop
git merge --no-ff feature/xyz -m "feat: merge xyz"
git branch -d feature/xyz

# Start a release
git checkout -b release/v1.0 develop
# ... bug fixes only, update version numbers ...
git switch main
git merge --no-ff release/v1.0 -m "release: v1.0"
git tag -a v1.0.0 -m "Release v1.0.0"
git switch develop
git merge --no-ff release/v1.0 -m "release: merge v1.0 back to develop"
git branch -d release/v1.0

# Start a hotfix
git checkout -b hotfix/critical-fix main
# ... fix ...
git switch main
git merge --no-ff hotfix/critical-fix -m "hotfix: critical fix"
git tag -a v1.0.1 -m "Release v1.0.1"
git switch develop
git merge --no-ff hotfix/critical-fix -m "hotfix: merge to develop"
git branch -d hotfix/critical-fix

# Push tags
git push origin --tags
# or push a specific tag
git push origin v1.0.0
```

### Git tags

```bash
# Create an annotated tag (use for releases — has message + author)
git tag -a v1.0.0 -m "Release v1.0.0 — initial release"

# Create a lightweight tag (just a pointer, no metadata)
git tag v1.0.0

# List all tags
git tag

# View tag details
git show v1.0.0

# Push all tags to remote
git push origin --tags

# Push a specific tag
git push origin v1.0.0

# Delete a tag locally
git tag -d v1.0.0

# Delete a tag on remote
git push origin --delete v1.0.0
```

### When to use GitFlow

```
✅ Use GitFlow for:                 ❌ Avoid GitFlow for:
─────────────────────────────       ─────────────────────────────────
Versioned software (v1, v2)         Web apps deploying daily
Mobile apps (App Store cycles)      Small teams (1-3 people)
Enterprise products                 Continuous deployment pipelines
Libraries / npm / pip packages      Startups moving fast
Multiple versions maintained        Teams using trunk-based dev
```

---

## 10. Professional team workflow

### Feature branch workflow — daily loop

```bash
# Start of day
git switch main
git pull                              # get latest

# Create feature branch
git checkout -b feature/your-task

# Work cycle (repeat as needed)
vim your-file.md                      # edit
git diff                              # check changes
git add -p your-file.md               # stage selectively
git diff --staged                     # review before committing
git commit -m "type: description"     # commit

# Keep branch current (if main moved during your work)
git fetch origin
git rebase origin/main               # replay your commits on top

# Ship it
git push -u origin feature/your-task  # push branch
# Open PR on GitHub
# Address review feedback, push more commits
# PR approved → merge on GitHub

# Sync and clean up
git switch main
git pull
git branch -d feature/your-task
git remote prune origin
```

### Code review checklist

When reviewing a PR (as reviewer):

```
✅ Does the code do what the PR description says?
✅ Are commit messages clear and properly formatted?
✅ Are there any obvious bugs or logic errors?
✅ Is anything hardcoded that should be configurable?
✅ Are secrets or credentials accidentally included?
✅ Does the documentation match the changes?
✅ Are filenames and folder structure consistent?
```

### Reading `git log --graph`

```bash
git log --oneline --graph --decorate --all
```

Output symbols:

```
*     = a commit
|     = branch line continuing
/     = branch merging in from the right
\     = branch diverging to the right
|\    = branch split (divergence point)
|/    = branch merged (convergence point)
*--*  = commits connected on same branch
```

Example output:
```
*   xxxxxxx (HEAD -> main) Merge branch 'feature/xyz'
|\
| * xxxxxxx feat: add new feature
* | xxxxxxx docs: update README
|/
* xxxxxxx chore: add .gitignore
```

---

## 11. Quick reference card

### Branch commands

```bash
git branch                       # list local branches
git branch -a                    # list all branches
git checkout -b feature/xyz      # create + switch (classic)
git switch -c feature/xyz        # create + switch (modern)
git switch main                  # switch branch
git switch -                     # switch to previous branch
git branch -d feature/xyz        # delete merged branch
git branch -D feature/xyz        # force delete
git branch -m new-name           # rename current branch
git remote prune origin          # clean stale remote refs
```

### Merge commands

```bash
git merge feature/xyz            # merge (auto FF or 3-way)
git merge --no-ff feature/xyz    # always create merge commit
git merge --squash feature/xyz   # squash all commits, manual commit
git merge --abort                # cancel in-progress merge
git branch --merged              # branches already merged
```

### Conflict resolution

```bash
git status -s                    # find UU (conflicted) files
vim conflicted-file.md           # edit — remove markers, keep final
grep -n "<<<<<<\|======\|>>>>>>" file.md   # verify markers gone
git add conflicted-file.md       # mark resolved
git commit                       # complete the merge
```

### Rebase commands

```bash
git rebase main                  # replay commits on top of main
git rebase -i HEAD~N             # interactive rebase last N commits
git rebase -i --root             # interactive rebase all commits
git rebase --abort               # cancel in-progress rebase
git rebase --continue            # continue after resolving conflict
git push --force-with-lease      # force push after rebase (safer)
```

### GitHub / Remote

```bash
git push -u origin feature/xyz   # push branch + set upstream
git push --force-with-lease      # force push safely
git fetch                        # download remote changes (no merge)
git pull --rebase                # fetch + rebase instead of merge
git remote prune origin          # remove stale remote refs
```

### Tags (GitFlow releases)

```bash
git tag -a v1.0.0 -m "message"  # create annotated tag
git tag                          # list all tags
git show v1.0.0                  # view tag details
git push origin v1.0.0           # push specific tag
git push origin --tags           # push all tags
git tag -d v1.0.0                # delete tag locally
```

### HEAD navigation

```bash
HEAD       # current commit
HEAD~1     # one commit before (parent)
HEAD~3     # three commits before
HEAD^2     # second parent of a merge commit
```

---

## Appendix: Week 2 skills summary

```
Day 08  ✅  Branches: create, switch, list, HEAD pointer
Day 09  ✅  Merging: fast-forward vs 3-way, --no-ff
Day 10  ✅  Merge conflicts: detect, resolve, abort
Day 11  ✅  git rebase: linear history, squash, rebase onto
Day 12  ✅  GitHub PRs: open, review, address feedback, merge
Day 13  ✅  Team workflow: full feature delivery cycle
Day 14  ✅  Week 2 quiz: 10/10 perfect score
```

**All Week 2 Git fundamentals mastered.**

---

*Document version: Week 2 complete — Days 8–14 covered*
*Previous: [git-week1-reference.md](git-week1-reference.md)*
*Next: Week 3 — GitFlow, trunk-based dev, hooks, CI/CD integration*
