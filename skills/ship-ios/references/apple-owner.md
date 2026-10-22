# Connect the Apple owner account privately

Use this only when the app lacks a usable team App Store Connect key, or an
owner must configure missing App IDs/capabilities. Connected member builds
skip Apple sign-in and reuse the app's configured team.

## Private form first

1. Check `build_credentials_list` and validate an existing key. If usable,
   continue the build instead of creating a new connection or key.
2. Use `dashboard_open` and the in-app Apple Account control when the host
   supports the embedded app. Its internal preparation tool is app-only:
   never call it directly or through `tool_call`. The widget receives its
   short-lived authorization outside model-visible results. A host without
   embedded-app support uses the structured `human_action` private-form URL.
3. The owner enters Apple Account email, password and verification code in
   that form, chooses the correct team/provider and reviews key creation or
   reuse. Password/OTP travel privately to Swiss-Knife and Apple, never to
   the agent. Do not ask the person to paste them into chat, call a tool with
   them, or capture form fields in screenshots or logs.
4. If the host cannot display the private widget, use the server's structured
   `human_action` for the private dashboard form. Keep the secret entry in
   that form. Do not turn the absence of a widget into a password tool call.
5. After completion, re-read and validate the app credential. The form does
   not install the app or itself prove signing/build success. Continue the
   connected local build or remote build and verify the actual device/store.

An expired sign-in can be restarted in the form. For an uncertain Apple key
creation, use its explicit reconciliation action to check the saved attempt;
never create or revoke another key automatically. Verification resend is for
phones; on a trusted Apple device use the code already shown there.

## Optional local privacy helper

Use this when the owner prefers Apple password entry only on their Mac and
Apple, or private form login is unavailable. Prepare the verified executor
using [local-executor.md](local-executor.md). The agent handles installation.

1. Create a private temporary directory (0700) and choose a fresh result
   filename. Open a private native terminal for the owner and run the verified
   executable with `apple connect --app-id APP_UUID --team-id TEAM_ID
--result-file /absolute/private/result.json`. Omit `--team-id` if not known.
   The owner enters password/2FA there and reviews the intended team/key.
   Do not capture or read that terminal transcript: verification prompts can
   echo codes. Do not pass passwords or codes in arguments, stdin from the
   agent, environment, or `--json`.
2. Read only the result file, which contains public status/metadata. For
   `credential_upload_required`, call `build_credential_upload_authorize`
   with `app_project_id`, `platform`, `kind`, `payload_sha256`, `payload_bytes`
   and `replace:false`. Do not pass `pending_id` or `status` to that tool.
3. Run `apple connect --pending PENDING_UUID --upload` and deliver the tool's
   `{endpoint,authorization}` through stdin. The helper transfers the private
   key directly, deletes its pending secret only after confirmed success,
   and leaves no private bytes for the agent to read. Validate the saved key.
4. After interruption, reuse `apple connect --pending PENDING_UUID` for
   public metadata, or repeat the original app/team command to reconcile an
   uncertain create. Preserve the same attempt and key. A consumed one-time
   Apple download cannot be restored without the original PEM; report the
   returned key ID/recovery state instead of repeatedly creating keys.

## Missing App IDs or capabilities

The private account form connects credentials. It currently does not run
App ID, App Group or extension-capability configuration.

When signing reports those missing prerequisites, the agent saves the public
signing `request` to a local file; do not ask a member to exchange CSR files.
Run the verified executor in the owner's private native terminal with
`apple configure --team-id TEAM_ID --request /absolute/public-request.json
--result-file /absolute/private/configure-result.json`. Read only that public
result file, not the terminal transcript. The helper reuses its Apple session,
finds or creates app/extension IDs and shared groups, preserves existing
capabilities, and verifies readback. Unsupported entitlements are reported
before writes; fix the stated capability rather than removing it.
After configuration, retry the same member signing request.

The local helper keeps Apple cookies in the owner's private cache; the
private form uses an isolated short-lived server process. These workflows
use Apple's unofficial web APIs, so Apple may require a fresh sign-in or
agreement acknowledgment. Follow the specific returned prerequisite.
