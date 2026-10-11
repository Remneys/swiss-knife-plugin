# Android pitfalls

Read when a prerequisite, preflight, build or upload step fails.

## Contents

- Failure table
- version_code_already_used
- play_upload_key_mismatch
- Play language codes
- Reading human_action

## Failure table

Format: error code or situation -> what to do.

| Situation                                                                                                                                                      | What to do                                                                                                                                                                               |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `version_code_already_used` (fatal)                                                                                                                            | New build with a fresh number. See below.                                                                                                                                                |
| `play_upload_key_mismatch` (fatal)                                                                                                                             | Stop, human action. See below.                                                                                                                                                           |
| `preflight_upload_key_mismatch` (our own check, nothing sent)                                                                                                  | The stored keystore is not the key Play has registered. Fix the keystore or the recorded fingerprint, then upload again.                                                                 |
| `play_release_in_review`                                                                                                                                       | Google refused the commit and cancelled nothing. Wait for the review, then upload again. `replace_in_review: true` cancels the review in progress: only with the user's explicit say-so. |
| `play_policy_declaration_incomplete`, `play_financial_features_declaration_missing`, `play_health_declaration_missing`, `play_app_content_declaration_missing` | The user completes it in Play Console (App content). Relay the `human_action`; once done, `swiss-knife:preflight_declaration_acknowledge` records it, then upload again.                 |
| `draft_app_requires_draft_release` (fatal)                                                                                                                     | An app never published accepts only draft releases; tell the user, this is Play's rule.                                                                                                  |
| `human_action` in any answer                                                                                                                                   | Relay verbatim, STOP, wait, then call its `verify_with` tool.                                                                                                                            |
| Refusal that needs an approval code                                                                                                                            | `swiss-knife:approval_request`, relay, wait for the human's code, repeat the same call with `approval`.                                                                                  |
| Service account cannot see the app                                                                                                                             | Relay the `human_action` from `swiss-knife:ship_prerequisites`; only a person can grant access in Play Console.                                                                          |
| No `swiss-build.config.*` at the ref, or no `android` section                                                                                                  | Add it to the repo as a plain object. Never import the package or tell anyone to `npm install` it.                                                                                       |
| Private repo, clone or ref check fails                                                                                                                         | Save a `git_token`; see references/credentials.md.                                                                                                                                       |
| `play_edit_conflict`, `play_edit_expired`, Play server error                                                                                                   | Retryable; the upload retries by itself. Check `swiss-knife:submission_upload_status`; do not start a second upload.                                                                     |

## version_code_already_used

A versionCode is immutable in Play: retrying sends the same bytes and fails the same way. The control plane assigns the number to each build and passes it to Gradle as `versionCode` / `versionName` properties.

1. Start a new `swiss-knife:build_start` (platform `android`); it gets a fresh number. `swiss-knife:build_retry` also takes a fresh one if the failed build was a build failure.
2. If the new build fails the same way, the project is ignoring the assigned number: the Gradle file hardcodes `versionCode`. Show the user the line and have it read the `versionCode` Gradle property instead, then rebuild.
3. For a bundle built outside Swiss-Knife: `swiss-knife:artifact_register` refuses a number already on record; raise it in the project and rebuild.
4. Confirm with `swiss-knife:submission_upload_status`; `assignedBuildNumber` may differ from the number baked into the bundle.

## play_upload_key_mismatch

Play rejected the bundle because it was signed with a different key than the upload key registered for this package. Retrying cannot help.

1. Do not generate or save a new keystore on a guess, and do not overwrite the stored one.
2. Ask the user which keystore Play expects. Either they hold it (save it with `swiss-knife:build_credentials_save`, kind `android_keystore`; an overwrite needs an approval code), or they reset the upload key in Play Console (App integrity, App signing) registering the certificate from `swiss-knife:build_android_keystore_certificate`.
3. After they confirm, record what Play shows with `swiss-knife:build_expected_upload_cert_set`, run `swiss-knife:preflight_android` (`keystore_matches_expected` must pass), rebuild with `swiss-knife:build_start`, then upload.

## Play language codes

Play's language codes are its own and are not Apple's: Russian is `ru-RU` (Apple `ru`), Hebrew `iw-IL`, Latin American Spanish `es-419`. When any listing or release-note step names a language, use Play's code; a push that would create a language under a code Play does not publish in is refused before anything is sent.

## Reading human_action

Shape: `{ who, what, url, why, verify_with: { tool, args?, note } }`. Show who, what, url and why unchanged. Do not drop the url. After the user confirms, call `verify_with.tool` once; if it still shows the block, relay it again and stop.
