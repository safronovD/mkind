---
name: "git-workflow"
description: "Enforce this repo's git workflow: no merges without a PR, one GitHub issue per PR, and branch names shaped usr/<git-user>/<ISSUE-ID>-<short-description>."
argument-hint: "Short description of the change (e.g. \"fix subnet collision on multi-host bootstrap\")"
compatibility: "Requires git, a GitHub remote, and the gh CLI (gh auth login)"
metadata:
  author: "mkind"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). It is the change description used to derive the issue title and branch slug. If empty, ask the user for a one-line description before continuing.

## Rules (non-negotiable)

1. **No direct merges to `main`.** Every change lands via a pull request. Never `git push origin <branch>:main`, never `git merge` into a local `main` that gets pushed, never use admin/force-merge to bypass review.
2. **One GitHub issue per PR.** Every PR MUST reference an issue (create one first if none exists). The PR body MUST include a closing keyword (`Closes #<ISSUE-ID>`) so the issue auto-closes on merge.
3. **Branch naming**: `usr/<git-user>/<ISSUE-ID>-<short-description>`
   - `<git-user>`: `git config user.name`, lowercased, spaces → hyphens, stripped of any character outside `[a-z0-9-]`. Fall back to the local part of `git config user.email` if `user.name` is unset.
   - `<ISSUE-ID>`: the numeric GitHub issue number.
   - `<short-description>`: 3-6 words distilled from the change description, lowercased, hyphen-separated, no articles.
   - Example: `usr/danil-safronov/42-fix-subnet-collision`

## Prerequisites

- Verify `gh` is installed: `gh --version`. If missing, stop and tell the user to install the GitHub CLI (`gh`) and run `gh auth login` — do not fall back to raw REST calls.
- Verify `gh` is authenticated: `gh auth status`.
- Verify the working tree is clean (`git status --porcelain`). If there are uncommitted changes unrelated to this task, stop and ask how to proceed rather than stashing or discarding anything.
- Verify the current branch is up to date with `origin/main`: `git fetch origin main` then branch from `origin/main`, not a possibly-stale local `main`.

## Execution

1. **Compute the git-user slug** per the Branch naming rule above.
2. **Create the GitHub issue** (skip if the user supplied an existing issue number in `$ARGUMENTS`):
   `gh issue create --title "<concise title>" --body "<1-3 sentence description derived from the input>"`
   Capture the issue number from the output URL.
3. **Compute the short-description slug** from the change description.
4. **Create the branch** from `origin/main`:
   `git checkout -b usr/<git-user>/<ISSUE-ID>-<short-description> origin/main`
5. **Do the implementation work** (this skill does not write code itself — it hands off to normal editing once the branch exists).
6. **Commit** using this repo's conventional-commit style (see the project constitution).
7. **Before pushing or creating the PR, confirm with the user** — pushing a branch and opening a PR are visible, hard-to-fully-reverse actions per the standard safety protocol. Do not push or open the PR without explicit go-ahead.
8. **Push**: `git push -u origin usr/<git-user>/<ISSUE-ID>-<short-description>`
9. **Open the PR**:
   `gh pr create --base main --title "<title>" --body "Closes #<ISSUE-ID>\n\n<summary>"`
10. Report the issue URL and PR URL back to the user. Do not merge the PR yourself — merging happens after review/approval, and even then only through the PR (never a manual merge/push to `main`).

## Graceful Degradation

- If `gh` is unavailable or unauthenticated, stop before any git state changes and give the user the manual steps (create issue in the GitHub UI, note the number, then re-run this skill or follow the branch-naming rule manually).
- If the repo has no GitHub remote, tell the user this workflow requires one and stop.
