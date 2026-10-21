# Android credentials

Read when a credential is missing or invalid, the fingerprint is unknown, or the repo is private.

## Contents

- Play service account
- Upload keystore
- Expected upload-key fingerprint
- Git token for a private repo
- Replacing credentials

## Play service account

Creating the service account and granting it access in Play Console is a human step (no API). If `swiss-knife:ship_prerequisites` says the app is not visible to it, relay its `human_action` and pause only work requiring that access. Once the user has the JSON key:

```
swiss-knife:build_credentials_save
  app_project_id, platform: "android", kind: "play_service_account",
  payload: { serviceAccountJson }
```

Read `validation` in the answer; `swiss-knife:build_credentials_validate` re-checks.

## Upload keystore

Two cases, never mixed:

- The app already has an upload key Play accepted: the user supplies that keystore. Save it with kind `android_keystore`, payload `{ keystoreBase64, keystorePassword, keyAlias, keyPassword }`.
- A brand-new app with no upload key: `swiss-knife:build_android_keystore_generate` (native write connections omit approval). Then `swiss-knife:build_android_keystore_certificate` returns the public certificate; the user registers it in Play Console (App integrity, App signing, upload key certificate; Play has no API for it) and tells you what fingerprint Play shows.

Replacing a stored upload key requires the user’s explicit intent. To generate a new key in place of a stored one, use `build_android_keystore_generate` with `replace: true`; importing the user’s existing keystore uses `build_credentials_save` and its payload, which has no replace flag. Native write connections omit approval; restricted legacy connections use the returned human step.

## Expected upload-key fingerprint

`swiss-knife:build_expected_upload_cert_set` with `sha256` (and `sha1` if known) records what Play expects. The upload then fails early in `swiss-knife:preflight_android` (`keystore_matches_expected`) instead of at Google. Note Google's error names a SHA-1; the build host compares SHA-256, so record both when you can.

## Git token for a private repo

Inspect `build_repo_status` first and retain an existing GitHub App installation that can read the repository. Private repositories work when included in that installation. Use a fine-grained token with read-only Contents for that repository only as a fallback when app access cannot be configured (GitLab: `read_repository`). Do not request a second credential when the existing connection works.

```
swiss-knife:build_credentials_save
  app_project_id, platform: "android", kind: "git_token", payload: { token }
```

Then `swiss-knife:build_git_token_check`.

## Replacing credentials

Read existing metadata first and retain a working credential unless the user requested replacement. An authenticated native connection with the required write scope and role can save the requested replacement directly, omitting `approval`. Read its validation result afterwards. Private bytes come from a user-provided local file or the private upload form, never chat.

Only if the server actually returns an approval refusal on a restricted legacy connection, call `swiss-knife:approval_request` for the same operation and relay its human step. Never invent or reuse a code. A missing role or scope needs the appropriate access, not an approval code.
