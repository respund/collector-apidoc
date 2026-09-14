# Collector External API — agent guide

## Scope and current status

Use Collector through its HTTP API, not database access or application internals. This documentation repository is being established: the generated endpoint and payload specification is not available yet. Do not invent missing contracts.

## Connection

- Obtain the deployment base URL and an issued bearer API key from the operator through an approved secure channel.
- External endpoints use the `/api/external` prefix.
- Send `Authorization: Bearer <API_KEY>` and `Accept: application/json`; send JSON writes with `Content-Type: application/json`.
- Editing requires an `editor` key and authorization for the target survey.
- A successful `/api/health` request only establishes service availability, not authenticated access. Verify access using `GET /api/external/surveys/{id}/agent-context`.
- Never print, commit or publish API keys. Do not create keys or change permissions without authorization.

## Safe survey changes

1. Confirm the target survey and the user's requested scope.
2. Fetch its `agent-context` to inspect the current snapshot, revision sequence and supported operations.
3. Use the documented payload for the chosen operation. If that contract is unavailable, ask for it rather than guess.
4. Submit `dry_run: true` with the current `expected_sequence`; inspect validation results and the diff.
5. Apply only the approved change through the mutations endpoint with the same expected revision. On HTTP 409, refetch and reconcile before retrying.
6. Read back and verify the result. If a write times out, inspect the current state before resending to avoid duplicates.
7. Do not finalize/activate a survey, delete existing content or change access settings unless explicitly requested.

## Documentation maintenance

Keep credentials, application source and real customer/respondent data out of this public repository. Use synthetic examples. Once generation is installed, update generated references through the application's generator, not by hand. Review documentation changes before publication.
