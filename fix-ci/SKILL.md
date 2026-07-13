---
name: fix-ci
description: Diagnose and fix failing GitHub Actions CI by inspecting runs with the GitHub CLI, reproducing as much of the repository's formatting, linting, type-checking, build, and test surface locally as practical, then committing, pushing, and monitoring the exact pushed revision. Use when the user asks to fix CI, repair a failing GitHub check, make a branch green, or investigate a GitHub Actions failure.
---

# Fix CI

Treat GitHub Actions as the final confirmation, not the primary feedback loop. Reproduce failures locally, make the smallest correct fix, and push only after the relevant local gates pass.

## Workflow

1. Inspect the repository and branch state.
   - Read `git status`, the current branch, recent commits, repository instructions, and the relevant workflow files.
   - Preserve unrelated user changes. Do not reset, discard, or stash them.
   - Use existing package scripts, task runners, pre-commit hooks, and documented commands instead of inventing equivalents.

2. Inspect GitHub Actions with `gh`.
   - Confirm `gh` authentication and identify the PR or current branch.
   - Use `gh pr checks`, `gh run list`, and `gh run view` to find the failing run and job.
   - Read failed logs with `gh run view <run-id> --log-failed` or the narrowest useful job log.
   - Distinguish the first actionable failure from jobs that were skipped or cancelled downstream.

3. Build a local acceptance set.
   - Map the failing workflow steps to repository-native local commands.
   - Include as much of CI as practical: formatting, linting, type-checking, builds, and tests.
   - Run the narrowest relevant checks first for fast iteration, then the broader package or repository gates before committing.
   - Match CI versions, flags, working directories, and environment assumptions when they affect the result.

4. Reproduce and fix locally.
   - Fix the root cause with the smallest scoped change.
   - Do not weaken checks, lower coverage thresholds, add broad exclusions, or rerun blindly merely to get green. Make such policy changes only when they correctly describe the intended checked surface.
   - Treat environmental failures separately from code failures. Retry only when evidence indicates a transient setup, cache, network, or concurrency problem.
   - If unrelated worktree changes contaminate a broad check, isolate validation with repository-native affected filters or a temporary clean worktree. Never alter the user's unrelated files to obtain a clean result.

5. Pass local gates before publishing.
   - Rerun every focused check that failed.
   - Run the broadest practical local equivalent of the affected CI workflows.
   - Do not commit or push while a relevant local failure remains unexplained.
   - Stage only the CI fix. Inspect the staged diff before committing.
   - Do not bypass hooks unless unrelated worktree state makes them unreliable and the equivalent hook checks have already passed in an isolated environment.

6. Push and watch the exact revision.
   - Push normally; do not force-push without explicit user approval.
   - Record the pushed commit SHA and find the new run for that SHA.
   - Watch it with `gh run watch <run-id> --exit-status` or `gh pr checks --watch` until completion.
   - Do not report success from a queued or still-running workflow.

7. Iterate on concrete failures.
   - If GitHub exposes a broader failure than local checks caught, inspect the new failed log, add its closest local reproduction, fix it, and repeat the local-first loop.
   - Continue until the relevant checks are green or a genuine external blocker requires user input.

## Decision Rules

- Ask the user before changing CI policy, supported platforms, coverage scope, security gates, or required checks.
- Ask before deleting or rewriting unrelated user work.
- Proceed without asking for ordinary formatting, lint, typing, build, and test fixes whose intended behavior is clear.
- Prefer the first causal failure. A cancelled test job after lint failed is not evidence of a test failure.
- Keep monitoring after the push; fixing CI includes confirming GitHub Actions, not merely starting it.

## Report

Report:

- The root cause and fix.
- The exact local commands that passed.
- The commit SHA pushed.
- The final GitHub Actions result and run URL.
- Any checks not reproduced locally and why.
