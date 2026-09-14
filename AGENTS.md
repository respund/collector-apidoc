# Collector External API — agents

Read [OpenAPI](openapi.yaml), the [workflow](workflow.md), and the [12 mutation operations](operations.md). Use HTTP only, never application internals or database access. Maintain generated files through the application's offline generator, not manual edits; review before publication.

Obtain the deployment URL and issued editor bearer key securely from the operator. Health is not authentication: fetch authenticated agent-context. Confirm the user's scope, preview with the current sequence, review the diff, apply with the same sequence, and read back. Reconcile conflicts and timeouts before retrying. Never finalize, delete content or change access without explicit approval. Never print or publish credentials or real records.
