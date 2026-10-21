# Store listing pitfalls

Read when a listing, media or age-rating step fails, or before choosing locales for Google Play.

## Contents

- Field limits
- Google Play language codes
- Which version states accept what
- Media rules
- Failure table

## Field limits

Counted in characters after trimming; newlines and spaces count. `swiss-knife:listing_get` returns the same table with its source.

| Field             | iOS                                  | Android                                                                  |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------ |
| title             | 30 (min 2)                           | 30 (the API accepts 50 and Play shortens it silently; we hold you to 30) |
| subtitle          | 30                                   | not present                                                              |
| short description | not present                          | 80                                                                       |
| full description  | 4000                                 | 4000                                                                     |
| keywords          | 100, commas count, spaces are wasted | not present                                                              |
| what's new        | 4000                                 | 500                                                                      |
| promotional text  | 170                                  | not present                                                              |
| video URL         | not present                          | a YouTube URL                                                            |

Push targets carry fixed fields: "app_info" = title, subtitle, privacy URL; `version` = description, keywords, what's new, promotional text, marketing and support URL; `play_listing` = title, short and full description, video URL; `play_release_notes` = what's new per `version_code`.

## Google Play language codes

Play spells languages its own way. Use these exactly as written for `locale` in `swiss-knife:listing_update`, `swiss-knife:listing_push`, `swiss-knife:media_upload` and the rest:

`af am ar az-AZ be bg bn-BD ca cs-CZ da-DK de-DE el-GR en-AU en-CA en-GB en-IN en-SG en-US en-ZA es-419 es-ES es-US et eu-ES fa fa-AE fa-AF fa-IR fi-FI fil fr-CA fr-FR gl-ES gu hi-IN hr hu-HU hy-AM id is-IS it-IT iw-IL ja-JP ka-GE kk km-KH kn-IN ko-KR ky-KG lo-LA lt lv mk-MK ml-IN mn-MN mr-IN ms ms-MY my-MM ne-NP nl-NL no-NO pa pl-PL pt-BR pt-PT rm ro ru-RU si-LK sk sl sq sr sv-SE sw ta-IN te-IN th tr-TR uk ur vi zh-CN zh-HK zh-TW zu`

Common traps: `es` has three Play codes (`es-ES` Spain, `es-419` Latin America, `es-US`), so ask or take the market the user names; `ru` is `ru-RU`; Hebrew is `iw-IL`, not `he`; `pt` is `pt-BR` or `pt-PT`; `zh` is `zh-CN`, `zh-TW` or `zh-HK`; Norwegian is `no-NO`. Apple's codes differ (`ru`, `es-MX`), so never reuse an iOS locale list for Android.

A language the store already lists stays updatable under its existing code. A NEW language is created automatically by `swiss-knife:listing_push` target `play_listing`, but only under a valid Play code; the dry run flags anything else as `unsupported_locale` and names the code to use.

## Which version states accept what

| Apple version state                                                                     | Listing text (`version`)                                                                       | Screenshots                                                   |
| --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| PREPARE_FOR_SUBMISSION, DEVELOPER_REJECTED, REJECTED, METADATA_REJECTED, INVALID_BINARY | allowed                                                                                        | allowed                                                       |
| WAITING_FOR_REVIEW                                                                      | needs an approval code only on a restricted legacy connection (it changes what reviewers read) | needs an approval code only on a restricted legacy connection |
| IN_REVIEW                                                                               | refused                                                                                        | refused                                                       |
| live (READY_FOR_SALE)                                                                   | only promotional text                                                                          | refused: frozen until the next version                        |

"app_info" (name, subtitle, privacy URL) stays editable while a version is live, but is gated the same way while a submission waits for review. Google Play has no per-version listing and no version-state gate on listing or image pushes.

## Media rules

- Apple's enum names are not the marketing sizes: `APP_IPHONE_67` is today's 6.9 inch iPhone. Never guess a `size_spec`; read `swiss-knife:media_list` or the `swiss-knife:media_validate` message.
- PNG or JPEG only. Every Apple screenshot forbids an alpha channel. Play's `icon` is the only asset that requires one.
- Play: images 320 to 3840 px, long side at most twice the short side, at most 15 MiB. Icon, feature graphic and two screenshots are required for the default language. Promo graphics and video previews are not pushed.
- Replacing a locale's screenshots removes what the store shows now. `swiss-knife:media_remove` is local until the next `swiss-knife:media_push`, which then deletes it in the store.
- `swiss-knife:media_upload` commit: a master goes at the device class's master `size_spec`; other sizes are `override`.

## Failure table

Format: error code or situation -> what to do.

| Situation                                                                          | What to do                                                                                                                                                                                                                                               |
| ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `unsupported_locale`, `play_unsupported_language`, `play_listing_not_found`        | Wrong or unknown Play code. Fix the locale per the list above, redraft, dry run again. Never retry the same code.                                                                                                                                        |
| "listing_version_not_editable", "media_version_not_editable" (needs_manual_action) | Version left an editable state. No retry. See the state table; offer the next version or waiting.                                                                                                                                                        |
| "listing_push_conflict", status `conflict`                                         | Compare the store and draft; keep the requested draft with exact `conflict_overrides`, or discard it to keep the store. Preview again.                                                                                                                   |
| "listing_field_rejected", `play_invalid_listing` (fatal)                           | Store refused the text. Shorten or fix, push again.                                                                                                                                                                                                      |
| "media_asset_rejected", `play_image_rejected`, `play_image_too_large` (fatal)      | Image size, format, alpha or bytes refused. Fix per `media_validate`, upload again.                                                                                                                                                                      |
| "age_rating_answer_rejected" (fatal)                                               | Apple refused an answer. Check allowed values in `age_rating_get` (two frequency scales exist), answer again.                                                                                                                                            |
| "age_rating_push_conflict"                                                         | Compare the current store answer with the user’s factual answer; use exact `conflict_overrides` to retain the requested draft, or discard it to retain the store. Read `age_rating_get` and preview again. Ask only for facts the user has not supplied. |
| "age_rating_state_invalid"                                                         | Something else is in the wrong state; a human looks. No retry.                                                                                                                                                                                           |
| `confirm: true` refused for unanswered required questions                          | `completeness.missing` in the dry run lists them; answer with `age_rating_update`.                                                                                                                                                                       |
| Actual approval refusal on a restricted connection                                 | `swiss-knife:approval_request` for the push tool, relay, wait, repeat with `approval`.                                                                                                                                                                   |
| `pending: true` for a long time                                                    | Keep polling the status tool; retryable store errors (rate limit, 503) retry on their own up to a limit.                                                                                                                                                 |
| Play edit busy                                                                     | An upload or promote holds the app's one edit; wait a minute and push again.                                                                                                                                                                             |
| `listing_translate` or `listing_draft_whats_new` reports no AI key                 | Write the text yourself and save it with `listing_update`.                                                                                                                                                                                               |
