---
name: profile-guided-optimization
description: Optimize an existing stable algorithm or hot path through reproducible whole-workload baselines, profiler-guided experiments, fast rejection, behavior and quality gates, and optional sub-agent acceptance. Use for iterative or overnight throughput or latency campaigns that must preserve conceptual behavior. Do not use for a one-off known fix or microbenchmark-only tuning.
---

# Profile-guided optimization

Improve the real workload without silently changing what the algorithm means. Let the profiler choose where to look, use a whole-workload benchmark to decide what stays, and keep each experiment small enough to accept or discard on its own.

## Establish the contract

Before editing:

1. Read the repository instructions and trace the algorithm, mutation boundaries, caches, checkpoints, and output-quality calculation end to end.
2. Inspect the branch, worktree, dependency lock state, existing tests, benchmark entry points, production execution path, and available profilers.
3. Define the observable behavior. Record what must remain exact, what may be conceptually equivalent, and how result quality is ordered.
4. Record the performance metric, representative input, fixed-work benchmark, sustained-run duration, and frozen baseline revision.
5. Choose the operating mode:
   - **Interactive** is the default. Propose one increment, implement it uncommitted, present the evidence, and commit only after user sign-off.
   - **Unattended campaign** requires explicit authority to keep working and separate authority for commits, pushes, PR edits, or other external changes. When authorized, read [references/unattended-campaign.md](references/unattended-campaign.md) before starting.

Do not pull, rebase, switch branches, upgrade dependencies, or change the measurement environment merely to begin. Surface repository-state choices that would invalidate the baseline.

## Build the oracles first

Use the smallest set of checks that covers the real risk:

- Make the primary performance oracle run complete algorithm iterations on a representative input. Do not replace it with fine-grained microbenchmarks. Add a kernel benchmark only when that kernel is itself the product boundary or when it helps explain an end-to-end result.
- Make stochastic work reproducible with a fixed seed. Keep setup, parsing, reporting, and fixture construction outside the timed section unless they belong to the requested performance boundary.
- Prevent the measured result from being optimized away.
- Add or strengthen independent debug assertions for implicit structural, cache, and index invariants. Recompute expected state from authoritative data, not from the derived state being checked.
- Prefer one short debug run on a small full-system fixture over broad fine-grained test growth. Remove temporary development tests when the durable debug oracle and integration run cover the risk.
- Keep a fixed-iteration optimized run for trajectory, output, and quality comparison.
- Keep a fixed-time production-path run for sustained throughput, stability, and final quality.

Establish these checks and record the baseline before claiming an improvement.

## Use the profiler visibility ladder

Profile the latest accepted revision with the normal optimized workload. Keep fixed work for comparable traces and isolate the computation phase from setup and reporting where possible.

1. Build with optimization, symbols, and frame pointers when the profiler needs them.
2. Inspect inclusive stacks first to locate the broad phase that owns time. Then inspect leaf samples and allocator, clone, drop, hashing, comparison, and memory-move descendants.
3. Treat a large opaque inlined block as a visibility problem. In Rust, add a temporary `#[inline(never)]` to the broad suspected phase, rebuild, and profile again.
4. If that frame remains broad, move one layer inward. Add temporary boundaries to only its likely children, profile again, and repeat until the trace distinguishes useful optimization surfaces.
5. Avoid annotating the whole call tree at once. Progressive boundaries preserve enough normal optimization to keep the trace representative and make each new frame explainable.
6. Stop narrowing when the profile answers where time goes. Record the target, its inclusive and leaf evidence, and the concrete mechanism to test.
7. Remove every profiling-only annotation, rebuild the normal optimized binary, and run the performance gate on that build. Use a search or diff check to prove no temporary boundary remains.

No-inline annotations change code generation. Their traces guide investigation only. Never use their sample percentages as throughput claims or benchmark the annotated build as the candidate.

For Rust on macOS, prefer a temporary profiling build that inherits release optimization, retains full debug symbols, disables stripping, and forces frame pointers. Use Instruments Time Profiler on a fixed-iteration run and inspect the solver worker separately from setup, monitoring, and reporting threads. Use allocation tracing only on a reduced workload when ordinary time samples implicate allocation, because full allocation tracing can distort the run.

## Run the discovery loop

Repeat from the latest accepted revision:

1. Use the profiler visibility ladder to choose one surface.
2. Form one narrow hypothesis and implement the smallest isolated experiment. Avoid bundled cleanup, speculative abstractions, and unrelated formatting.
3. Run the cheapest compile or all-target check that catches structural breakage.
4. Compare the candidate with its immediate parent using the short whole-workload benchmark.
5. If warm-up already shows a severe regression, stop collection, restore only the experiment's changes, and record the rejection. Do not spend time on sustained runs, full correctness checks, PR prose, or CI.
6. If the result is flat, reject added complexity. A simpler cleanup may stay only when behavior is verified, performance does not regress, and the rationale is explicit.
7. If the result is promising, move to the acceptance gate.

The profiler supplies hypotheses. It does not supply speedup claims. A high sample percentage can include inlined descendants or surrounding optimized work.

## Apply the acceptance gate

For a promising candidate:

1. Audit the complete diff, callers, semantics, invariants, and restoration paths.
2. Compare the fixed-seed, fixed-iteration optimized output with the immediate parent. Classify it as:
   - **Exact** when the normalized output and trajectory match.
   - **Equivalent** when ordering or randomness changes but the algorithm, eligibility, scoring, and quality objective remain conceptually unchanged. Explain the difference.
   - **Changed** when policy or quality semantics differ. Reject unless the user explicitly approved that change.
3. Run the short debug full-system fixture so expensive invariant checks execute.
4. Run the sustained production path and record throughput plus complete and incomplete result quality.
5. Run the repository's relevant tests, all-target build or check, formatting or diff check, and any required lint surface. Distinguish pre-existing failures from new ones.
6. Re-run the final parent and candidate comparison sequentially under a controlled environment when the fast gate overlapped other work or the result is small or surprising.
7. Keep the change only when the measured benefit or concrete simplification justifies its complexity.

In interactive mode, present the uncommitted diff and evidence now. Wait for sign-off before committing.

## Keep measurements comparable

Parent and candidate must use the same compiler, dependencies and lockfile, allocator, feature flags, build profile, benchmark code, fixture, seed, target directory, machine power state, and background-load conditions.

- Prefer sequential measurements in the same worktree and target directory. Detached worktrees can resolve different ignored lockfiles or dependencies.
- Do not reject or promote a candidate from a loaded machine. Record overlapping work and repeat final measurements in isolation.
- Compare each experiment with its immediate parent. Report cumulative progress separately against one frozen baseline revision.
- Use the benchmark framework's interval and significance result for adjacent claims. Treat a single sustained-run rate as a checkpoint, not a confidence interval.
- When a surprising result fails to reproduce, correct the report and ledger. Do not average incompatible sessions into a cleaner story.
- Re-measure candidates after rebasing or composing them with another accepted change. Gains can disappear when the parent changes.

## Preserve safety and readability

- Do not weaken validation, invariant checks, numerical semantics, or error handling for speed.
- Do not introduce unsafe code solely as an optimization unless the user explicitly authorizes that exact experiment.
- Prefer deleting work, moving rejection earlier, compacting hot representations, and removing allocation or indirection before adding caches or duplicate state.
- Treat allocation reuse as a hypothesis, not a fact. Retained capacity, free lists, larger working sets, and extra bookkeeping can be slower than fresh allocation.
- Keep new derived state only when one mutation boundary owns its updates and an independent oracle can detect drift.
- Reject a small gain when the permanent code obscures the algorithm or creates fragile invariants without a narrow owner.

## Close the loop

For every accepted or rejected experiment, preserve its exact parent, hypothesis, measurement, behavior result, decision, and lesson somewhere durable when the campaign is long enough that repetition is plausible.

When the user asks to wrap up, start no new exploration. Finish or reject candidates already in flight, remove profiling-only edits and temporary files, run final checks, reconcile the worktree and report, and summarize the accepted head, cumulative result, behavior status, and remaining hypotheses.
