---
name: handoff-review
description: Use only when the user explicitly asks for an AI-agent handoff or plan review, including handoff-review to Claude/Codex, reviewer prompts, external review ingestion, feedback triage, or plan revision before implementation. Produces `.agent-handoffs/` artifacts, preserves provenance, and requires reviewers to verify repository claims directly.
---

# Handoff Review

Use this skill only for an explicit plan-review loop. Do not apply it silently to ordinary planning, coding, or code review.

The loop is:

1. Create a handoff package.
2. Create a reviewer prompt.
3. Optionally run or prepare a reviewer agent.
4. Ingest the review.
5. Triage feedback critically.
6. Report a concise digest.
7. Revise the plan only after the triage is clear.

## Artifact Folder

Write artifacts to a repo-local folder that is ignored by Git:

1. Find the Git root with `git rev-parse --show-toplevel`.
2. If there is no Git root, use the current working directory.
3. Create `.agent-handoffs/` there.
4. Ensure `.agent-handoffs/.gitignore` exists with exactly:

```gitignore
*
```

Do not use OS temp folders, home-directory scratch paths, or committed `docs/` folders unless the user explicitly asks.

Name files with a stable topic slug:

```text
.agent-handoffs/YYYY-MM-DD-topic-handoff.md
.agent-handoffs/YYYY-MM-DD-topic-review-request.md
.agent-handoffs/YYYY-MM-DD-topic-review-response.md
.agent-handoffs/YYYY-MM-DD-topic-review-triage.md
.agent-handoffs/YYYY-MM-DD-topic-revised-plan.md
```

## Roles

Use model-agnostic roles:

- **Originating agent**: prepares the current plan for review.
- **Reviewer agent**: independently inspects the repository and critiques the plan.
- **Integrating agent**: evaluates reviewer feedback and revises the plan.
- **User**: decides ambiguous, scope-changing, or product-intent questions.

The same model may play different roles in different loops.

## Handoff Package

Use `assets/handoff-template.md` as the structure. Keep the handoff concise but complete enough for a fresh reviewer to start.

Include:

- Goal and non-goals.
- Repository context: files inspected, commands run, current branch/state if relevant.
- Current implementation plan.
- Provenance ledger.
- Assumptions and open questions.
- Known risks and test strategy.

Mark each important claim with provenance:

- `observed`: verified directly from repository files, command output, tests, docs, or issue/PR metadata.
- `user`: stated or approved by the user.
- `inference`: reasoned from available evidence but not directly verified.
- `external`: supplied by another agent or external reviewer.
- `unverified`: not yet checked.

## Reviewer Prompt

Use `assets/review-request-template.md` as the structure.

The reviewer prompt must instruct the reviewer agent to:

- Treat the handoff as a map, not ground truth.
- Inspect the repository directly before accepting factual claims.
- Check assumptions, file references, implementation order, risks, tests, and simpler alternatives.
- Identify which findings are blocking versus optional.
- Write its Markdown review directly to the named review response artifact when it has file-write access.
- Return the exact Markdown review in chat/stdout if it cannot write the artifact.

Always save the exact prompt. Prefer having the reviewer agent create `.agent-handoffs/YYYY-MM-DD-topic-review-response.md` itself; if the reviewer can only return text, save that returned text unchanged as the response artifact.

## Reviewer CLI Permission

If the user explicitly names an external reviewer target, treat that as permission to hand the review package to that target. Examples include:

- "handoff-review to Claude"
- "handoff review to Claude"
- "handoff-review to Codex"
- "handoff review to Codex"
- "ask Claude to review this plan"

Do not ask for a second approval just because the reviewer runs outside the local sandbox or may receive repository context. Record the external handoff in the provenance ledger and use the named CLI only when it is available and authenticated. If sandboxed auth is unavailable but the normal terminal is authenticated, run the named reviewer CLI outside the sandbox. If no CLI is requested or available, provide the prompt for the user to paste into the reviewer agent.

Ask before handing off only when the requested target is ambiguous, the handoff scope is unclear, or the package appears to contain secrets or unrelated sensitive files.

## Review Ingestion

When ingesting a reviewer response, do not accept feedback wholesale. Create a triage artifact using `assets/triage-template.md`.

At the top of the triage artifact, include an artifact links section with Markdown links to:

- Handoff package.
- Review request.
- Review response.
- Revised plan, when present.

Use the concrete artifact filenames for the current loop. Prefer clickable local links when the environment supports them; otherwise use relative Markdown links.

Classify every material reviewer finding:

- `accept`: confirmed or strongly supported; update the plan.
- `reject`: incorrect, irrelevant, over-scoped, or contradicted by repository evidence.
- `verify`: plausible but needs code inspection, test execution, source confirmation, or a smaller spike.
- `ask-user`: changes scope, behavior, architecture, timeline, product intent, migration strategy, public API, or risk tolerance.

For each classification, include a one-sentence rationale and provenance.

## Digest Gate

After ingesting a review, report back to the user before implementation. Keep the chat digest short and action-oriented.

Start the chat digest with a clickable Markdown link to the triage artifact. If useful, include links to the review request, review response, handoff package, and revised plan immediately after it. Put links before the summary so the user can open the details instantly.

Use this shape:

```text
Review digest:
- Triage: [review triage](...)
- Request: [review request](...)
- Response: [review response](...)
- Bottom line: ...
- Counts: accepted N, rejected N, verify N, ask-user N.
- Biggest material issue: ...
- Plan impact: ...
- Needs your decision: ...
```

If there are no `ask-user` items and no major unresolved `verify` items, recommend the next action in one sentence. If there are `ask-user` items, ask the smallest number of questions needed before revising or implementing.

Do not paste large tables into chat unless the user asks; keep detailed tables in the triage artifact.

## Revised Plan

After triage:

- Preserve the original handoff and reviewer response unchanged.
- Write a revised plan artifact when accepted or verified feedback changes the plan.
- Record rejected suggestions with brief rationale so they are not re-litigated.
- Explicitly list remaining verification tasks and user decisions.

Do not start writing substantial implementation code until the digest gate is complete and any `ask-user` blockers are resolved.
