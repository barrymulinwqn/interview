# Git & Bitbucket — Interview Fundamentals

## Git Core Concepts

### What is Git?
Git is a distributed version control system (DVCS). Every clone is a full repository with complete history. Supports non-linear development through branching and merging.

### Data Model
```
Working Directory → (git add) → Staging Area (Index) → (git commit) → Local Repo → (git push) → Remote Repo
```

Git stores **snapshots** (not diffs) of the entire repository at each commit. A commit object contains:
- Tree (snapshot of files)
- Parent commit(s)
- Author, committer, timestamp
- Commit message
- SHA-1 hash (40 hex chars)

---

## Essential Commands

### Setup & Config
```bash
git config --global user.name "John Doe"
git config --global user.email "john@example.com"
git config --global core.editor "vim"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --list

# SSH key auth (preferred over HTTPS for automation)
ssh-keygen -t ed25519 -C "john@example.com"
```

### Repository Basics
```bash
git init
git clone https://bitbucket.org/org/repo.git
git clone git@bitbucket.org:org/repo.git    # SSH
git clone --depth 1 https://...             # shallow clone
git remote -v
git remote add upstream https://...
git remote set-url origin git@...
```

### Staging & Committing
```bash
git status
git diff                            # unstaged changes
git diff --staged                   # staged vs last commit
git add file.py
git add -p                          # interactive patch (partial add)
git add .
git reset HEAD file.py              # unstage file
git commit -m "feat: add pricing module"
git commit --amend                  # modify last commit (before push only)
git commit --amend --no-edit        # amend without changing message
```

### Branching
```bash
git branch                          # list local branches
git branch -a                       # all (local + remote)
git branch feature/JIRA-123         # create branch
git checkout feature/JIRA-123       # switch
git checkout -b feature/JIRA-123    # create + switch
git switch -c feature/JIRA-123      # modern syntax

git branch -d feature/done          # delete (safe — merged only)
git branch -D feature/done          # force delete

git branch -m old-name new-name     # rename
```

### Merging & Rebasing
```bash
# Merge (preserves history, creates merge commit)
git checkout main
git merge feature/JIRA-123
git merge --no-ff feature/JIRA-123  # always create merge commit
git merge --squash feature/JIRA-123 # squash into one commit

# Rebase (rewrites history — linear)
git checkout feature/JIRA-123
git rebase main
git rebase -i HEAD~3                 # interactive rebase (squash, edit, reorder)

# Cherry-pick
git cherry-pick abc1234              # apply specific commit
git cherry-pick abc1234..def5678     # range of commits
```

### Remote Operations
```bash
git fetch origin                     # download changes (no merge)
git fetch --all --prune              # fetch all remotes, remove stale branches
git pull                             # fetch + merge
git pull --rebase                    # fetch + rebase
git push origin feature/JIRA-123
git push -u origin feature/JIRA-123  # set upstream tracking
git push origin --delete feature/JIRA-123  # delete remote branch
git push --force-with-lease          # safe force push (checks for upstream changes)
```

### Undoing Changes
```bash
# Undo uncommitted changes
git restore file.py                  # discard working dir changes
git restore --staged file.py         # unstage (newer syntax)
git checkout -- file.py              # discard working dir (old syntax)
git clean -fd                        # remove untracked files/dirs

# Undo commits (safe — doesn't rewrite history)
git revert abc1234                   # new commit that undoes abc1234
git revert HEAD~2..HEAD              # revert range

# Undo commits (rewrites history — use with caution)
git reset --soft HEAD~1              # undo commit; keep staged
git reset --mixed HEAD~1             # undo commit; keep in working dir (default)
git reset --hard HEAD~1              # undo commit; DISCARD changes

# Recover lost commits
git reflog
git checkout abc1234                 # recover to detached HEAD
git branch recovery-branch abc1234
```

### Stashing
```bash
git stash                            # stash working dir + index
git stash push -m "WIP: pricing fix"
git stash list
git stash apply stash@{1}            # apply without removing
git stash pop                        # apply and remove
git stash drop stash@{1}
git stash branch feature/from-stash  # create branch from stash
git stash show -p                    # show stash diff
```

### Inspection & Log
```bash
git log
git log --oneline --graph --all --decorate    # visual branch graph
git log --author="John" --since="2025-01-01"
git log -p file.py                            # commits affecting a file
git log --follow -p -- old-name.py            # track file renames

git show abc1234                             # show commit details
git blame file.py                            # who changed each line
git bisect start                             # binary search for bug-introducing commit
git bisect bad HEAD
git bisect good v1.0.0

git diff main..feature/branch               # compare branches
git diff HEAD~3..HEAD                        # last 3 commits
```

---

## Branching Strategies

### Git Flow
```
main ──────────────────────────────► production
  └── develop ────────────────────► integration
        ├── feature/xxx            (feature branches, merge to develop)
        ├── release/1.0.0          (stabilisation, merge to main + develop)
        └── hotfix/xxx             (from main, merge to main + develop)
```
Best for: scheduled release cycles, versioned software.

### GitHub Flow / Trunk-Based Development
```
main ──────────────────────────────► always deployable
  └── feature/JIRA-123             (short-lived, merged via PR)
```
Best for: continuous delivery, fast iteration.

### Bitbucket Flow (common in enterprise)
```
main / master    → production
develop          → staging / integration
release/*        → release candidate
feature/*        → feature work (branched from develop)
hotfix/*         → urgent prod fixes (branched from main)
```

---

## Bitbucket-Specific Features

### Pull Requests
```
1. Push feature branch
2. Open Pull Request (PR) in Bitbucket
3. Code review — reviewers add comments/suggestions
4. Required checks: CI pipeline, minimum approvals
5. Merge strategies:
   - Merge commit (default)
   - Squash merge (clean history)
   - Fast-forward (if no divergence)
```

### Branch Permissions
- Prevent direct push to `main`/`develop`
- Require minimum N approvals
- Require passing builds (CI integration)
- Restrict who can merge

### Bitbucket Pipelines (CI/CD)
```yaml
# bitbucket-pipelines.yml
image: python:3.11-slim

pipelines:
  default:
    - step:
        name: Test
        caches:
          - pip
        script:
          - pip install -r requirements-dev.txt
          - pytest --cov=app tests/

  branches:
    main:
      - step:
          name: Test
          script:
            - pip install -r requirements-dev.txt
            - pytest
      - step:
          name: Deploy to Production
          deployment: production
          script:
            - ./scripts/deploy.sh production
          trigger: manual

  pull-requests:
    '**':
      - step:
          name: PR Validation
          script:
            - flake8 .
            - pytest

definitions:
  caches:
    pip: ~/.cache/pip
```

### SSH Keys in Bitbucket
```bash
# Add SSH key to Bitbucket account
# Settings → Security → SSH keys → Add key

# For automation: use Access Tokens (not passwords)
# Settings → App passwords → Create (scopes: repo read/write)
```

---

## Git Internals

### Object Types
| Type | Description |
|------|-------------|
| `blob` | File contents |
| `tree` | Directory listing (names + blobs/trees) |
| `commit` | Snapshot + metadata + parent |
| `tag` | Annotated tag |

```bash
git cat-file -t abc1234              # show object type
git cat-file -p abc1234              # show object content
git ls-tree HEAD                     # list tree
```

### Refs & HEAD
```
.git/
  HEAD               → points to current branch (or commit in detached state)
  refs/heads/main    → SHA of latest commit on main
  refs/remotes/      → remote tracking branches
  refs/tags/         → tags
  ORIG_HEAD          → saved before dangerous operations
  MERGE_HEAD         → commit being merged
```

---

## .gitignore

```gitignore
# Python
__pycache__/
*.pyc
*.pyo
.venv/
venv/
*.egg-info/
dist/
build/
.pytest_cache/

# Secrets (CRITICAL — never commit credentials)
.env
.env.*
!.env.example
secrets/
*.key
*.pem
*.p12

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Build artifacts
target/
*.jar
```

---

## Git Hooks

Located in `.git/hooks/` (local, not committed) or managed via tools like `husky`.

| Hook | When | Use Case |
|------|------|---------|
| `pre-commit` | Before commit is created | Lint, format check |
| `commit-msg` | After commit message entered | Enforce message format |
| `pre-push` | Before `git push` | Run tests |
| `post-receive` | After push received on server | Trigger CI/CD |

```bash
# pre-commit hook example
#!/usr/bin/env bash
set -e
echo "Running pre-commit checks..."
flake8 .
pytest --fast
echo "All checks passed."
```

---

## Common Interview Questions

### Q1: What is the difference between merge and rebase?
- **Merge**: Combines histories; creates a merge commit; preserves full context
- **Rebase**: Moves branch commits onto target; linear history; rewrites commit hashes
- Rule: Never rebase shared/public branches

### Q2: What is a detached HEAD state?
`HEAD` points directly to a commit SHA rather than a branch ref. Any commits made will be "orphaned" when you switch branches. Create a new branch to preserve them: `git switch -c new-branch`.

### Q3: What is the difference between `git fetch` and `git pull`?
- `git fetch`: Downloads remote changes to remote tracking branches; does NOT update working files
- `git pull`: `git fetch` + merge (or rebase) into current branch

### Q4: What is `git stash`?
Temporarily saves uncommitted changes (working directory + index) so you can switch branches with a clean state. Not a permanent storage — don't rely on it for long-term work.

### Q5: How do you resolve merge conflicts?
1. Run merge/rebase — conflicted files marked
2. Open files; conflict markers show both versions:
   ```
   <<<<<<< HEAD
   current branch code
   =======
   incoming branch code
   >>>>>>> feature/branch
   ```
3. Edit to desired state, remove markers
4. `git add file` to mark resolved
5. `git commit` (merge) or `git rebase --continue`

### Q6: What is `git cherry-pick`?
Applies the changes from specific commits to the current branch without merging the entire branch. Useful for backporting hotfixes.

### Q7: What is `git reflog`?
A local log of all HEAD movements (commits, checkouts, resets, merges). Allows recovery of "lost" commits after a hard reset. Reflog entries expire after 90 days by default.

### Q8: What is `--force-with-lease`?
A safer alternative to `--force`. Refuses to push if someone else has pushed to the remote branch since your last fetch, preventing accidental overwrites of others' work.

### Q9: How do you squash multiple commits?
```bash
git rebase -i HEAD~N   # N = number of commits to squash
# Change "pick" to "squash" or "s" for commits to squash
# Edit combined commit message
```
Or use `git merge --squash feature/branch` when merging.

### Q10: How do you find which commit introduced a bug?
```bash
git bisect start
git bisect bad HEAD       # current commit is bad
git bisect good v1.0.0    # known good commit
# Git checks out midpoint; test it
git bisect good/bad       # repeat until found
git bisect reset
```
