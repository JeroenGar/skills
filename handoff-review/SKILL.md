---
name: handoff-review
description: Use only when the user explicitly asks for an AI-agent handoff or plan review, including handoff-review to Claude/Codex, reviewer instructions, external review ingestion, feedback triage, or plan revision before implementation. Produces `.agent-handoffs/` artifacts, preserves provenance, and requires reviewers to verify repository claims directly.
---

# Handoff Review

Use this skill only for an explicit plan-review loop. Do not apply it silently to ordinary planning, coding, or code review.

The loop is:

1. Create one handoff package with reviewer instructions included.
2. Give the user exact interactive commands to run Claude Code and Codex.
3. Ingest the review response artifacts.
4. Triage feedback critically.
5. Report a concise digest.
6. Revise the plan only after the triage is clear.

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
.agent-handoffs/YYYY-MM-DD-topic-claude-response.md
.agent-handoffs/YYYY-MM-DD-topic-codex-response.md
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
- Reviewer instructions and response artifact paths for Claude Code and Codex.

Mark each important claim with provenance:

- `observed`: verified directly from repository files, command output, tests, docs, or issue/PR metadata.
- `user`: stated or approved by the user.
- `inference`: reasoned from available evidence but not directly verified.
- `external`: supplied by another agent or external reviewer.
- `unverified`: not yet checked.

## Reviewer Instructions

Put reviewer instructions directly in the handoff package. Do not create separate review request files by default. The handoff must instruct each reviewer agent to:

- Treat the handoff as a map, not ground truth.
- Inspect the repository directly before accepting factual claims.
- Check assumptions, file references, implementation order, risks, tests, internal contradictions, security concerns, and simpler alternatives.
- Identify which findings are blocking versus optional.
- Write only to its named review response artifact unless the user explicitly approves another edit.
- Write its Markdown review directly to its named response artifact when it has file-write access.
- Stay interactive and ask the user in the terminal if anything is unclear.
- Return the exact Markdown review in chat/stdout only if it cannot write the artifact.

Prefer having each reviewer agent create its named response artifact itself. If a reviewer can only return text, tell the user to save that returned text unchanged as that reviewer's response artifact.

## User-Run Reviewer Commands

Do not run external reviewer CLIs yourself by default. After creating the handoff artifact, give the user copy-pasteable interactive commands so the user can watch progress, answer questions, and own any external data transfer, authentication, and local permission prompts.

Always use interactive reviewer mode. Do not use `claude -p`, `claude --print`, `codex exec`, `codex review`, or other fully automated/non-interactive reviewer commands unless the user explicitly asks for automation.

Always include both Claude Code and Codex command templates. Do not ask the user which reviewer target they want first; the user can decide which command, or both commands, to run. Replace placeholders with concrete paths for the current loop. For the default two-reviewer loop, use separate reviewer-specific artifacts:

For Claude Code, use `--permission-mode auto` so file inspection and test commands can proceed with fewer prompts while still avoiding full permission bypass.

```bash
REPO_ROOT="<repo-root>"
HANDOFF="$REPO_ROOT/.agent-handoffs/YYYY-MM-DD-topic-handoff.md"
RESPONSE="$REPO_ROOT/.agent-handoffs/YYYY-MM-DD-topic-claude-response.md"
cd "$REPO_ROOT"
claude --permission-mode auto "Read the handoff at $HANDOFF. Act as the Claude reviewer. Write the Markdown review to $RESPONSE."
test -s "$RESPONSE" && echo "Claude response written: $RESPONSE" || echo "Claude response missing or empty. If Claude printed Markdown instead, save it unchanged to: $RESPONSE"
```

```bash
REPO_ROOT="<repo-root>"
HANDOFF="$REPO_ROOT/.agent-handoffs/YYYY-MM-DD-topic-handoff.md"
RESPONSE="$REPO_ROOT/.agent-handoffs/YYYY-MM-DD-topic-codex-response.md"
cd "$REPO_ROOT"
codex -C "$REPO_ROOT" --sandbox workspace-write --ask-for-approval on-request "Read the handoff at $HANDOFF. Act as the Codex reviewer. Write the Markdown review to $RESPONSE."
test -s "$RESPONSE" && echo "Codex response written: $RESPONSE" || echo "Codex response missing or empty. If Codex printed Markdown instead, save it unchanged to: $RESPONSE"
```

Also include the exact response artifact path or paths and tell the user to ask you to ingest the response after any command succeeds. If a reviewer cannot write the artifact, tell the user to ask the reviewer to print the exact Markdown review and save that text unchanged at that reviewer's response artifact path.

Only run a reviewer CLI yourself when the user separately and explicitly asks you to execute that command, the environment policy allows it, and the command does not require unsafe permission bypass flags. Never add `--dangerously-bypass-approvals-and-sandbox`, `--dangerously-skip-permissions`, or similar bypass flags to reviewer commands unless the user explicitly asks and policy permits it.

Ask before preparing commands only when the handoff scope is unclear or the package appears to contain secrets or unrelated sensitive files.

## Review Ingestion

When ingesting a reviewer response, do not accept feedback wholesale. Create a triage artifact using `assets/triage-template.md`.

At the top of the triage artifact, include an artifact links section with Markdown links to:

- Handoff package.
- Review response, or reviewer-specific responses when there are multiple.
- Revised plan, when present.

Use the concrete artifact filenames for the current loop. Prefer clickable local links when the environment supports them; otherwise use relative Markdown links.

When ingesting multiple reviewer responses, preserve reviewer provenance. Classify every material finding from each reviewer, merge duplicate findings only when the underlying claim is the same, and record which reviewer or reviewers raised it.

Classify every material reviewer finding:

- `accept`: confirmed or strongly supported; update the plan.
- `reject`: incorrect, irrelevant, over-scoped, or contradicted by repository evidence.
- `verify`: plausible but needs code inspection, test execution, source confirmation, or a smaller spike.
- `ask-user`: changes scope, behavior, architecture, timeline, product intent, migration strategy, public API, or risk tolerance.

For each classification, include a one-sentence rationale and provenance.

## Digest Gate

After ingesting a review, report back to the user before implementation. Keep the chat digest short and action-oriented.

Start the chat digest with a clickable Markdown link to the triage artifact. If useful, include links to the review response, handoff package, and revised plan immediately after it. Put links before the summary so the user can open the details instantly.

Use this shape:

```text
Review digest:
- Triage: [review triage](...)
- Response: [review response](...)
- Bottom line: ...
- Counts: accepted N, rejected N, verify N, ask-user N.
- Biggest material issue: ...
- Plan impact: ...
- Needs your decision: ...
```

For multi-reviewer loops, use `Responses:` with one link per reviewer instead of a singular response link.

If there are no `ask-user` items and no major unresolved `verify` items, recommend the next action in one sentence. If there are `ask-user` items, ask the smallest number of questions needed before revising or implementing.

Do not paste large tables into chat unless the user asks; keep detailed tables in the triage artifact.

## Revised Plan

After triage:

- Preserve the original handoff and reviewer response unchanged.
- Write a revised plan artifact when accepted or verified feedback changes the plan.
- Record rejected suggestions with brief rationale so they are not re-litigated.
- Explicitly list remaining verification tasks and user decisions.

Do not start writing substantial implementation code until the digest gate is complete and any `ask-user` blockers are resolved.
