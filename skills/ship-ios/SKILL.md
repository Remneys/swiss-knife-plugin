---
name: ship-ios
description: Takes an iOS app from a connected repo to a processed build in TestFlight through the Swiss-Knife MCP server (credentials, preflight, build, upload, store check). Use when the user says "ship my app to TestFlight", "build and upload my iOS app", "get a TestFlight build", or a build or signing step fails. Not for Android (use ship-android), store listings, App Review or production release, in-app purchases, or SDK setup (use setup-sdk).
---

# Ship an iOS build to TestFlight

Success: `swiss-knife:testflight_overview` lists the new build as processed. Our own build status is not proof; Apple's answer is.

## Local iPhone development

For local changes and native debugging on an iPhone, use `swiss-build dev ios`.
Prepare the local executor yourself with
[local-executor.md](references/local-executor.md): it downloads the pinned
release, checks its SHA-256 and returns an absolute executable path. Use that
path for subsequent commands, even when `swiss-build` is absent from `PATH`.
The user does not need to install a CLI, Bun or npm package manually.
Remote worker builds use MCP and do not need this local executor.

```sh
swiss-build --json dev ios --project Example.xcodeproj --scheme Example --device DEVICE_UDID
```

Replace `swiss-build` with the verified absolute path and substitute actual
project, scheme and device values. A human terminal can omit
them and select interactively. `--dry-run` inspects without creating keys or
building. Full Xcode tools must be installed and the phone paired, trusted and
in Developer Mode. An Apple portal login on the developer's Mac is not required.

The first run returns `human_action` and exits 2. Relay its URL and key code.
An app administrator confirms the exact key and phone in their signed-in
browser. Never approve that form yourself or copy a session cookie. The printed
key fingerprint is not an approval token. The app needs existing Apple App IDs,
extension capabilities, a valid App Store Connect team key and signing writes
enabled by its owner. After human confirmation, rerun the identical command.
When the developer is a workspace member, do not ask their connection to enable
signing writes: that operation requires an administrator. Pass the app name and
the returned refusal or approval request to the administrator; do not invent an
approval URL. Local executor installation and a `--dry-run` remain independent
of this permission, so complete them before handing off the signing blocker.

The CLI reuses an existing Apple Development identity when unambiguous, or
creates a key in macOS Keychain. `--identity SHA1` selects an existing identity.
Private keys remain in the developer's Keychain. Profiles for the app and every extension
are validated before building Debug, installing and launching. Later builds
reuse valid signing material. In a human terminal, `--console` attaches logs
and `--debug` starts LLDB. `--no-install` only builds. A successful build is
not proof of installation: require `installed` or `launched` and device evidence.
For trusted project dependencies, `--trust-package-plugins` permits Swift package
plugins and macros without opening Xcode; inspect the dependencies before opting in.

## Rules that never bend

- Any tool answer with `human_action` (or a refusal that needs an approval): relay `who`, `what`, `url`, `why` to the user verbatim, then STOP. No retry, no polling, no workaround. When they say it is done, call the `verify_with` tool.
- Approval codes come from a signed-in human. Call `swiss-knife:approval_request` with the gated tool and your arguments, relay what it says, wait for the code, and send the same call again with it as `approval`. Never invent or reuse a code.
- Never revoke or delete a certificate, profile or credential yourself.
- Always `swiss-knife:ship_prerequisites` and `swiss-knife:build_preflight` before `swiss-knife:build_start`.
- Do not tell anyone to `npm install` swiss-build: it is not on npm. Its config is a plain object (see Step 3).
- In ChatGPT your model tools read status and propose builds. Use `swiss-knife:ship_prerequisites`, `swiss-knife:build_preflight`, `swiss-knife:build_get` and `swiss-knife:testflight_overview` for status; `swiss-knife:build_start` and `swiss-knife:build_retry` only propose a build here. A person confirms a build in its card. The in-chat app also provides repository configuration and credential upload under Builds > CI, with current permissions and required approval. App Store Connect .p8 keys, Play service-account JSON, iOS P12 certificates and Android keystores travel directly from that form to Swiss-Knife, never through chat or tool arguments. For an IPA or AAB already built elsewhere, the person can use Builds > CI > Upload a built app, then review registration after the private upload. Registration verifies the actual binary; store upload remains a separate action. Open the app with `swiss-knife:dashboard_open`. Never ask for private key bytes or passwords in chat. For an operation the app does not offer, relay its `human_action`; use a separate connected Codex or Claude Code session for a missing write tool. Never call a hidden tool.
- In Codex, as in Claude Code, you call the write tools yourself, including `swiss-knife:build_start` and `swiss-knife:build_retry`, which queue a build at once; a gated write (a store submit, a release, a signing, credential or catalogue change) stops and waits for a human approval code an admin issues, so call `swiss-knife:approval_request`, relay what it says and stop. Never make up a code.

## Checklist

Copy and tick:

```
- [ ] 1. Workspace and app found (adopt, never duplicate)
- [ ] 2. App Store Connect key stored and valid
- [ ] 3. Repo connected, config present, private repo readable
- [ ] 4. ship_prerequisites ready: true
- [ ] 5. build_preflight has no blocking issue
- [ ] 6. Build succeeded
- [ ] 7. Uploaded and processed
- [ ] 8. Verified in TestFlight
```

## Steps

1. **Find the app.** `swiss-knife:workspace_whoami`, then `swiss-knife:app_list`. Reuse the app whose bundle id matches the project; keep its `app_project_id`. If there is none, ask before creating one.
2. **Credentials.** `swiss-knife:build_credentials_list`. No `asc_api_key`: ask the user for issuer id, key id and the .p8, then `swiss-knife:build_credentials_save` (kind `asc_api_key`). Replacing a stored key needs an approval code. Then `swiss-knife:build_credentials_validate`; `invalid` means fix it now, `limited` means read the summary. Recipes: read references/credentials.md.
3. **Repo.** `swiss-knife:build_repo_status`. Not connected: `swiss-knife:build_repo_connect` with the https URL. Keep an existing GitHub App connection when it can read the repository; do not ask for a token just because the repo is private. A `git_token` is a fallback when the app connection cannot provide access; read references/credentials.md. The repo needs a `swiss-build.config.ts` that default-exports a plain object (no import from `@app-swiss-knife/swiss-build`); without one, preflight blocks. Keep the shape `swiss-build init` writes: `app` (appProjectId, name, iosBundleId) plus `stages`.
4. **Prerequisites.** `swiss-knife:ship_prerequisites` with target `testflight`. Fix every `failed` item and re-run until `ready` is true. `not_verified` items are things no API can show; tell the user, do not claim them. Certificate item failed (`ASC_CERT_LIMIT`): do not build; read references/pitfalls.md.
5. **Preflight.** `swiss-knife:build_preflight` with platform `ios`. Fix each `blocking` issue, re-run. Note the `version` it reports: that is what the build will be filed under.
6. **Build.** `swiss-knife:build_start` with platform `ios` (pass `version` on the first build of an app; the build number is assigned for you). Keep `ids.jobId`. Poll `swiss-knife:build_get` with it until `succeeded` or `failed`; on `failed` read `swiss-knife:build_logs`, then the table in references/pitfalls.md. `swiss-knife:build_start` is not idempotent: before calling it again check `swiss-knife:release_list`, or pass `cancel_previous: true`. After a failure use `swiss-knife:build_retry`, which takes a fresh build number.
7. **Upload.** `swiss-knife:submission_upload_build` with the `release_id` (platform defaults to ios; TestFlight only, it cannot reach review). Then `swiss-knife:submission_upload_status` until `uploaded`. `blocked` carries a `human_action`: relay and stop. `failed` carries `failureReason`: see references/pitfalls.md.
8. **Export compliance.** A build Apple holds at `MISSING_EXPORT_COMPLIANCE` reaches no tester. `swiss-knife:preflight_platform_requirements` (`release_id`) lists it under `exportCompliance.builds[]` with `waiting: true` and a `buildExternalId`. Ask the user whether the app uses only encryption Apple exempts (HTTPS/TLS through the operating system); never answer on your own to get a build moving. Then `swiss-knife:testflight_export_compliance_answer` with `dry_run` (the default): it sends nothing and returns the verdict, the exact `confirmation` phrase and the approval handshake. After the user's yes, `swiss-knife:approval_request`, relay, wait for the code, and send the real call (`dry_run: false`, that `confirmation`, `approval`). It is a legal declaration under US export rules; if the binary declares non-exempt encryption it is refused and a person answers in App Store Connect with the export documentation.
9. **Verify.** `swiss-knife:testflight_overview`: the build must be listed. If it is not, say so; do not report success from the build status.

## When it breaks

- Build stuck at `MISSING_EXPORT_COMPLIANCE` -> step 8; do not wait for it to clear by itself.
  One line each; the full table is in references/pitfalls.md (read when any step fails).

- `ASC_CERT_LIMIT` -> stop; offer the human a .p12 upload or a portal revoke of their choice (and, if they own a second workspace on the team that holds a certificate, a team certificate link: references/pitfalls.md). Never call `swiss-knife:signing_action` with `revoke_certificate`.
- Private repo, clone fails -> `git_token` (fine-grained, read-only Contents).
- App record missing in App Store Connect -> `human_action` from `swiss-knife:ship_prerequisites`; relay and stop.
- Need to look inside the build VM over SSH -> never put `debug` on the build trigger (the trigger route refuses it, `debug_requires_approval`). Start the build first, then `swiss-knife:approval_request` with tool `build_debug_request` and that `job_id`, relay the card, and send `swiss-knife:build_debug_request` with the admin's code as `approval`.
- Duplicate build number -> `swiss-knife:build_retry` or a new `swiss-knife:build_start`; the number is assigned for you.

## Native SDK note

Native iOS apps that use the Swiss-Knife SDK add it as a local SwiftPM package (`.package(path:)`); it is not published. That is `setup-sdk` territory, not this skill.

## Private build environment input

The in-chat app's Builds > CI environment form supports secret/sensitive/plain
values, UTF-8 text files, app/profile/workspace scope and environment targeting.
Values go directly to Swiss-Knife after review; do not ask for a secret in chat
or put it into app tool arguments. Secret edits can keep the existing encrypted
value. Workspace variables require an unrestricted admin connection.
`build_env_list` with `include_values:false` returns metadata and versions;
reviewed native writes may pass `expected_version` to reject competing edits.
