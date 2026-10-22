---
name: ship-ios
description: Takes an iOS app from a connected repo to a processed build in TestFlight through the Swiss-Knife MCP server (credentials, preflight, build, upload, store check). Use when the user says "ship my app to TestFlight", "build and upload my iOS app", "get a TestFlight build", or a build or signing step fails, or asks to build, install or debug locally on an iPhone. Not for Android (use ship-android), store listings, App Review or production release, in-app purchases, or SDK setup (use setup-sdk).
---

# Ship an iOS build to TestFlight

**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

Success: `swiss-knife:testflight_overview` lists the new build as processed. Our own build status is not proof; Apple's answer is.

## Local iPhone development

Use the connected plugin to prepare signing and the local executor to build,
install and debug. Prepare the executor yourself using
[local-executor.md](references/local-executor.md); use its verified absolute
path even if `swiss-build` is absent from `PATH`. The user does not install a CLI,
Bun or npm package manually. Remote worker builds do not need this executor.

1. Match the project's bundle ID to `app_list`; reuse that app. Inspect the
   Xcode project, scheme and connected device. Full Xcode tools, device pairing,
   trust and Developer Mode are required; an Apple portal login on the
   developer's Mac is not. `--dry-run` inspects without creating a key or building.
2. Run the verified executable with `--json dev ios --project Example.xcodeproj
--scheme Example --device DEVICE_UDID --agent`, substituting the actual
   project, scheme and device. Keep this process running with writable stdin.
   On a valid cache it proceeds directly. Otherwise it prints an NDJSON frame
   `{protocol:1,type:"signing_required",id,request,tool:"ios_development_prepare"}`.
   The request contains a public CSR, phone and every target's entitlements;
   its private key stays in macOS Keychain.
3. Call `swiss-knife:ios_development_prepare` with `{app_project_id,request}`.
   If absent from the compact list, use `help {tool:"ios_development_prepare"}`
   then `tool_call`. Native members with write access can prepare signing
   directly after the app's owner setup. Send one JSON line to the same
   executor's stdin: `{protocol:1,type:"signing_bundle",id,bundle}`, using
   that frame's exact `id` and the returned `bundle`. Do not ask the person
   to copy a CSR, certificate or profile. The executor validates its key,
   phone and every extension, caches signing privately, then builds,
   installs and launches. Never send its private key to a tool.
4. Verify `installed` or `launched` and actual device evidence. A successful
   build alone is not proof of installation. `--refresh-signing --agent`
   requests fresh signing with the same local key. On interruption repeat
   the same command; do not delete the key or issue a new CSR needlessly.

If the execution host cannot keep stdin writable, `--prepare-signing` and
`--signing-bundle` remain an agent-managed fallback. Keep any bundle file
in a private directory (0700), mode 0600, outside source control and logs.
Do not combine `--signing-bundle` with `--refresh-signing`.

### One-time owner setup

Check `build_credentials_list` and validate the existing team key first.
Members reuse a connected app; they do not need the owner's Apple password,
a new team key, or a separate approval for each Mac or phone.
An administrator can enable signing with `signing_push_set` when the user
has authorized app setup. A member cannot grant themselves that permission.

For a missing or unusable team key, prefer the private Apple Account form:
Use `dashboard_open` and the in-app Apple Account control when the host
supports the embedded app. The internal form-preparation tool is app-only;
never invoke it directly or through `tool_call`. If the host cannot show the
widget, relay the structured `human_action` for the private dashboard form. The owner enters their Apple password
and verification code in the private widget or dashboard form, chooses the
team/provider and reviews any key creation there. Never request these secrets
in chat or tool arguments. Read [apple-owner.md](references/apple-owner.md)
for host fallback, verification and the optional local privacy helper.

Missing App IDs, App Groups or extension capabilities are a separate setup
issue. The private account form connects the team key; it does not configure
those capabilities. The reference describes the existing local owner helper
for that conditional case. Reuse existing state and retry signing after
configuration; never strip an entitlement to make a build pass.

### Phones and installation links

`ios_device_list {app_project_id}` lists existing Apple devices available
to the connection and phones collected for this app. Administrators can
see the configured Apple team's inventory; member visibility is app-scoped.
For a new phone, call `ios_device_enrollment_create {app_project_id}` and
share its registration URL or QR. The person opens it in iPhone Safari
and installs the device-registration profile in Settings. Poll
`ios_device_enrollment_get` only while pending, bounded by its expiry.
Collection does not itself register a device with Apple or install the app.
Actual signing registers the selected device and prepares every target.

`--identity SHA1` selects an existing development identity when ambiguous.
`--console` attaches logs; `--debug` starts LLDB in a human terminal;
`--no-install` only builds. Inspect trusted project dependencies before
using `--trust-package-plugins` for Swift package plugins and macros.

## Rules that never bend

- If a tool returns `human_action`, relay the required step and pause only the work that depends on it; continue independent authorized work. After the user completes it, call `verify_with`. Do not retry unchanged refusals.
- On restricted connections only, approval codes come from a signed-in human. Call `swiss-knife:approval_request` with the gated tool and your arguments, relay what it says, wait for the code, and send the same call again with it as `approval`. Never invent or reuse a code.
- Revoke or delete signing material only when the user explicitly selected that operation; inspect usage and honor the server’s in-use guard. A full certificate quota is not authorization to revoke another app’s key.
- Always `swiss-knife:ship_prerequisites` and `swiss-knife:build_preflight` before `swiss-knife:build_start`.
- Do not tell anyone to `npm install` swiss-build: it is not on npm. Its config is a plain object (see Step 3).
- In ChatGPT your model tools read status and propose builds. Use `swiss-knife:ship_prerequisites`, `swiss-knife:build_preflight`, `swiss-knife:build_get` and `swiss-knife:testflight_overview` for status; `swiss-knife:build_start` and `swiss-knife:build_retry` only propose a build here. A person confirms a build in its card. The in-chat app also provides private Apple Account connection, repository configuration and credential upload under Builds > CI, with current permissions and one in-app confirmation. App Store Connect .p8 keys, Play service-account JSON, iOS P12 certificates and Android keystores travel directly from that form to Swiss-Knife, never through chat or tool arguments. For an IPA or AAB already built elsewhere, the person can use Builds > CI > Upload a built app, then review registration after the private upload. Registration verifies the actual binary; store upload remains a separate action. Open the app with `swiss-knife:dashboard_open`. Never ask for private key bytes or passwords in chat. For an operation the app does not offer, relay its `human_action`; use a separate connected Codex or Claude Code session for a missing write tool. Never call a hidden tool.
- In Codex and Claude Code, use delegated write access as described above. Do not request an approval code proactively.

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

1. **Find the app.** `swiss-knife:workspace_whoami`, then `swiss-knife:app_list`. Reuse the app whose bundle id matches the project; keep its `app_project_id`. If there is none and the user authorized connecting a new app, create it; ask only when workspace or app identity is ambiguous.
2. **Credentials.** `swiss-knife:build_credentials_list`, then `swiss-knife:build_credentials_validate` for an existing `asc_api_key`. Reuse a valid connection. For a missing or invalid key, use the private Apple Account form described in references/apple-owner.md; a person can also upload an existing .p8 privately. Do not read private-key bytes into model context. Replace a stored key only when requested; native writes use delegated authorization. Read validation: `invalid` blocks shipping, `limited` needs its specific summary. Recipes: references/credentials.md.
3. **Repo.** `swiss-knife:build_repo_status`. Not connected: `swiss-knife:build_repo_connect` with the https URL. Keep an existing GitHub App connection when it can read the repository; do not ask for a token just because the repo is private. A `git_token` is a fallback when the app connection cannot provide access; read references/credentials.md. The repo needs a `swiss-build.config.ts` that default-exports a plain object (no import from `@app-swiss-knife/swiss-build`); without one, preflight blocks. Keep the shape `swiss-build init` writes: `app` (appProjectId, name, iosBundleId) plus `stages`.
4. **Prerequisites.** `swiss-knife:ship_prerequisites` with target `testflight`. Fix every `failed` item and re-run until `ready` is true. `not_verified` items are things no API can show; tell the user, do not claim them. Certificate item failed (`ASC_CERT_LIMIT`): do not build; read references/pitfalls.md.
5. **Preflight.** `swiss-knife:build_preflight` with platform `ios`. Fix each `blocking` issue, re-run. Note the `version` it reports: that is what the build will be filed under.
6. **Build.** `swiss-knife:build_start` with platform `ios` (pass `version` on the first build of an app; the build number is assigned for you). Keep `ids.jobId`. Poll `swiss-knife:build_get` with it until `succeeded` or `failed`; on `failed` read `swiss-knife:build_logs`, then the table in references/pitfalls.md. `swiss-knife:build_start` is not idempotent: before calling it again check `swiss-knife:release_list`, or pass `cancel_previous: true`. After a failure use `swiss-knife:build_retry`, which takes a fresh build number.
7. **Upload.** `swiss-knife:submission_upload_build` with the `release_id` (platform defaults to ios; TestFlight only, it cannot reach review). Then `swiss-knife:submission_upload_status` until `uploaded`. `blocked` carries a `human_action`: relay the required step and pause that dependent operation. `failed` carries `failureReason`: see references/pitfalls.md.
8. **Export compliance.** A build Apple holds at `MISSING_EXPORT_COMPLIANCE` reaches no tester. `swiss-knife:preflight_platform_requirements` (`release_id`) lists it under `exportCompliance.builds[]` with `waiting: true` and a `buildExternalId`. Use the user’s existing factual declaration about encryption, or ask if it has not been supplied; never invent an answer to get the build moving. Then `swiss-knife:testflight_export_compliance_answer` with `dry_run` (the default): it sends nothing and returns the verdict, the exact build and declaration. When the user’s factual answer matches that build, send the real call (`dry_run: false`, using any returned `confirmation` value as an API argument rather than asking the user to retype it); Codex and Claude Code omit `approval`. Only a restricted connection that actually requires a code uses `swiss-knife:approval_request`. It is a legal declaration under US export rules; if the binary declares non-exempt encryption it is refused and a person answers in App Store Connect with the export documentation.
9. **Verify.** `swiss-knife:testflight_overview`: the build must be listed. If it is not, say so; do not report success from the build status.

## When it breaks

- Build stuck at `MISSING_EXPORT_COMPLIANCE` -> step 8; do not wait for it to clear by itself.
  One line each; the full table is in references/pitfalls.md (read when any step fails).

- `ASC_CERT_LIMIT` -> stop; offer the human a .p12 upload or a portal revoke of their choice (and, if they own a second workspace on the team that holds a certificate, a team certificate link: references/pitfalls.md). Never revoke a certificate merely to free a slot. If the owner explicitly selects an unused certificate for revocation, use signing_action after inspecting its usage; the server’s in-use guard still applies.
- Private repo, clone fails -> inspect `build_repo_status` and the installation's repository access; retain a working GitHub App connection. Use a read-only `git_token` only if app access cannot be configured.
- App record missing in App Store Connect -> `human_action` from `swiss-knife:ship_prerequisites`; relay the required step and pause that dependent operation.
- Need to look inside the build VM over SSH -> never put `debug` on the build trigger (the trigger route refuses it, `debug_requires_approval`). Start the build first, then call `swiss-knife:build_debug_request` for that `job_id` and the requested public key and hold policy. Native admin connections omit `approval`; restricted connections use the returned human step.
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
