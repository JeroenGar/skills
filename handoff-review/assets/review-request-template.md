# Independent Plan Review Request

You are the reviewer agent. Treat the handoff as a map, not ground truth.

## Instructions

1. Inspect the repository directly before accepting factual claims.
2. Verify referenced files, assumptions, implementation order, risks, and tests.
3. Look for simpler approaches, missed edge cases, and incorrect assumptions.
4. Customize this review for the handoff context: check domain-specific risks, installation paths, public API claims, migration concerns, security boundaries, or other task-specific issues when relevant.
5. Separate blocking findings from optional improvements.
6. Keep the review concise and evidence-backed.

## Response Artifact

Write the full Markdown review to:

```text
.agent-handoffs/YYYY-MM-DD-topic-review-response.md
```

Replace the path above with the concrete response artifact path before sending the request.

If you cannot write files, return the exact Markdown review in your response so the originating agent can save it unchanged.

## Response Format

| Finding | Severity | Evidence | Recommendation |
|---|---|---|---|

## Blocking Issues

## Optional Improvements

## Questions for the Originating Agent or User
