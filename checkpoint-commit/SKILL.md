---
name: checkpoint-commit
description: Commit and push the current task's relevant changes without hooks or checks, then open a draft GitHub PR if the branch has no open PR. Use only when the user explicitly asks for a checkpoint commit.
---

# Checkpoint commit

Treat the request as approval to publish the current reviewed slice. Scope it, commit it, push it, ensure it has a PR, and return.

## Workflow

1. Read the task context, `git status --short --branch`, and the current branch. Identify only the files changed for the current task. Inspect file names or a concise diff only when needed to resolve scope. Do not review the implementation again.
2. Preserve unrelated changes. Stage relevant paths explicitly with `git add -- <paths>`, including relevant untracked files. Do not use `git add .` or `git add -A` when the worktree contains unrelated changes.
3. Confirm the staged file list matches the task. If nothing relevant changed, return without creating an empty commit.
4. Create a new commit with a short message describing the slice. Use `git commit --no-verify`; do not amend an earlier commit.
5. Push the current branch with `git push --no-verify`. If it has no upstream, use `git push --no-verify -u origin HEAD`. Never force-push.
6. Check for an open PR whose head is the current branch. Reuse any existing open PR without changing its draft state. If none exists, create one with `gh pr create --draft --fill`.
7. Return immediately with:
   - A one-line summary containing the short commit SHA and branch.
   - A compact tree of the files in the new commit. Derive the paths from the commit, not the remaining worktree.
   - A clickable Markdown link to the PR, including its number and title when available.

Do not run tests, formatters, linters, builds, hooks, CI commands, or GitHub checks. Do not wait for or monitor CI. If the branch is detached, the current branch is the repository's default branch, the relevant file set is ambiguous, or commit, push, or PR creation fails, stop and report the blocker without expanding into unrelated cleanup or diagnosis.
