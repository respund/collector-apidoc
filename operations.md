# Generic survey mutations

POST `/api/external/surveys/{survey}/mutations`; see [workflow](workflow.md) and [OpenAPI](openapi.yaml).
All examples are independent previews against a synthetic survey at sequence 3, not a batch to run blindly. Replace the sequence with the current effective sequence (0 if no revision). After preview and user-approved diff, send the same request with `dry_run:false`; read back before the next operation.

Envelope: required `operation` (one of the twelve below), required nonempty `target` object; optional nullable `payload` object, `dry_run` boolean (default false), `expected_sequence` integer >= 0, `note` string <= 255. Never omit the guard in agent writes. Use `{"survey":true}` when the operation consumes no target; `{}` and `[]` fail Laravel's required rule. Invalid envelopes return Laravel 422 `{message,errors}`; invalid operations/snapshots return controller 422 `{success:false,message}`. Missing/ambiguous named references are 422, not HTTP resource 404. A stale guard returns 409; refetch rather than overwrite.

Collector validates the whole resulting snapshot: pages/elements must be arrays, supported question types need names and choice-based questions need nonempty choices. Use unique names and complete type-specific SurveyJS properties (including rows/columns where needed); Collector is not a complete SurveyJS-property validator, and duplicate-name rejection depends on the operation. Operations target top-level page elements, not a general nested JSON patch. Question renderers must be enabled and valid on the deployment. No mutation creates or enables a template package.

## create_page

Requires a new nonempty page name. Appends; `target.page` identifies intent, `payload.page.name` supplies the name. Existing page or duplicate question names are rejected. Elements default to empty; move separately for placement.

```json
{"operation":"create_page","target":{"page":"feedback"},"payload":{"page":{"name":"feedback","elements":[]}},"dry_run":true,"expected_sequence":3}
```

## upsert_page

Creates a missing page at the end or shallow-merges an existing page. `target.page` overrides payload name. **Supplying `elements` replaces the complete array**, not an incremental merge: obtain approval before removing questions. Example assumes feedback exists and preserves its elements by omitting them.

```json
{"operation":"upsert_page","target":{"page":"feedback"},"payload":{"page":{"name":"feedback","title":{"en":"Feedback","et":"Tagasiside","ru":"Обратная связь"}}},"dry_run":true,"expected_sequence":3}
```

## move_page

Requires existing feedback and summary pages. Supply `payload.before_page` OR `after_page`; both or self-anchors are rejected. Without anchors, append. Missing anchors fail.

```json
{"operation":"move_page","target":{"page":"feedback"},"payload":{"before_page":"summary"},"dry_run":true,"expected_sequence":3}
```

## delete_page

Requires existing feedback. Destructive: removes the page and every contained question. A nonempty page requires explicit `confirm_non_empty:true` and user approval; empty pages need no flag.

```json
{"operation":"delete_page","target":{"page":"feedback"},"payload":{"confirm_non_empty":true},"dry_run":true,"expected_sequence":3}
```

## insert_question

Requires existing feedback and a globally unused question name. Appends unless `target.before_question` OR `after_question` references an existing question on that page. Both anchors fail. Supply a complete SurveyJS question, including nonempty choices for radiogroup.

```json
{"operation":"insert_question","target":{"page":"feedback"},"payload":{"question":{"name":"service_rating","type":"radiogroup","title":{"en":"Service","et":"Teenindus","ru":"Обслуживание"},"choices":["good","poor"]}},"dry_run":true,"expected_sequence":3}
```

## upsert_question

Requires destination feedback. `target.question`, if supplied, overrides payload name. **Full question replacement**, not property merge; omitted properties disappear. A matching question on another page is moved; duplicate matches fail. Same-page replacement retains position without anchors; new/cross-page inserts append unless one target before/after anchor is given. Obtain approval for replacement/removal effects.

```json
{"operation":"upsert_question","target":{"page":"feedback","question":"service_rating"},"payload":{"question":{"name":"service_rating","type":"radiogroup","title":{"en":"Service","et":"Teenindus","ru":"Обслуживание"},"choices":["good","poor"],"isRequired":true}},"dry_run":true,"expected_sequence":3}
```

## update_question_props

Requires an existing question; optional page disambiguates. Nonempty `payload.props` shallowly sets properties. Cannot include `name` (no rename); nested maps/arrays are replaced, not merged. The service also accepts properties directly in payload; prefer explicit props.

```json
{"operation":"update_question_props","target":{"page":"feedback","question":"service_rating"},"payload":{"props":{"isRequired":true}},"dry_run":true,"expected_sequence":3}
```

## move_question

Requires existing service_rating on feedback and destination summary. Source page is optional when unambiguous. `payload.to_page` names destination; payload before/after anchors refer to destination elements. Omitted anchors append; both or self-anchors fail. Contents are preserved.

```json
{"operation":"move_question","target":{"page":"feedback","question":"service_rating"},"payload":{"to_page":"summary"},"dry_run":true,"expected_sequence":3}
```

## delete_question

Requires an existing unambiguous named question, optionally scoped by page. Destructively removes the whole question; no confirmation field is enforced by the endpoint, so obtain explicit user approval before apply.

```json
{"operation":"delete_question","target":{"page":"feedback","question":"service_rating"},"dry_run":true,"expected_sequence":3}
```

## set_survey_languages

Nonempty supported list and an included default are required, all lowercase two-letter codes. Updates survey language metadata, deduplicating supported languages; does not translate text. Target is a nonempty survey marker.

```json
{"operation":"set_survey_languages","target":{"survey":true},"payload":{"supported_languages":["en","et","ru"],"default_language":"en"},"dry_run":true,"expected_sequence":3}
```

## set_survey_settings

Nonempty settings shallowly set SurveyJS root properties. Cannot replace `pages`; this is not a way to change database status/access/title. The service also accepts settings directly in payload; prefer explicit settings. Nested values replace existing values.

```json
{"operation":"set_survey_settings","target":{"survey":true},"payload":{"settings":{"showProgressBar":"top"}},"dry_run":true,"expected_sequence":3}
```

## set_localized_text

Requires nonempty field/language and a text value. Optional question/page selects the field; no question/page means SurveyJS root field. Missing named targets fail; absent fields are created as maps. Scalar text converts using the survey's default language, preserving its old text; `payload.default_language` may explicitly override conversion language. Existing map entries remain except the language being set. Use supported two-letter languages and string text.

```json
{"operation":"set_localized_text","target":{"page":"feedback","question":"service_rating","field":"title"},"payload":{"language":"ru","text":"Как вы оцениваете обслуживание?"},"dry_run":true,"expected_sequence":3}
```
