---
name: use-collector-api
description: Use Collector's authenticated External API to inspect or update SurveyJS surveys with the required-by-default, matrix-row, page-layout, and concise-history-note rules.
---

# Collector survey operations

Read `AGENTS.md`, `workflow.md`, `operations.md`, and `surveyjs-guide.json` before using the API. Use HTTP only and never expose credentials or real response records.

## Survey defaults

- Every actual question is required by default.
- Make a question optional only when the requirements give a specific reason; preserve and record that exception.
- For SurveyJS `matrix`, use both `isRequired: true` and `eachRowRequired: true`. `isRequired` alone allows one answered row, but every visible row must be answered here.
- For other matrix variants, use their row/cell-required settings and verify every visible row is validated.
- When uploading a new survey, put one question on each page by default. Keep clearly related questions together only when both content and technical/user-flow reasons make the shared page better.

## Revision history

Include a short `note` with every survey mutation. It should read like a compact git log entry: a few words describing what changed, not a narrative. Examples: `Lisa kriisikaardid`, `Matrix read kohustuslikuks`, `Luba automaatne edasi`.

## Safe mutation flow

1. Fetch authenticated `/api/external/surveys/{survey}/overview` for the compact page/question outline, scope, supported operations and revision sequence (0 when latest_revision is null). Resolve an unknown UUID through by-alias agent-context if necessary.
2. Read only the required `/pages?name=...` or `/questions?name=...&page=...`; URL-encode exact names. These return the complete selected object and its sequence, not a full snapshot. Targeting is top-level; for nested questions read the containing page. If sequences differ between reads, refetch/reconcile. Fetch full agent-context for root settings or cross-page dependencies; the outline is not a logic/dependency map.
3. Submit a `dry_run` with `expected_sequence`, the short `note`, and `include_snapshot:false` for a focused edit. Default/true returns the full candidate if that is needed.
4. Review the request, diff and validation result; preserve unrelated schema content. The diff is a changed-field summary, not old/new values.
5. Apply the identical guarded request with the same sequence and note (only change `dry_run` to false).
6. Read back the affected question/page and verify the requested result and applied sequence; a later sequence means another writer may have intervened. For deletion verify absence plus the overview sequence. Filter large HTTP responses before exposing tool output to the model, but never truncate required context silently.

On `409`, refetch and reconcile instead of overwriting. Never finalize or publish unless explicitly requested.
