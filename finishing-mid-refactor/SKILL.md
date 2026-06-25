---
name: finishing-mid-refactor
description: Use when the user asks Codex to finish an intentionally unfinished refactor after they edited code to show the desired direction, left notes, simplified APIs, or knowingly broke compilation/tests as a hint. Especially relevant for requests like "finish this rework", "complete my mid-refactor changes", or "I simplified some files; make it compile".
---

# Finishing Mid-Refactor

## Overview

The user's edits are design intent, not noise. Treat the broken tree as a partially applied refactor to complete, while preserving the user's simplification direction.

## Workflow

1. Inspect the current worktree before changing files.
   - Read `git status`, the relevant diff, and the files the user touched or has open.
   - Identify which changes are user-authored hints versus ordinary breakage.
   - Do not revert user edits unless explicitly asked.

2. State the intended rework in one or two sentences.
   - Name the boundary, invariant, or simplification the user appears to want.
   - If the direction is ambiguous or could change public behavior, ask the smallest useful question before editing.
   - If the direction is clear, proceed without a long plan.

3. Complete the refactor in focused passes.
   - Prefer the user's simpler shape over restoring the previous architecture.
   - Move responsibilities to the boundary implied by the user's edits.
   - Remove compatibility scaffolding, stale tests, and old call sites that contradict the new shape.
   - Keep changes narrow; do not add defensive abstractions, broad error handling, or configurability unless the user asked.

4. Repair the verification surface.
   - Update or add focused tests for the new invariant.
   - Run the narrowest useful check first, then broader checks once the tree compiles.
   - Use failure output to find stale assumptions, not as a reason to undo the user's direction.

5. Report the result by invariant.
   - Say what design boundary now holds.
   - List any remaining ambiguity or follow-up separately.
   - Include the verification commands that actually ran.

## Decision Rules

Ask before editing when:

- Multiple plausible designs fit the user's hints.
- The requested finish would alter product behavior, public APIs, data migrations, or persisted formats.
- The user left contradictory notes.
- Completing the refactor requires deleting substantial code whose replacement is not clear.

Proceed when:

- The user's notes and edits point to one coherent direction.
- Breakages are stale call sites, imports, types, or tests from the old shape.
- The work is local to the refactor boundary already touched.

## Common Mistakes

- Treating compile errors as evidence the user's edits are wrong.
- Reintroducing old indirection just to make tests pass quickly.
- Expanding scope into nearby cleanup that was not part of the refactor.
- Asking broad design questions when a specific invariant is already visible.
- Leaving the tree in a "mostly done" state after claiming the rework is finished.
