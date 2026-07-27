# Session 6 — Version Control — Git

## Lab 13 — Git Workflows — Branching, Merging & Conflict Resolution

> **🎯 Objective:** Master Git branching strategies, pull requests (PRs), merging, and conflict resolution used in DevOps teams.

---

## 🧭 Overview

Git is a distributed version control system that enables DevOps teams to collaborate on source code, track changes, work in parallel, and safely integrate changes through controlled workflows.

In this lab, you will practice:

* Git configuration
* Repository initialization and cloning
* Branch creation and management
* Conventional Commits
* Feature branches
* Pull requests
* Rebasing and merging
* Merge conflict resolution
* Stashing unfinished work
* Cherry-picking commits
* Reverting changes
* Git Flow
* GitHub branch protection
* Commit-message validation

---

# 🔄 Core Git Workflow

The standard workflow used throughout this lab is:

```text
Working Directory
       │
       ▼
   git add
       │
       ▼
Staging Area
       │
       ▼
  git commit
       │
       ▼
 Local Repository
       │
       ▼
  git push
       │
       ▼
Remote Repository
       │
       ▼
 Pull Request
       │
       ▼
Code Review + CI
       │
       ▼
     main
```

---

# ⚙️ Git Setup

Configure your Git username:

```bash
git config --global user.name "Your Name"
```

Configure your Git email:

```bash
git config --global user.email "you@example.com"
```

Initialize a new Git repository:

```bash
git init
```

Clone an existing repository:

```bash
git clone <repo-url>
```

Verify your configuration:

```bash
git config --global --list
```

---

# 🌿 Branching Workflow

Create and switch to a feature branch:

```bash
git checkout -b feature/login-page
```

> 💡 **Best Practice:** Feature branches isolate development work from the stable `main` branch.

Check repository status:

```bash
git status
```

Stage all changes:

```bash
git add .
```

Commit changes using a Conventional Commit message:

```bash
git commit -m "feat: add login page"
```

Push the feature branch:

```bash
git push origin feature/login-page
```

---

# 🔄 Synchronize with `main`

Fetch the latest remote information:

```bash
git fetch origin
```

Rebase your branch onto the latest `main`:

```bash
git rebase origin/main
```

Alternatively, merge the latest `main` into your current branch:

```bash
git merge origin/main
```

### Rebase vs. Merge

```text
Rebase:

feature ── A ── B
               │
main ── M ─────┘

After rebase:

main ── M ── A' ── B'
```

Rebasing creates a cleaner linear history but rewrites commit history on the rebased branch.

Merging preserves the existing branch history and may create a merge commit.

> ⚠️ **Warning:** Avoid rebasing shared branches that other developers are actively using unless your team has explicitly agreed on the workflow.

---

# ⚔️ Conflict Resolution

Check the repository status:

```bash
git status
```

Git will identify files containing conflicts.

Open the conflicted file:

```bash
nano conflicted-file.txt
```

A conflict may look like:

```text
<<<<<<< HEAD
Content from your branch
=======
Content from another branch
>>>>>>> other-branch
```

Edit the file and remove:

```text
<<<<<<<
=======
>>>>>>>
```

Keep the desired final content.

Stage the resolved file:

```bash
git add conflicted-file.txt
```

If the conflict occurred during a rebase:

```bash
git rebase --continue
```

---

# 🧰 Useful Git Commands

## View Commit History

```bash
git log --oneline --graph --all
```

This provides a compact visual representation of branches and commits.

---

## Save Unfinished Work

Temporarily save uncommitted changes:

```bash
git stash
```

Restore the stashed changes:

```bash
git stash pop
```

---

## Apply a Single Commit

Use cherry-pick to apply one specific commit:

```bash
git cherry-pick <commit-sha>
```

> 💡 **Use Case:** Cherry-picking is useful when a specific fix needs to be transferred to another branch without merging the entire branch.

---

## Undo the Last Commit — Dangerous

```bash
git reset --hard HEAD~1
```

> ⚠️ **DANGER:** `git reset --hard` can permanently discard uncommitted work and changes. Use it carefully, especially on shared branches.

---

## Safely Undo a Commit

```bash
git revert <sha>
```

`git revert` creates a new commit that reverses the specified commit while preserving the existing history.

> ✅ **Best Practice:** Prefer `git revert` for commits that have already been pushed to a shared branch.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                               |
| -------- | ---------------------------------------------------------------------------------------- |
| **T1.1** | Clone your GitHub repo. Create branch: `feature/hello`. Add `hello.py`. Commit and push. |
| **T1.2** | On GitHub: open Pull Request from `feature/hello` → `main`. Add description. Merge it.   |
| **T1.3** | On local: `git pull origin main`. Verify your feature file is now in `main`.             |

---

## T1.1 — Create and Push a Feature Branch

Clone your GitHub repository:

```bash
git clone <repo-url>
```

Enter the repository:

```bash
cd <repository-name>
```

Check the current branch:

```bash
git branch
```

Create the feature branch:

```bash
git checkout -b feature/hello
```

Create `hello.py`:

```bash
touch hello.py
```

Add the following content:

```python
print("Hello DevOps!")
```

Check the changes:

```bash
git status
```

Stage the file:

```bash
git add hello.py
```

Commit:

```bash
git commit -m "feat: add hello script"
```

Push the branch:

```bash
git push origin feature/hello
```

Verify the branch on GitHub.

---

## T1.2 — Create a Pull Request

Open your GitHub repository.

Create a Pull Request:

```text
Source:
feature/hello

Target:
main
```

Add a meaningful PR description.

Example:

```text
## Summary

Added a simple Python Hello World script.

## Changes

- Added hello.py
- Added DevOps greeting output

## Testing

Executed the script locally and verified the output.
```

Review the changes.

Merge the Pull Request into `main`.

> 💡 **DevOps Practice:** In a professional environment, PRs should normally go through code review and automated CI checks before merging.

---

## T1.3 — Synchronize the Local Repository

Switch to `main`:

```bash
git checkout main
```

Pull the latest changes:

```bash
git pull origin main
```

Verify the file:

```bash
ls -la
```

Run:

```bash
python hello.py
```

Expected output:

```text
Hello DevOps!
```

Check the Git history:

```bash
git log --oneline --graph --all
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                             |
| -------- | ------------------------------------------------------------------------------------------------------ |
| **T2.1** | Simulate conflict: 2 developers edit the same file. Create conflict, resolve it, and commit the merge. |
| **T2.2** | Use `git stash`: start editing, stash, do unrelated work, unstash, and continue.                       |
| **T2.3** | Use `git revert` to safely undo a bad commit without rewriting history.                                |

---

## T2.1 — Simulate and Resolve a Merge Conflict

Create a branch:

```bash
git checkout -b feature/dev-a
```

Edit the same file that another developer will modify.

For example:

```bash
nano README.md
```

Commit the change:

```bash
git add README.md
git commit -m "docs: update README"
```

Create another branch representing another developer:

```bash
git checkout main
git checkout -b feature/dev-b
```

Modify the same section of the same file:

```bash
nano README.md
```

Commit:

```bash
git add README.md
git commit -m "docs: update README content"
```

Switch to `feature/dev-a`:

```bash
git checkout feature/dev-a
```

Merge the second branch:

```bash
git merge feature/dev-b
```

Git may report a conflict.

Check:

```bash
git status
```

Open the conflicted file:

```bash
nano README.md
```

Resolve the conflicting content.

Stage the resolution:

```bash
git add README.md
```

Complete the merge:

```bash
git commit
```

Verify:

```bash
git log --oneline --graph --all
```

---

## T2.2 — Use Git Stash

Start editing a file:

```bash
nano README.md
```

Check the changes:

```bash
git status
```

Stash the unfinished work:

```bash
git stash
```

Verify the working directory:

```bash
git status
```

Perform unrelated work.

When ready, restore your unfinished changes:

```bash
git stash pop
```

Check:

```bash
git status
```

View available stashes:

```bash
git stash list
```

---

## T2.3 — Safely Revert a Bad Commit

View recent commits:

```bash
git log --oneline
```

Identify the commit SHA that needs to be reversed.

Run:

```bash
git revert <commit-sha>
```

Git creates a new commit that reverses the selected change.

Verify:

```bash
git log --oneline
```

Push the revert:

```bash
git push origin main
```

> 🛡️ **Best Practice:** `git revert` is safer than rewriting shared history because the original commit remains visible.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                                      |
| -------- | --------------------------------------------------------------------------------------------------------------- |
| **T3.1** | Implement Git Flow: `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` branches. Simulate a release cycle. |
| **T3.2** | Set up branch protection on GitHub: require PR review + CI pass before merge to `main`.                         |
| **T3.3** | Add a `commit-msg` hook that enforces Conventional Commits format (`feat:`, `fix:`, `chore:`, `docs:`).         |

---

# T3.1 — Implement Git Flow

A Git Flow-style model can be represented as:

```text
                         feature/login
                              │
                              ▼
main ────────────────────●───────────────●──────
                         │               ▲
                         ▼               │
                      develop ──────────┘
                         │
                         ▼
                    release/1.0
                         │
                         ▼
                       main

hotfix/critical-fix ───────────────────────► main
```

Create `develop`:

```bash
git checkout main
git checkout -b develop
git push -u origin develop
```

Create a feature branch:

```bash
git checkout develop
git checkout -b feature/login
```

Make changes and commit:

```bash
git add .
git commit -m "feat: add login functionality"
```

Push:

```bash
git push -u origin feature/login
```

Merge the feature into `develop` through a Pull Request.

---

## Create a Release Branch

Create:

```bash
git checkout develop
git checkout -b release/1.0
```

Push:

```bash
git push -u origin release/1.0
```

Use this branch for final release preparation, testing, and documentation.

After validation, merge the release into `main`.

Then merge the release changes back into `develop` so both branches remain synchronized.

---

## Create a Hotfix Branch

For an urgent production issue:

```bash
git checkout main
git checkout -b hotfix/critical-fix
```

Apply the fix:

```bash
git add .
git commit -m "fix: resolve critical production issue"
```

Push:

```bash
git push -u origin hotfix/critical-fix
```

Create a Pull Request into `main`.

After the fix is released, synchronize the change back into `develop`.

---

# T3.2 — Configure GitHub Branch Protection

Open your GitHub repository.

Navigate to:

```text
Repository
   │
   └── Settings
        │
        └── Branches / Rules
```

Create a branch protection rule or ruleset for:

```text
main
```

Configure protections such as:

* Require a Pull Request before merging
* Require approvals
* Require status checks to pass
* Require branches to be up to date before merging
* Restrict direct pushes where appropriate
* Require conversation resolution
* Prevent force pushes
* Prevent branch deletion

A recommended DevOps workflow is:

```text
Developer
    │
    ▼
feature/*
    │
    ▼
Pull Request
    │
    ├── Code Review
    │
    ├── CI Build
    │
    ├── Automated Tests
    │
    └── Security Checks
           │
           ▼
      Approved PR
           │
           ▼
          main
```

> 🛡️ **Security Principle:** Production branches should not depend on developer discipline alone. Repository rules should enforce the required controls.

---

# T3.3 — Enforce Conventional Commits

Conventional Commits provide a standardized structure for commit messages.

Common prefixes include:

```text
feat:     New feature
fix:      Bug fix
chore:    Maintenance task
docs:     Documentation change
```

Examples:

```bash
git commit -m "feat: add user authentication"
git commit -m "fix: correct login validation"
git commit -m "docs: update deployment guide"
git commit -m "chore: update dependencies"
```

---

## Create a `commit-msg` Hook

Create the Git hooks directory if required:

```bash
mkdir -p .git/hooks
```

Create the hook:

```bash
nano .git/hooks/commit-msg
```

Add:

```bash
#!/bin/sh

commit_msg_file="$1"
commit_msg=$(head -n 1 "$commit_msg_file")

if ! printf '%s\n' "$commit_msg" | grep -Eq '^(feat|fix|chore|docs): .+'; then
    echo "ERROR: Invalid commit message."
    echo "Use one of:"
    echo "  feat: description"
    echo "  fix: description"
    echo "  chore: description"
    echo "  docs: description"
    exit 1
fi

exit 0
```

Make the hook executable:

```bash
chmod +x .git/hooks/commit-msg
```

---

## Test the Hook

Try an invalid commit:

```bash
git commit -m "added something"
```

The hook should reject it.

Try a valid commit:

```bash
git commit -m "feat: add login functionality"
```

The commit should be accepted.

> 💡 **Important:** Git hooks stored only inside `.git/hooks` are local to each clone and are not normally committed to the repository. For team-wide enforcement, consider implementing the same validation in CI or using a version-controlled hook framework.

---

# 🔍 Git Branching Strategy

A practical branch structure for the lab is:

```text
main
│
├── develop
│   │
│   ├── feature/login
│   ├── feature/dashboard
│   └── feature/api
│
├── release/1.0
│
└── hotfix/critical-fix
```

### Branch Responsibilities

| Branch        | Purpose                               |
| ------------- | ------------------------------------- |
| **main**      | Stable, production-ready code         |
| **develop**   | Integration branch for upcoming work  |
| **feature/*** | Individual feature development        |
| **release/*** | Release preparation and stabilization |
| **hotfix/***  | Urgent production fixes               |

---

# 🧪 Git Troubleshooting

## Check Current Branch

```bash
git branch --show-current
```

---

## View Working Tree Changes

```bash
git status
```

---

## View Changes Before Commit

```bash
git diff
```

---

## View Staged Changes

```bash
git diff --cached
```

---

## Show Remote Repository

```bash
git remote -v
```

---

## Fetch Remote Changes

```bash
git fetch origin
```

---

## Update Local Main

```bash
git checkout main
git pull origin main
```

---

## View Branches

Local branches:

```bash
git branch
```

Remote branches:

```bash
git branch -r
```

All branches:

```bash
git branch -a
```

---

## Abort an In-Progress Rebase

If you encounter a problem during a rebase:

```bash
git rebase --abort
```

---

## Abort a Merge

If a merge conflict cannot be resolved and you want to return to the previous state:

```bash
git merge --abort
```

> ⚠️ **Note:** These commands discard the in-progress merge/rebase operation. Review `git status` before using them.

---

# 🛡️ Git Best Practices

* Use meaningful branch names.
* Keep feature branches focused and short-lived.
* Make small, logical commits.
* Use Conventional Commits consistently.
* Never commit passwords, API keys, tokens, or private keys.
* Use `.gitignore` for generated files and local secrets.
* Pull or fetch regularly to reduce integration conflicts.
* Use Pull Requests for important changes.
* Require code review for protected branches.
* Run CI checks before merging.
* Prefer `git revert` for shared history.
* Avoid `git reset --hard` on shared branches.
* Avoid force-pushing shared branches.
* Resolve conflicts carefully and test afterward.
* Keep `main` stable and production-ready.
* Tag important releases.
* Keep documentation updated alongside code.

---

# 🚨 Security Reminder

Never commit credentials such as:

```text
passwords
API keys
AWS access keys
private SSH keys
database credentials
tokens
.env files containing secrets
```

For example, do **not** commit:

```text
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
DB_PASSWORD=SuperSecretPassword
```

Instead, use appropriate secret-management mechanisms such as:

* GitHub Actions Secrets
* AWS Secrets Manager
* AWS Systems Manager Parameter Store
* Kubernetes Secrets
* External Secrets Operator

---

# 📋 Lab 13 Completion Checklist

* [ ] Configure Git username and email.
* [ ] Initialize a Git repository.
* [ ] Clone a GitHub repository.
* [ ] Create a feature branch.
* [ ] Create and commit a new file.
* [ ] Push a feature branch to GitHub.
* [ ] Create a Pull Request.
* [ ] Review and merge a Pull Request.
* [ ] Pull the merged changes locally.
* [ ] Understand `git fetch`.
* [ ] Understand `git merge`.
* [ ] Understand `git rebase`.
* [ ] Create and resolve a merge conflict.
* [ ] Use `git stash`.
* [ ] Use `git stash pop`.
* [ ] Use `git cherry-pick`.
* [ ] Understand the risks of `git reset --hard`.
* [ ] Safely undo a commit with `git revert`.
* [ ] Implement a Git Flow-style branch structure.
* [ ] Create `feature/*` branches.
* [ ] Create a `release/*` branch.
* [ ] Create a `hotfix/*` branch.
* [ ] Configure GitHub branch protection.
* [ ] Require PR review before merging.
* [ ] Require CI checks before merging.
* [ ] Implement Conventional Commit messages.
* [ ] Create a `commit-msg` hook.
* [ ] Test valid and invalid commit messages.
* [ ] Understand Git security best practices.

---

# 🧠 Lab 13 — Key Takeaways

| Git Concept              | Purpose                                                |
| ------------------------ | ------------------------------------------------------ |
| **Repository**           | Stores project history and source code                 |
| **Branch**               | Isolated line of development                           |
| **Feature Branch**       | Used for individual development work                   |
| **Commit**               | Records a set of changes                               |
| **Push**                 | Sends local commits to a remote repository             |
| **Pull**                 | Retrieves and integrates remote changes                |
| **Fetch**                | Retrieves remote information without integrating it    |
| **Merge**                | Combines branches                                      |
| **Rebase**               | Replays commits on top of another branch               |
| **Pull Request**         | Controlled mechanism for reviewing and merging changes |
| **Conflict**             | Occurs when Git cannot automatically combine changes   |
| **Stash**                | Temporarily stores unfinished changes                  |
| **Cherry-pick**          | Applies a specific commit to another branch            |
| **Revert**               | Safely reverses a previous commit                      |
| **Git Flow**             | Structured branching strategy                          |
| **Branch Protection**    | Prevents unsafe direct changes to protected branches   |
| **Conventional Commits** | Standardized commit-message format                     |

---

# 🔄 End-to-End DevOps Git Workflow

```text
                    GitHub Repository
                           │
                           ▼
                         main
                           │
                           ▼
                       develop
                           │
                           ▼
                     feature/login
                           │
                           ▼
                    Developer Changes
                           │
                           ▼
                         Commit
                           │
                           ▼
                          Push
                           │
                           ▼
                    Pull Request
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
         Code Review      CI/CD       Security
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                       PR Approved
                           │
                           ▼
                     Merge to develop
                           │
                           ▼
                    release/1.0
                           │
                           ▼
                          main
                           │
                           ▼
                    Production 🚀
```

> **🎓 Lab Outcome:** By completing Lab 13, you should be able to work effectively with Git in a DevOps team environment, use feature branches and Pull Requests, resolve merge conflicts, manage releases and hotfixes, protect production branches, and enforce consistent commit standards.
