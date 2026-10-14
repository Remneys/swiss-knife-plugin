---
name: ship-ios
description: Takes an iOS app from a connected repo to a processed build in TestFlight through the Swiss-Knife MCP server (credentials, preflight, build, upload, store check). Use when the user says "ship my app to TestFlight", "build and upload my iOS app", "get a TestFlight build", or a build or signing step fails. Not for Android (use ship-android), store listings, App Review or production release, in-app purchases, or SDK setup (use setup-sdk).
---

# Ship an iOS build to TestFlight

Success: `swiss-knife:testflight_overview` lists the new build as processed. Our own build status is not proof; Apple's answer is.

## Rules that never bend

- Any tool answer with `human_action` (or a refusal that needs an approval): relay `who`, `what`, `url`, `why` to the user verbatim, then STOP. No retry, no polling, no workaround. When they say it is done, call the `verify_with` tool.
- Approval codes come from a signed-in human. Call `swiss-knife:approval_request` with the gated tool and your arguments, relay what it says, wait for the code, and send the same call again with it as `approval`. Never invent or reuse a code.
- Never revoke or delete a certificate, profile or credential yourself.
- Always `swiss-knife:ship_prerequisites` and `swiss-knife:build_preflight` before `swiss-knife:build_start`.
- Do not tell anyone to `npm install` swiss-build: it is not on npm. Its config is a plain object (see Step 3).
- In ChatGPT and Codex you hold the read tools, plus `swiss-knife:build_start` and `swiss-knife:build_retry` which only PROPOSE a build there (the card shows a button; the person presses it and confirms, you queue nothing and do not ask for an approval code; if the call is refused, the refusal carries a `human_action`). Credentials, repository connection, uploads and anything that changes a store are not offered there: do the read steps (`swiss-knife:ship_prerequisites`, `swiss-knife:build_preflight`, `swiss-knife:build_get`, `swiss-knife:testflight_overview`), name the step that needs a write, say that Claude Code does it, not this chat, and stop.

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
3. **Repo.** `swiss-knife:build_repo_status`. Not connected: `swiss-knife:build_repo_connect` with the https URL. A private repo needs a `git_token` credential; read references/credentials.md. The repo needs a `swiss-build.config.ts` that default-exports a plain object (no import from `@app-swiss-knife/swiss-build`); without one, preflight blocks. Keep the shape `swiss-build init` writes: `app` (appProjectId, name, iosBundleId) plus `stages`.
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
