# External agent survey workflow

Read [OpenAPI](openapi.yaml) for all 15 routes and [operations](operations.md) for complete mutation payloads. The [SurveyJS extension guide](surveyjs-guide.json) is the canonical reference for Collector-specific question behavior. This reference uses synthetic data only. The server URL in OpenAPI is deliberately nonfunctional: obtain the actual deployment URL and an issued bearer key through the operator's approved secure channel. Never extract/mint keys or use SQL/application internals as shortcuts.

## Authentication and errors

Send `Authorization: Bearer <API_KEY>` and `Accept: application/json`; writes also need `Content-Type: application/json`. Health only proves availability, not authentication. Prefer GET `/api/external/surveys/{survey}/overview` (survey UUID) to verify authorized access without downloading the full schema. GET `/api/external/surveys/{survey}/agent-context` returns the full context; resolve an alias with GET `/api/external/surveys/by-alias/{alias}/agent-context` when the UUID is unknown.

Every listed route requires the exact `editor` API-key role except GET `/api/external/surveys/{survey}/revisions/latest`, which requires a valid key but no specific role. The authenticated agent-context response includes `data.documentation.surveyjs_guide`; fetch that linked guide before using Collector-specific properties or renderer values. There is no implicit role hierarchy: an appAdmin/surveyAdmin key still needs editor on editor routes. Survey access requires the key's user to own the survey, or the key itself to have `appAdmin` or `surveyAdmin`; browser session identity never grants extra external access.

- 401: missing/malformed/invalid/revoked key, `{success:false,message}`.
- 403: missing required key role or survey access, `{success:false,message}`.
- 404: missing survey route binding (Laravel `{message}`); missing alias uses `{success:false,message:"Survey not found"}`.
- 422: Laravel envelope validation is `{message,errors:{field:["..."]}}`, **without** `success`; controller schema/operation failures use `{success:false,message}` and sometimes `errors` (finalize has `errors.activation`). Do not flatten these shapes.
- 409: mutation sequence conflict has `{success:false,message,errors:{expected_sequence:["..."],latest_sequence:4}}`; participant token conflict has `errors.error_code:"token_already_assigned_to_other_survey"`.
- 429: `{success:false,message:"Too Many Attempts."}`. All external routes share 120 requests/minute per validated key; invalid/missing credentials share 60/minute per IP. Respect `Retry-After` seconds, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and on 429 `X-RateLimit-Reset` (Unix seconds). Normal middleware responses include limit/remaining; framework-thrown errors need not carry them.

## SurveyJS extension guide

GET `/api/external/surveyjs-guide` returns the pinned, public-safe Collector guide in the standard `{success:true,message,data}` envelope. Add `?question_type=radiogroup` to retain the version/general sections while filtering `data.question_types` to that documented type. A 422 for an undocumented type means only that no guide entry exists; it does not mean the SurveyJS type itself is unsupported. The guide covers `autoAdvanceIf`, its 300 ms/native-pointer/sole-question limits, and the `renderAs: "cards"` package prerequisite. Documentation never proves that `core/radiogroup-cards` is installed, enabled or valid on the target deployment.

After fetching context, use the normal mutation sequence: dry-run → review `data.diff` and `data.validation` → apply the identical guarded request → read back. `update_question_props` is the supported way to add `autoAdvanceIf`; do not write survey content through SQL or application internals.

## Token-efficient reads and mutations

Prefer these authenticated reads for focused edits; all require `editor` and the same owner/explicit manager-key access as agent-context:

- GET `/api/external/surveys/{survey}/overview`: same survey metadata, latest_revision, publishability, workflow and documentation as agent-context, but **no snapshot**. Instead `data.outline` lists pages in schema order, each with name/title (when present) and top-level `elements` containing name/type/title only. Titles preserve all existing localized values; choices, descriptions, nested content and expressions are omitted. A panel is one top-level element, not an expanded list of its children. Empty surveys have an empty outline. Use `latest_revision.sequence`, or **0** when null.
- GET `/api/external/surveys/{survey}/pages?name=feedback`: returns `{survey_id,sequence,page}` in `data`; `page` is the complete effective page, including nested elements and page conditions.
- GET `/api/external/surveys/{survey}/questions?name=service_rating&page=feedback`: returns `{survey_id,sequence,page_name,question}` in `data`; `question` is the complete effective top-level question/element. `page` is optional for disambiguation. Nested questions are not separately addressable, matching mutation targeting; read their containing page (or panel element) instead.

Page/question names are exact and case-sensitive. URL-encode **query values**, including slashes, dots and spaces; do not interpolate names as URL path segments. Required `name` and optional nonempty `page` must be strings: invalid queries return Laravel 422 `{message,errors}`. Missing targets return controller 404 `{success:false,message}`; ambiguous matches return controller 422, never an arbitrary match. Malformed page/element structure can return controller 422. Missing survey remains Laravel 404. Reads never mutate surveys or create revisions. Page/question `sequence` identifies the effective snapshot read, with **0** when no revision exists.

For POST `/mutations`, send **`"include_snapshot": false`** in both dry-run and apply requests. This omits only `data.survey.snapshot`; metadata, actor, validation, diff and revision/sequence fields remain. Omitted or true preserves the legacy full response. Use JSON booleans (not the strings `"true"`/`"false"`); null is invalid. Stored revisions and validation still use the complete schema. `data.diff` is a summary of targets/changed fields, **not old/new property values**: review it alongside your request and the content you read. Use the default full dry-run response if you need to inspect the complete candidate.

Recommended small-edit flow:
1. Fetch overview to find the target, supported operations and revision sequence.
2. Read the question and, if its context matters, its page. If the returned sequence differs from the overview or earlier reads, refetch/reconcile before composing a change; do not combine snapshots from different revisions.
3. Send a named-target dry-run with `expected_sequence`, a short `note`, and `include_snapshot:false`; review request, diff and validation.
4. Apply the same request/sequence with only `dry_run` changed to false.
5. Read the affected question/page back and compare its sequence to the applied sequence. A later sequence means another edit may have intervened; reconcile before claiming your version is still current. For deletions, verify absence plus the current overview sequence.

These reads are not dependency discovery. For cross-page expressions, triggers, renames, moves/deletions or broad structural edits, inspect the relevant other pages or the full agent-context before changing anything. Overview deliberately omits root settings and logic. On 409 or timeout, refetch and reconcile; do not blindly retry. Client-side filtering before model/tool output remains useful for large pages; never silently truncate the fields needed to verify an edit.

## 1. Create or resume

Only create when requested. POST `/api/external/surveys` creates a draft (201), owned by the key's user. Alias is unique, alias/title <=255; language codes exactly two characters, default included in supported list. Optional JSON schema defaults to empty pages, access_type defaults public (or closed); status can only be draft. Optional page_template_package_id is a deployment-valid enabled page-package UUID, otherwise omit/null for system default. Import is not publishability validation.

```json
{"alias":"example-feedback","title":"Example feedback","supported_languages":["en","et","ru"],"default_language":"en","access_type":"public"}
```

A timeout is not proof of failure. Resolve the intended alias and inspect authenticated context before retrying creation. Never blindly invent a new alias to bypass a conflict.

## 2. Context → dry-run → review → guarded apply → readback

Full agent-context gives `data.survey`, effective `snapshot`, nullable `latest_revision`, `publishability`, and operation/workflow links. Compact overview substitutes `outline` for `snapshot` (see the token-efficient flow above). Use its supported operations. If latest_revision is null (and workflow expected_sequence is null), use **0**, not an unguarded null. Otherwise use latest_revision.sequence.

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
