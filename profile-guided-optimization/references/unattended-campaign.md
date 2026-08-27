# Unattended optimization campaign

Read this reference only after the user explicitly authorizes a long-running or unattended campaign. The campaign does not itself authorize commits, pushes, PR edits, dependency changes, permission changes, or a broader behavior change. Resolve each authority from the user's request.

## Preflight

Record:

- The accepted starting revision and clean working state.
- The behavior and quality contract.
- The benchmark, profiler, debug oracle, fixed-iteration run, sustained run, and CI commands.
- Whether accepted changes may be committed, pushed, and added to an existing or new PR.
- The stop condition, such as a deadline, explicit wrap-up, exhausted profiler-backed hypotheses, or a maximum number of attempts.
- Machine constraints, background workloads, protected folders, required credentials, and operations that will be skipped if permission is unavailable.

Never request or store a sudo password. If the platform repeatedly prompts when a repository under a protected folder launches a benchmark binary, copy the target directory, executable, input, config, working directory, optional output, and profiler artifacts into a dedicated temporary directory. Run there. Record the workaround in the workflow ledger so every agent uses it.

## Keep restart-safe ledgers

For a campaign likely to cross context compaction, restarts, or several experiments, create a repository-local directory such as `.agent-notes/performance-optimization/` with three files. Ignore it by default. Commit it only when the user asks or when durable team reuse is explicitly part of the task.

### `WORKFLOW.md`

Keep stable operational facts here:

- Exact commands and fixtures for every gate.
- Metric and quality definitions.
- Baseline and latest accepted revisions.
- Profiler setup, symbol and frame-pointer settings, and temporary no-inline procedure.
- Benchmark-environment requirements.
- Permissions and temporary-directory workarounds.
- Delegation boundaries, PR update rules, and stop conditions.

### `EXPERIMENTS.md`

Record both accepted and rejected work. Each entry should contain:

- Exact parent revision.
- Hypothesis and profiler evidence.
- Changed files or representation boundary.
- Fast-gate result with interval when available.
- Exact, equivalent, or changed behavior result.
- Sustained throughput and solution quality when the candidate reached acceptance.
- Decision, commit or candidate reference, and the lesson that prevents repetition.

Do not erase rejected work. A rejected ledger often saves more time than the accepted-change report.

### `BACKLOG.md`

Keep profiler-backed hypotheses, possible representation changes, cleanup ideas, and known ceilings. This is a queue of experiments, not a commitment. Re-profile after each accepted change and reorder it from current evidence.

Read all three files at the start of a resumed session, after context compaction, before repeating a familiar idea, and before declaring the campaign complete.

## Use two lanes

Use sub-agents only when delegation is available and the acceptance work is expensive enough to overlap useful discovery.

### Discovery lane

The primary agent:

1. Works from the latest accepted head.
2. Profiles, forms one hypothesis, implements one experiment, and runs the compile plus fast whole-workload gate.
3. Rejects clear losses immediately and writes the experiment ledger.
4. Snapshots a promising candidate on its own temporary branch or worktree without mixing unrelated changes.
5. Returns its discovery checkout to the accepted head and profiles or develops the next independent hypothesis. It does not build a speculative stack on top of an unaccepted candidate.

### Acceptance lane

Give one acceptance owner a separate worktree containing only the candidate and its exact parent. Ask it to inspect the diff independently and perform the full acceptance gate:

1. Semantic, invariant, caller, and restoration-path audit.
2. Fixed-seed, fixed-iteration behavior and quality comparison.
3. Small debug full-system run with invariant assertions.
4. Controlled adjacent benchmark, repeated in isolation when needed.
5. Sustained production run with throughput and quality.
6. Relevant tests, all-target check, diff check, and lint or formatting surface.
7. Accepted commit, push, live report update, and exact pushed-revision CI only when those actions were authorized.

The acceptance owner must report a failed candidate without quietly repairing it into a different experiment. A small correctness repair that preserves the hypothesis may be proposed to the primary agent, but it needs a fresh fast gate.

## Coordinate the lanes

- Never let two agents edit the same worktree.
- Keep one writer for the PR body or shared ledger at a time.
- Use exact commit hashes in handoffs. Do not refer only to "current" or "latest."
- Final performance runs should be sequential and isolated. Concurrent experiments may provide rough triage only; record the overlap and repeat any result used for acceptance.
- When an accepted candidate lands, rebase or recreate later candidates on the new accepted head and re-measure them. Do not carry forward their old speedup claim.
- Do not remove a candidate branch or worktree until its result is recorded and any accepted commit is reachable from the authorized branch.
- Keep the discovery lane moving only on independent work. If the next hypothesis depends on the candidate being accepted, profile another area or wait for the acceptance result.

## Maintain a live PR report

Use a live PR only when authorized. Serialize body edits and keep the report useful to someone returning hours later.

The overview should contain only accepted performance-relevant changes:

| # | Commit | Change | Sustained rate | Delta vs baseline | Behavior |
|---:|---|---|---:|---:|---|

- Link each commit hash to the exact commit.
- Use consecutive performance-entry numbers, or include explicit "measurement only" and "docs only" rows. Do not create unexplained numbering gaps.
- Prefer numbered entries over fragile generated heading anchors in a PR description.
- State the frozen baseline revision and raw rate below the table.
- Label cumulative single-run deltas as checkpoints, not statistical intervals.

Give each accepted change a short numbered section with:

- Before and after.
- Why the representation or control-flow change was plausible.
- Immediate-parent benchmark result and interval.
- Exact or equivalent behavior result.
- Sustained throughput and solution quality.
- Any remaining limitation or follow-up separated from the accepted claim.

Keep measurement-only setup and rejected experiments out of the overview. Put measurement methodology in its own section and rejected experiments in the ledger. If a claim proves invalid because of baseline drift, dependency mismatch, load, or concurrency, correct the PR immediately and state why.

## Stop and wrap up

When the deadline or explicit wrap-up arrives:

1. Start no new experiments.
2. Let acceptance owners finish candidates already in flight, or reject and record them if they cannot finish safely.
3. Integrate only candidates that pass every gate and were based on the correct parent.
4. Remove temporary no-inline markers, profiler scaffolding, generated configs, traces, and disposable worktrees without touching user-owned changes.
5. Re-run final build, tests, behavior oracle, and CI on the exact pushed revision when pushing was authorized.
6. Reconcile the overview, detailed sections, experiment ledger, backlog, and accepted head.
7. Report the final accepted commit, cumulative checkpoint, behavior status, CI state, rejected count, and remaining profiler-backed ideas.

Do not extend the stop condition because one more idea looks easy.
