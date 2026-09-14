# External agent survey workflow

Read [OpenAPI](openapi.yaml) for all 11 routes and [operations](operations.md) for complete mutation payloads. This reference uses synthetic data only. The server URL in OpenAPI is deliberately nonfunctional: obtain the actual deployment URL and an issued bearer key through the operator's approved secure channel. Never extract/mint keys or use SQL/application internals as shortcuts.

## Authentication and errors

Send `Authorization: Bearer <API_KEY>` and `Accept: application/json`; writes also need `Content-Type: application/json`. Health only proves availability, not authentication. Verify authorized access with GET `/api/external/surveys/{survey}/agent-context` (survey UUID) or GET `/api/external/surveys/by-alias/{alias}/agent-context`.

Every listed route requires the exact `editor` API-key role except GET `/api/external/surveys/{survey}/revisions/latest`, which requires a valid key but no specific role. There is no implicit role hierarchy: an appAdmin/surveyAdmin key still needs editor on editor routes. Survey access requires the key's user to own the survey, or the key itself to have `appAdmin` or `surveyAdmin`; browser session identity never grants extra external access.

- 401: missing/malformed/invalid/revoked key, `{success:false,message}`.
- 403: missing required key role or survey access, `{success:false,message}`.
- 404: missing survey route binding (Laravel `{message}`); missing alias uses `{success:false,message:"Survey not found"}`.
- 422: Laravel envelope validation is `{message,errors:{field:["..."]}}`, **without** `success`; controller schema/operation failures use `{success:false,message}` and sometimes `errors` (finalize has `errors.activation`). Do not flatten these shapes.
- 409: mutation sequence conflict has `{success:false,message,errors:{expected_sequence:["..."],latest_sequence:4}}`; participant token conflict has `errors.error_code:"token_already_assigned_to_other_survey"`.
- 429: `{success:false,message:"Too Many Attempts."}`. All external routes share 120 requests/minute per validated key; invalid/missing credentials share 60/minute per IP. Respect `Retry-After` seconds, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and on 429 `X-RateLimit-Reset` (Unix seconds). Normal middleware responses include limit/remaining; framework-thrown errors need not carry them.

## 1. Create or resume

Only create when requested. POST `/api/external/surveys` creates a draft (201), owned by the key's user. Alias is unique, alias/title <=255; language codes exactly two characters, default included in supported list. Optional JSON schema defaults to empty pages, access_type defaults public (or closed); status can only be draft. Optional page_template_package_id is a deployment-valid enabled page-package UUID, otherwise omit/null for system default. Import is not publishability validation.

```json
{"alias":"example-feedback","title":"Example feedback","supported_languages":["en","et","ru"],"default_language":"en","access_type":"public"}
```

A timeout is not proof of failure. Resolve the intended alias and inspect authenticated context before retrying creation. Never blindly invent a new alias to bypass a conflict.

## 2. Context → dry-run → review → guarded apply → readback

Context gives `data.survey`, effective `snapshot`, nullable `latest_revision`, `publishability`, and operation/workflow links. Use its supported operations. If latest_revision is null (and workflow expected_sequence is null), use **0**, not an unguarded null. Otherwise use latest_revision.sequence.

The following three independent request examples illustrate a named page followed by two different questions. Start with a survey supporting EN/ET/RU. Fetch the current sequence before each step. Examples show a new survey's expected apply progression 0 → 1 → 2 → 3; dry-runs alone do not advance it.

POST `/api/external/surveys/{survey}/mutations`:

```json
{"operation":"create_page","target":{"page":"feedback"},"payload":{"page":{"name":"feedback","title":{"en":"Feedback","et":"Tagasiside","ru":"Обратная связь"},"elements":[]}},"dry_run":true,"expected_sequence":0}
```

Review `data.validation`, `data.diff` and candidate `data.survey.snapshot`. Preview has `dry_run:true`, null revision_id/sequence and current `latest_sequence`. Apply the identical approved request with `dry_run:false` and the **same expected_sequence**. A successful apply returns a revision_id, advanced sequence/latest_sequence and dry_run:false. GET context to verify the actual saved page before continuing.

```json
{"operation":"insert_question","target":{"page":"feedback"},"payload":{"question":{"name":"service_rating","type":"radiogroup","title":{"en":"How was the service?","et":"Kuidas hindate teenindust?","ru":"Как вы оцениваете обслуживание?"},"choices":[{"value":"good","text":{"en":"Good","et":"Hea","ru":"Хорошо"}},{"value":"poor","text":{"en":"Poor","et":"Halb","ru":"Плохо"}}]}},"dry_run":true,"expected_sequence":1}
```

Repeat review → guarded apply → readback. The second question uses SurveyJS's native radiogroup plus the cards renderer. **Prerequisite:** the deployment's `core/radiogroup-cards` question package must be installed, enabled and valid. Ask the operator to confirm it; there is no External API package-discovery/enable endpoint. Do not silently drop renderAs or claim the package exists.

```json
{"operation":"insert_question","target":{"page":"feedback","after_question":"service_rating"},"payload":{"question":{"name":"visit_again","type":"radiogroup","renderAs":"cards","title":{"en":"Would you visit again?","et":"Kas külastaksite uuesti?","ru":"Вы посетили бы нас снова?"},"choices":[{"value":"yes","text":{"en":"Yes","et":"Jah","ru":"Да"}},{"value":"no","text":{"en":"No","et":"Ei","ru":"Нет"}}]}},"dry_run":true,"expected_sequence":2}
```

On 409, refetch context and reconcile, then preview again; do not force overwrite. On timeout, inspect context/latest revision and intended diff before resending: creation/insertion is not idempotent. Mutation preview does not persist; apply does. No revision-list or rollback route is exposed externally.

## 3. Translate with exported keys

GET `/api/external/surveys/{survey}/translations/source?source_language=en`; optional nullable source_language is exactly two characters and defaults to the survey default. The bundle contains `survey` language metadata and `items` with key/entity/field/page, optional question/choice_index/collection/entry_index, and source_text. Only exported nonempty fields are included; do not manufacture paths. Preserve `items[].key` unchanged even if it looks like a path.

After confirming that this exact synthetic key was exported, POST `/api/external/surveys/{survey}/translations`:

```json
{"source_language":"en","supported_languages":["en","et","ru"],"default_language":"en","translations":[{"key":"pages.feedback.questions.service_rating.title","translations":{"ru":"Как вы оцениваете обслуживание?"}}],"note":"Reviewed Russian translation"}
```

Source/default languages and nonempty supported_languages/translations are required, codes exactly two characters; each key and nonempty translation map of nonempty strings are required. Note is nullable <=255. Unknown keys fail 422. Existing language entries are preserved except replacements; supported languages are augmented with source/default and translation-map languages. Response includes revision_id, sequence, source, snapshot, survey, validation, diff, actor and translated_items_count. **Translations have no dry-run/expected-sequence protection**: coordinate writers, review before sending, and read back/reconcile timeouts.

## 4. Finalize only on explicit instruction

POST `/api/external/surveys/{survey}/finalize` with `{ "note": "Approved for activation" }` (optional nullable string <=255). It validates publishability, creates a durable publish revision and sets status active, returning `{success:true,message,data:{survey,revision}}`. Failed activation returns 422 with errors.activation. It has **no mutation-style dry-run/sequence guard**; context publishability is advisory, not a lock. Never finalize merely because editing succeeded.

GET `/api/external/surveys/{survey}/revisions/latest` returns `{success:true,message,data:null}` if none, otherwise a revision object: revision_id, sequence, source, actor, survey{id,snapshot}, validation:null, diff, is_durable, nullable note, created_at. It is readback, not revision history or rollback.

## 5. Participants (only if requested)

POST `/api/external/surveys/{survey}/participants`:

```json
{"token":"synthetic-participant-001","language":"en","email":"participant@example.invalid","name":"Example Participant","max_responses":1,"expires_at":"2030-01-01T00:00:00Z"}
```

Token is required string <=255 (globally unique); language required <=10. Email nullable email <=255, name nullable <=255, expires_at nullable date, max_responses nullable integer >=1 (default 1). New registration returns 201 with data.participant (id, survey_id, token, language, email, name, status, max_responses, expires_at) and already_registered:false. Same-survey replay returns 200/already_registered:true **without updating existing details**. Other-survey token returns 409. Treat tokens as sensitive in actual operation; examples here are synthetic.

POST `/api/external/surveys/{survey}/participants/bulk` accepts `{ "participants": [{ "token": "synthetic-participant-001", "language": "en" }] }`, nonempty array, same per-item fields; duplicate tokens in the payload fail 422. A valid bulk request returns 200 with total/created/already_registered/failed and participants results. Each success has token/success/already_registered; conflicts have token/success:false/error_code/message. It is not all-or-nothing: inspect every item, not just HTTP 200.

GET `/api/external/surveys/{survey}/participants/{token}/exists` returns `{success:true,message:"Success",data:{registered:true}}` (or false). URL-encode path values; tokens containing a slash cannot be represented as ordinary single route segments. Readback is scoped to the requested survey.
