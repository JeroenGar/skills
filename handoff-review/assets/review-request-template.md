# Independent Plan Review Request

You are the `{reviewer-name}` reviewer agent. Treat the handoff as a map, not ground truth.

## Handoff Package

Read this handoff package first:

```text
{absolute-path-to-handoff}
```

Then inspect the repository directly before accepting any factual claim.

## Instructions

1. Inspect the repository directly before accepting factual claims.
2. Verify referenced files, assumptions, implementation order, risks, and tests.
3. Look for simpler approaches, missed edge cases, internal contradictions, security concerns, and incorrect assumptions.
4. Customize this review for the handoff context: check domain-specific risks, installation paths, public API claims, migration concerns, security boundaries, or other task-specific issues when relevant.
5. Separate blocking findings from optional improvements.
6. Keep the review concise and evidence-backed.
7. Stay interactive: show progress in the terminal and ask the user if anything is unclear.
8. Write only to the response artifact path below unless the user explicitly approves another edit.

## Response Artifact

Write the full Markdown review to:

```text
{absolute-path-to-response}
```

Replace the placeholders above before sending the request. For multi-reviewer runs, each reviewer must get a distinct response path such as `.agent-handoffs/YYYY-MM-DD-topic-claude-response.md` or `.agent-handoffs/YYYY-MM-DD-topic-codex-response.md`.

If you cannot write files, tell the user before ending the session and print the exact Markdown review so it can be saved unchanged.

## Response Format

| Finding | Severity | Evidence | Recommendation |
|---|---|---|---|

## Blocking Issues

## Optional Improvements

## Questions for the Originating Agent or User
