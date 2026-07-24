---
name: jeroens-rust
description: Apply Jeroen's preferences for designing, implementing, refactoring, and reviewing Rust software with a focus on correctness, robustness, readability, and performance. Use for general Rust development, including application code, ETL and data processing, optimization algorithms, heuristic or exact search, numerical or geometric kernels, specialized data structures, incremental computation, profiling, and performance regression work.
---

# Jeroen's Rust

Design Rust in Jeroen's preferred style: make correctness assumptions and invariants explicit, fail loudly on broken internal state, preserve readable domain-oriented dataflow, keep abstractions narrow, and optimize measured bottlenecks without compromising robustness. Apply this style to ordinary application code, ETL pipelines, and performance-critical algorithms alike.

When preferences compete, preserve correctness and explicit semantics first. Prefer readable, maintainable dataflow by default; let measured performance override style locally only when correctness remains independently verifiable.

## Work In This Order

1. Trace the data, control, and mutation flow end to end.
2. Define the observable result and the invariants that make it correct.
3. Implement the naive behavior that is easiest to inspect and verify.
4. Check it on small fixtures and edge cases. When performance matters, profile an optimized build on the most representative workload it can handle.
5. Design the production path for known input sizes and resource limits without optimizing for hypothetical scale.
6. Before implementing a smarter bottleneck, introduce its narrow concrete abstraction and required API with the naive behavior still behind it. Verify that this refactor does not change behavior.
7. Replace only the internals with the advanced data structure or algorithm. Keep the naive behavior as a bounded reference implementation or debug check.
8. Recheck the same correctness boundary and workload; keep the optimization only when its measured benefit justifies the additional state and invariants.

Treat state and indexes required by the chosen algorithm, dataflow, or known resource bounds as part of the baseline. Use profiling to gate optional caches, specialized indexes, parallelism, and low-level tuning.

## Separate Representations By Responsibility

Use distinct types for distinct responsibilities:

- Keep external or deserialized records at the trust boundary.
- Validate and normalize boundary data as it enters the system. Materialize bounded inputs into immutable, precomputed internal data; process large or streaming inputs incrementally instead of collecting the whole dataset.
- Keep mutable algorithm or pipeline state in the object that owns its invariants and caches.
- Represent checkpoints and results as immutable values containing only what is needed to compare, restore, or report.

Do not carry strings, nested boundary structures, or repeated metadata through hot code when compact internal values are sufficient.

Share large immutable structures when independent states or workers need the same data. Clone mutable state at coarse speculative boundaries when that makes ownership and rollback simpler; do not clone it inside the evaluation loop.

## Choose Data Layout From Operations

Prefer contiguous storage and direct indexing when identity is stable and the key space is dense. Use typed indices when confusing two index domains would be a correctness bug.

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
struct StateIdx(usize);

fn value(values: &[f64], index: StateIdx) -> f64 {
    values[index.0]
}
```

Choose identity separately from physical storage. When insertions and removals must not invalidate references, use stable handles. If hot numeric work also needs dense rows, map stable handles to dense indices and keep the hot values in vectors.

Exploit known structure behind a small API:

- Flatten dense multidimensional data when it improves locality.
- Store only one triangle of symmetric pair data.
- Use arrays for genuinely fixed arity.
- Use unordered removal when order has no meaning.

Keep index arithmetic in one place and assert its bounds, symmetry, or round trips in debug builds.

Choose sorted vectors, maps, sets, or specialized structures from the measured mix of lookup, traversal, insertion, and removal. Prefer the standard library until a different representation proves useful.

Wrap a standard collection in a small concrete type once it carries a domain-specific representation invariant, such as sorted order, uniqueness, partitioning, or synchronization with an index. Keep the storage private and expose only the minimal constructors, read-only views, and mutation methods that preserve the invariant; do not expose raw mutable access. Validate the invariant at construction and with debug assertions after mutation.

Keep frequently accessed fields compact. Split hot and cold data only when measurement shows that carrying the cold fields through the kernel matters.

## Prefer Local Computation Over Memory Traffic

Treat memory traffic, working-set size, and access predictability as first-class algorithm costs. A handful of simple arithmetic operations on data already in registers or cache is often cheaper than a cache miss, pointer chase, or load from a large derived table.

Favor compact working sets, contiguous storage, traversal in storage order, and batching that reuses nearby data. Avoid pointer-rich or scattered structures in hot paths unless their operations justify the locality cost.

Recompute cheap derived values from nearby authoritative data when storing them would add another memory stream, indirection, or invalidation logic. This reduces derived state and improves correctness as well as locality.

Store or precompute values when their computation is genuinely expensive, they are reused enough to amortize the memory access, or they can be kept compactly beside the data that consumes them. Compare recomputation against lookup on a representative optimized workload rather than assuming a cache is faster.

## Keep Algorithm Boundaries Small

Make each phase expose the smallest meaningful input and output contract. Keep orchestration focused on the sequence of phases, not their internal decisions.

Keep temporary concepts local to the phase that needs them. Promote a type or abstraction only after it has a second real use or forms a valuable correctness boundary.

Split a file when unrelated responsibilities make the algorithm hard to follow, not because it crossed an arbitrary line count.

Let each phase own concise diagnostics. Report summaries at phase boundaries and keep per-element logging out of hot loops.

## Readability & Documentation

Make the code explain itself first through structure, domain names, and small semantic checkpoints. Add comments only when important intent or a contract still cannot be derived from the code. Before commenting confusing code, first check whether clearer names or structure can remove the need for the comment.

Optimize for human scan review. A reviewer should be able to identify the model, assumptions, phase boundaries, state transitions, and policy choices without simulating every expression. Make context-dependent decisions visible through names, types, checkpoints, and concise comments: mechanically correct code can still be conceptually wrong because it was built from missing context or a false assumption.

### Name Semantic Checkpoints

Name a meaningful intermediate computation when the name identifies a stage in the algorithm, even if the value is consumed only once. Use these semantic checkpoints to make a sequence of transformations read in domain terms without explanatory comments.

Use a block initializer when deriving the checkpoint requires local setup, branching, or mutable scratch state. Return only the stage value from the block so the mechanics stay scoped behind its name.

```rust
let normalized_candidates = {
    let scale = normalization_scale(&candidates);
    candidates
        .into_iter()
        .map(move |candidate| candidate.normalized(scale))
};

select_best(normalized_candidates)
```

Keep iterator checkpoints lazy unless later operations require materialization. Place a checkpoint near its consumer, choose a name from the algorithm's vocabulary, and skip vague names such as `tmp`, `data`, or `result`. Do not add a binding that merely restates an already obvious expression.

Use a semantic checkpoint especially when a `match` would otherwise contain a query, iterator chain, or other non-trivial derivation. Name the value being classified, then let the `match` focus only on its possible outcomes.

```rust
let existing_entry = entries
    .iter_mut()
    .find(|entry| entry.key == incoming.key);

match existing_entry {
    Some(entry) => entry.merge(incoming),
    None => entries.push(incoming),
}
```

### Comment Intent And Contracts

Add a concise comment above a function when its name and signature do not fully communicate its purpose or contract. State what the function guarantees, assumes, or does differently from an obvious alternative. Mention preconditions, early-termination behavior, important side effects, or equivalence with a reference implementation when these matter. Do not restate parameters or narrate the implementation.

Inside complex logic, use short single-line comments as guideposts for algorithm stages or non-obvious policy decisions. Do not comment each statement. Prefer a semantic checkpoint with a block initializer when several mechanical steps exist only to produce one meaningful value:

```rust
/// Restarts the search from a solution in `pool`.
/// `pool` must be non-empty and sorted by increasing loss.
fn restart(search: &mut Search, pool: &[Candidate], rng: &mut impl Rng) {
    // Bias selection toward better solutions at the front of the pool.
    let selected_solution = {
        let sample = distribution.sample(rng).abs().min(0.999);
        let selected_index = (sample * pool.len() as f32) as usize;

        &pool[selected_index].solution
    };

    search.restore(selected_solution);
}
```

Keep comments short and close to the code they explain. Update or remove them when the contract or algorithm changes; a stale comment is worse than no comment.

## Express Dataflow Functionally

Prefer iterator pipelines for transformations, queries, and reductions. Use combinators whose names expose the operation—`map`, `filter` or `filter_map`, `flat_map`, `chain`, and `zip`—and finish with terminals such as `any`, `all`, `find` or `find_map`, `min_by_key`, `sum`, `fold`, or `collect`. Choose these over manual accumulators, flags, or nested loops when the pipeline reads directly from input to result.

Use `itertools` when it gives the domain operation a name. Prefer `tuple_combinations` or `combinations` for pairwise search, `unique` or `unique_by` for uniqueness, `sorted_by_cached_key` for expensive ranking keys, `collect_vec` for an explicit `Vec`, and `izip!` for several aligned sequences. Validate equal lengths before `zip` or `izip!` when truncation would hide malformed input.

Keep closures short and expression-oriented. Pass a function or method directly when a closure only forwards its arguments. Use `filter_map`, `find_map`, and `then_some` when absence is part of the transformation instead of filtering and then unwrapping.

Return `impl Iterator` from reusable traversals to keep the abstraction clean without allocating an intermediate `Vec`. Let callers compose, short-circuit, or collect only when their use case requires ownership. Materialize deliberately when sorting, shuffling, indexing, reusing results, transferring ownership, or collecting a snapshot before mutating the source. Do not collect only to resume iteration.

Use a normal loop when mutation, rollback, early exit, borrow coordination, or several evolving accumulators are the algorithmic story. Do not conceal stateful control flow or allocation behind a clever iterator chain, and do not replace a clear pipeline with a manual loop without measured reason.

## Put Abstractions Around Policy

Keep core state concrete. Introduce a trait when a real policy varies, such as evaluation, sampling, filtering, termination, or progress reporting.

Use generics or `impl Trait` for policies called from hot code so calls are statically dispatched and monomorphized. Use dynamic dispatch only when runtime heterogeneity is required and its cost is irrelevant or measured.

Separate the mechanism that explores possibilities from the mechanism that evaluates one possibility. Let the explorer own traversal and refinement; let the evaluator own the expensive domain calculation. Pass the current bound into the evaluator when it can stop once the result is known to be uncompetitive.

Keep control-plane hooks outside the algorithmic state. Reporting and cancellation should observe or interrupt work without owning the state being optimized.

## Isolate Experimental Semantics

Separate eligibility, ordering, and application:

- Eligibility decides whether an operation is allowed.
- Ordering chooses among allowed operations.
- Application performs the chosen operation and restores invariants.

Keep non-trivial or heuristic ordering in a named key, score, or comparison function, especially when it is likely to change through experimentation. Use a direct closure for an obvious stable order.

Model semantic outcomes before numeric quality. If outcomes such as valid, invalid, complete, or partial have different meanings, encode them explicitly and define their precedence. Compare numeric quality only within compatible outcomes; do not bury domain precedence in sentinel values or an unexplained weighted scalar.

Represent a proposed operation faithfully. If it changes several parts of the state, preserve every effect for validation and application even when ranking reduces it to a smaller key.

Use deterministic tie-breakers in tests, benchmarks, and seeded runs. Treat randomized tie-breaking as an explicit search policy.

## Control Mutation

Expose immutable inspection freely. Route mutation through narrow methods that leave the state consistent before returning.

Keep mutable borrows short: inspect first, decide, then mutate. Prefer validating an operation before mutation when that is cheap and exact.

Use restoration strategies according to frequency:

- Use apply/undo for tight trial operations only when undo restores every changed field bit-exactly.
- Record overwritten values or use a compact snapshot for floating-point, lossy, cache-heavy, phase, branch, or independent-worker updates.
- Use differential restoration when rebuilding expensive unchanged structures would dominate.

Treat caches, counters, and indexes as derived state. Keep one authoritative source of truth, update derived state at the same mutation boundary, and provide a slow recomputation for debug checks.

```rust
state.apply(change);
debug_assert_eq!(state.cached_metric(), recompute_metric(&state));
```

Run cheap local assertions after public mutations. Gate whole-state recomputation behind debug assertions or an explicit validation mode; sequence-level checks catch drift that field-local assertions miss.

## Shape The Hot Path

Avoid allocation, cloning, formatting, and temporary hash-table construction inside tight loops.

Allocate reusable buffers outside the loop, reserve realistic capacity, and clear or overwrite them between evaluations. When repeatedly deriving a value from the same baseline, update a buffer from the immutable baseline instead of cumulatively modifying the previous result; this also avoids numerical drift.

Order work from cheap and likely to reject toward expensive and exact. Use bounds and summaries to fail fast, and stop as soon as the caller's question is answered. Provide separate “find any” and “collect all” paths when the first can return early.

Extract a helper when it is reused, hides genuine complexity, or creates a valuable test boundary. Avoid helpers that only rename one expression.

Add parallelism only after identifying independent work and measuring a CPU bottleneck. Give workers independent mutable state, share immutable input, and combine results through an explicit deterministic rule when reproducibility matters.

## Make Evolution Compile-Visible

When a method consumes most fields of a struct, such as parser or converter logic, destructure it exhaustively without `..`. Bind used fields and mark intentionally ignored fields with `field: _`.

```rust
let Input {
    data,
    parameters,
    metadata: _,
} = input;
```

Adding or renaming a field then causes a compile error at the broad consumer. Treat that error as a prompt to map, validate, or explicitly ignore the change.

Use direct field access when a method needs only a few fields; exhaustive destructuring should create useful change detection, not noise.

Prefer an exhaustive `match` when several variants or mutually exclusive shapes of one value drive control flow. Use `if let` only when one pattern matters and every other case intentionally has the same behavior. Avoid stacking `if` or `if let` statements for mutually exclusive cases; one `match` makes exclusivity and future variants compile-visible.

Keep ordinary `if` for independent predicates and guard clauses. Avoid wildcard match arms unless every unlisted variant intentionally has identical behavior.

## Make Numerical Semantics Explicit

Validate numeric input at construction or parsing boundaries. Encode meaningful restrictions in types where doing so removes repeated checks or prevents invalid state.

Keep exact and tolerant operations separate. Define a named comparison policy for each quantity that needs approximation: its accepted finite domain, absolute or relative tolerance formula, and which side of a boundary it favors. Use exact comparison when no such policy is defined.

Use a float wrapper or explicit comparison only where a total order is required. Decide how NaN and infinity are handled before ordering.

Use monotonic proxies such as squared distance when they avoid expensive work without changing the decision. Preserve the exact calculation when the actual value is required.

## Validate Inputs And Fail Loudly On Broken State

Validate every relied-on shape, range, cardinality, ordering, and cross-field relationship of external data once, as it enters the internal representation. For streaming input, validate each record or chunk before downstream stages rely on it.

Return errors at trust boundaries such as parsing, files, network input, and public APIs.

Assert internal invariants where mutation or transformation could break them. Panic rather than turn an impossible internal state into a plausible fallback result.

Use warnings only for explicitly tolerated defects whose continuation semantics are defined.

Keep expensive redundant invariant checks in `debug_assert!`. Run a separate optimized build with debug assertions to exercise them on realistic workloads; profile and benchmark without those checks unless they are part of the production configuration.

## Specialize Only Verified Kernels

Keep a clear general implementation until profiling identifies a kernel worth specializing. Limit custom traversal, specialized layout, forced inlining, or unsafe code to the measured kernel.

Retain a small, obviously correct implementation for bounded test inputs as the oracle. Do not keep duplicate production paths unless runtime cross-checking is required; compare specialized results using exact or explicitly tolerant equality.

Use unsafe code only when a safe implementation cannot meet a measured need. Keep the unsafe region small, document the invariant that makes it sound, and leave a check that would fail if the surrounding assumptions drift.

## Measure Behavior

Run computation-heavy workloads with optimization enabled. Use debug builds for fast compile feedback, not production-scale performance conclusions.

Profile the optimized binary. Preserve symbols when the profiler needs them:

```bash
CARGO_PROFILE_RELEASE_DEBUG=1 cargo build --release
```

Use deterministic seeds, fixed workloads, and recorded configuration for regression measurements. Keep benchmark setup outside the timed kernel and prevent the measured result from being optimized away.

Measure both the isolated kernel and the end-to-end workload. Separate query, update, and mixed behavior when they stress different paths.

For stochastic algorithms, compare distributions of runtime and result quality across repeated runs. Do not trust one favorable outcome.

After every optimization, compare the same correctness boundary and workload before and after. Keep a change only when the measured benefit justifies its complexity.

## Review Checklist

Before handing off Rust work, verify the applicable points:

- Naive behavior is verified, placed behind the stable abstraction before optimization, and retained as a reference check.
- A realistic optimized workload has been run.
- Representations have clear responsibilities and lifetimes.
- Data layout matches measured access and update patterns.
- Collections with domain-specific invariants are encapsulated behind APIs that preserve them.
- Hot paths favor compact working sets, predictable access, and cheap recomputation over scattered derived state.
- Hot loops reuse storage and avoid accidental work.
- Cheap rejection precedes expensive exact work.
- Experimental policies are isolated from mechanism.
- Long computations expose meaningful algorithm stages through named checkpoints.
- Collection transformations use semantic iterator combinators and materialize only at explicit boundaries.
- Mutation has one owner and restores invariants.
- Derived state is checked against authoritative state.
- Broad consumers and enum mappings are exhaustive where evolution matters.
- Numerical approximation and edge-case bias are explicit.
- Specialized code is checked against a general implementation.
- Benchmarks are reproducible and measure meaningful behavior.
