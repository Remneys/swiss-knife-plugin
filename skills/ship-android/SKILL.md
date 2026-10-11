---
name: ship-android
description: Takes an Android app from a connected repo to a bundle on the Google Play internal testing track through the Swiss-Knife MCP server (service account, upload key, preflight, build, upload, store check). Use when the user says "ship my Android app to Play internal testing", "upload my AAB to Google Play", "build the Android release", or a Play upload fails. Not for iOS (use ship-ios), store listings, production rollout, in-app products, or SDK setup (use setup-sdk).
---

# Ship an Android build to Play internal testing

Success: `swiss-knife:submission_upload_status` (platform `android`) says `uploaded`, meaning Google Play holds the bundle, and its `assignedBuildNumber` is the versionCode Google assigned. Our build status alone is not proof.

## Rules that never bend

- Any tool answer with `human_action` (or a refusal that needs an approval): relay `who`, `what`, `url`, `why` to the user verbatim, then STOP. No retry, no polling, no workaround. When they say it is done, call the `verify_with` tool.
- Approval codes come from a signed-in human. Call `swiss-knife:approval_request` with the gated tool and your arguments, relay what it says, wait for the code, and send the same call again with it as `approval`. Never invent or reuse a code.
- Never generate a new upload key for an app Play already has one for, and never overwrite a stored keystore without an approval code.
- Always `swiss-knife:ship_prerequisites` and `swiss-knife:build_preflight` before `swiss-knife:build_start`.
- Do not tell anyone to `npm install` swiss-build: it is not on npm. Its config is a plain object (see Step 3).

## Checklist

Copy and tick:

```
- [ ] 1. Workspace and app found (adopt, never duplicate)
- [ ] 2. Play service account and upload keystore stored and valid
- [ ] 3. Expected upload-key fingerprint recorded
- [ ] 4. Repo connected, config present, private repo readable
- [ ] 5. ship_prerequisites ready: true, preflight_android clean
- [ ] 6. build_preflight has no blocking issue
- [ ] 7. Build succeeded
- [ ] 8. Uploaded to the internal track
- [ ] 9. Verified against Google Play
```

## Steps

1. **Find the app.** `swiss-knife:workspace_whoami`, then `swiss-knife:app_list`. Reuse the app whose package matches the project; keep its `app_project_id`.
2. **Credentials.** `swiss-knife:build_credentials_list`. Save the `play_service_account` (kind, `serviceAccountJson`) and, if the user has the upload key Play accepted, the `android_keystore`, with `swiss-knife:build_credentials_save`. Only for a NEW app with no upload key: `swiss-knife:build_android_keystore_generate` (needs an approval code). Then `swiss-knife:build_credentials_validate` for each kind. Recipes: read references/credentials.md.
3. **Fingerprint.** Record what Play expects with `swiss-knife:build_expected_upload_cert_set` so a wrong key is caught before upload. If you do not know it, ask the user to read it in Play Console (App integrity); read references/credentials.md.
4. **Repo.** `swiss-knife:build_repo_status`; if needed `swiss-knife:build_repo_connect` (https URL). Private repo: `git_token`, see references/credentials.md. The repo needs a `swiss-build.config.ts` default-exporting a plain object with `app.androidPackageName`. The Gradle project must read the `versionCode` / `versionName` properties the build assigns, or the number will not apply.
5. **Prerequisites.** `swiss-knife:ship_prerequisites` with target `play_internal`; fix every `failed` item, re-run until `ready` is true. `not_verified` items (Play declarations, draft-only app) are things no API can show; tell the user. Then `swiss-knife:preflight_android` (add `probe_store: true` to ask Google Play itself).
6. **Preflight.** `swiss-knife:build_preflight` with platform `android`; fix each `blocking` issue.
7. **Build.** `swiss-knife:build_start` with platform `android` (pass `version` on the first build of an app). Keep `ids.jobId`; poll `swiss-knife:build_get` until `succeeded` or `failed`. On `failed` read `swiss-knife:build_logs`. It is not idempotent: before calling it again check `swiss-knife:release_list`, or pass `cancel_previous: true`.
8. **Upload.** `swiss-knife:submission_upload_build` with `release_id` and `platform: "android"` (internal track only). Then `swiss-knife:submission_upload_status` (platform `android`) until `uploaded`. `blocked` carries a `human_action`: relay and stop. `failed` carries `failureReason`: see the table in references/pitfalls.md.
9. **Verify.** Final read from the store side: `swiss-knife:submission_upload_status` with platform `android` must say `uploaded` with an `assignedBuildNumber`; add `swiss-knife:release_overview` for the release. If it does not, say so; never report success from the build status.

## When it breaks

One line each; the full table is in references/pitfalls.md (read when any step fails).

- `version_code_already_used` -> final, not retryable: Play never accepts a versionCode twice. Start a new `swiss-knife:build_start` (fresh number); if the Gradle file hardcodes `versionCode`, make it read the assigned property first.
- `play_upload_key_mismatch` -> the bundle was signed with a different key than Play registered. Stop; human action: save the right keystore, or reset the upload key in Play Console. Never generate a new key for this app.
- Private repo, clone fails -> `git_token` (fine-grained, read-only Contents).
- Release in review on Play (`play_release_in_review`) -> wait, or the user chooses `replace_in_review`; that cancels the review, so it needs their say-so.
