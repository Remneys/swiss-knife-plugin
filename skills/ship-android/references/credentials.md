**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

# Android credentials

Read when a credential is missing or invalid, the fingerprint is unknown, or the repo is private.

## Contents

- Play service account
- Upload keystore
- Expected upload-key fingerprint
- Git token for a private repo
- Approval handshake

## Play service account

Creating the service account and granting it access in Play Console is a human step (no API). If `swiss-knife:ship_prerequisites` says the app is not visible to it, relay its `human_action` and stop. Once the user has the JSON key:

```
swiss-knife:build_credentials_save
  app_project_id, platform: "android", kind: "play_service_account",
  payload: { serviceAccountJson }
```

Read `validation` in the answer; `swiss-knife:build_credentials_validate` re-checks.

## Upload keystore

Two cases, never mixed:

- The app already has an upload key Play accepted: the user supplies that keystore. Save it with kind `android_keystore`, payload `{ keystoreBase64, keystorePassword, keyAlias, keyPassword }`.
- A brand-new app with no upload key: `swiss-knife:build_android_keystore_generate` (approval code needed). Then `swiss-knife:build_android_keystore_certificate` returns the public certificate; the user registers it in Play Console (App integrity, App signing, upload key certificate; Play has no API for it) and tells you what fingerprint Play shows.

Overwriting a stored keystore loses the current key, so it needs an approval code.

## Expected upload-key fingerprint

`swiss-knife:build_expected_upload_cert_set` with `sha256` (and `sha1` if known) records what Play expects. The upload then fails early in `swiss-knife:preflight_android` (`keystore_matches_expected`) instead of at Google. Note Google's error names a SHA-1; the build host compares SHA-256, so record both when you can.

## Git token for a private repo

The Swiss-Knife GitHub App is private, so a private repo is not readable through it. Use a fine-grained personal access token with read-only Contents on that one repository (GitLab: `read_repository`; classic GitHub: `repo`).

```
swiss-knife:build_credentials_save
  app_project_id, platform: "android", kind: "git_token", payload: { token }
```

Then `swiss-knife:build_git_token_check`.

## Approval handshake

Replacing a stored credential, or generating a keystore, is refused without a code. Call `swiss-knife:approval_request` with the gated tool (`build_credentials_save`, `build_credentials_delete` or `build_android_keystore_generate`) and the arguments you mean to use. Relay what it says verbatim and stop. When the user gives you the code, repeat the same call with `approval` set to it. Do not invent, guess or reuse a code.
