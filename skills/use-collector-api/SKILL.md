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

1. Fetch authenticated agent-context and confirm scope and the current revision sequence.
2. Submit a `dry_run` with `expected_sequence` and the short `note`.
3. Review the diff and validation result; preserve unrelated schema content.
4. Apply the identical guarded request with the same sequence and note.
5. Read back agent-context and verify the requested result.

On `409`, refetch and reconcile instead of overwriting. Never finalize or publish unless explicitly requested.
