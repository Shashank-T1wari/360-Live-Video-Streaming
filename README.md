# Git Basics for Team Members

This document contains the most commonly used Git commands with simple explanations.

---

## 1. Clone the Repository

Used when downloading the repository for the first time.

```bash
git clone https://github.com/Shashank-T1wari/360-Live-Video-Streaming
```

What it does:

* Downloads the entire repository to your computer.
* Creates a local copy linked to GitHub.

---

## 2. Check Current Status

```bash
git status
```

What it does:

* Shows modified files.
* Shows untracked files.
* Shows files ready to commit.

---

## 3. Get Latest Changes from GitHub

```bash
git pull origin main
```

What it does:

* Downloads latest changes from GitHub.
* Merges them into your local branch.

Always run this before starting work.

---

## 4. Create a New Branch

```bash
git checkout -b <branch-name>
```

What it does:

* Creates a new branch.
* Switches to that branch.

---

## 5. View Current Branch

```bash
git branch
```

What it does:

* Shows all local branches.
* Current branch is marked with `*`.

---

## 6. Switch Branches

```bash
git checkout <branch-name>
```

Example:

```bash
git checkout main
```

What it does:

* Moves from one branch to another.

---

## 8. Add Files to Staging Area

Add one file:

```bash
git add filename.v
```

Add all modified files:

```bash
git add .
```

What it does:

* Marks files to be included in the next commit.

---

## 9. Commit Changes

```bash
git commit -m "Describe what changed"
```

What it does:

* Saves a snapshot of your changes locally.

---

## 10. Push Changes to GitHub

```bash
git push
```

What it does:

* Uploads committed changes to GitHub.

---

## 11. First Push of a New Branch

```bash
git push -u origin <branch-name>
```

What it does:

* Creates the branch on GitHub.
* Links local and remote branches.

---

## 12. View Commit History

```bash
git log
```

Compact version:

```bash
git log --oneline
```

What it does:

* Shows previous commits.
* Useful for debugging and tracking changes.

---

## 13. Fetch Remote Updates

```bash
git fetch
```

What it does:

* Downloads latest information from GitHub.
* Does NOT modify your files.

Think of it as:

```text
fetch = check for updates
pull  = download + merge updates
```

---

## 14. Discard Local Changes

Discard changes in a specific file:

```bash
git restore filename.v
```

Discard all unstaged changes:

```bash
git restore .
```

What it does:

* Reverts file(s) back to last committed version.


__NOTE:__

Changes are lost permanently.

---

## 15. Remove File from Staging Area

```bash
git restore --staged filename.v
```

What it does:

* Removes file from staging area.
* Keeps file modifications.

---

## 16. Delete a Branch

Delete local branch:

```bash
git branch -d branch-name
```

Force delete:

```bash
git branch -D branch-name
```

---

## 17. Merge a Branch

Switch to target branch first:

```bash
git checkout main
```

Merge:

```bash
git merge <branch-name>
```

What it does:

* Combines changes from `<branch-name>` into `main`.

---

## 18. Check Remote Repository

```bash
git remote -v
```

What it does:

* Shows connected GitHub repository.

Example:

```text
origin  https://github.com/Shashank-T1wari/360-Live-Video-Streaming (fetch)
origin  https://github.com/Shashank-T1wari/360-Live-Video-Streaming (push)
```

---

## 20. Stash Temporary Work

Save unfinished work:

```bash
git stash
```

Restore it:

```bash
git stash pop
```

What it does:

* Temporarily hides changes.
* Lets you switch branches safely.

---
## Golden Rules

1. Run `git pull` before starting work.
2. Run `git status` often.
3. Commit small changes frequently.
4. Write meaningful commit messages.
5. Never force push to shared branches unless instructed.
6. Create a new branch for major features.
7. Ask before deleting branches or rewriting history.

---