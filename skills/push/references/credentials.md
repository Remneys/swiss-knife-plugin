**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

# Push credentials

Read when `swiss-knife:push_credentials_status` shows a provider not configured, or a send fails on credentials.

## APNs (iOS)

A human creates the key; no tool can. No tool answer carries a `human_action` for it, so tell the user yourself and STOP until they have the file:

1. Apple Developer portal, Certificates, Identifiers & Profiles, Keys, +. Name it, tick Apple Push Notifications service (APNs), register.
2. Download the `.p8` (Apple shows it once). Note the Key ID (10 characters) and the Team ID (10 characters).
3. The app's App ID needs Push Notifications enabled, and the build must carry the `aps-environment` entitlement.

Then `swiss-knife:push_credentials_set` with `kind: "apns_key"` and `credentials` = `{ key_id, team_id, bundle_id, private_key }`, where `private_key` is the PEM text of the `.p8`. The key is checked (EC P-256, able to sign a token) before it is stored and is never returned. One `.p8` works for every app on the team and for sandbox and production. Only a real device token proves the key with Apple, so the first real send is the real test.

## FCM (Android)

Firebase console, Project settings, Service accounts, Generate new private key. The JSON must contain `project_id`, `client_email` and `private_key`. The Firebase Cloud Messaging API (V1) must be enabled for the project and the service account needs the FCM Admin role. The Android app must use the same project's `google-services.json`. Save with `kind: "fcm_service_account"` and `credentials` = the JSON (object or text). Google refusing the account refuses the save; if Google is unreachable it is stored with `verified: null`.

## Replacing a key

The first key of a kind is free. Replacing a stored one is irreversible (a `.p8` cannot be downloaded again), so the save is refused until a human approves it:

1. `swiss-knife:approval_request` with `tool: "push_credentials_set"`, `app_project_id` and `kind`.
2. Relay it, wait for the code, repeat the same `swiss-knife:push_credentials_set` call with `approval`.

An identical re-send is a no-op and needs no code. Check the result with `swiss-knife:push_credentials_status`: it shows `key_id`, `team_id`, `bundle_id` (APNs) and `project_id`, `client_email`, `verified` (FCM), never a key.
