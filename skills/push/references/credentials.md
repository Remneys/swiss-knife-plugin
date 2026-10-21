# Push credentials

Read when `swiss-knife:push_credentials_status` shows a provider not configured, or a send fails on credentials.

## APNs (iOS)

A human creates the key; no tool can. No tool answer carries a `human_action` for it, so explain this prerequisite and pause only work that needs the key:

1. Apple Developer portal, Certificates, Identifiers & Profiles, Keys, +. Name it, tick Apple Push Notifications service (APNs), register.
2. Download the `.p8` (Apple shows it once). Note the Key ID (10 characters) and the Team ID (10 characters).
3. The app's App ID needs Push Notifications enabled, and the build must carry the `aps-environment` entitlement.

Then `swiss-knife:push_credentials_set` with `kind: "apns_key"` and `credentials` = `{ key_id, team_id, bundle_id, private_key }`, where `private_key` is the PEM text of the `.p8`. The key is checked (EC P-256, able to sign a token) before it is stored and is never returned. One `.p8` works for every app on the team and for sandbox and production. Only a real device token proves the key with Apple, so the first real send is the real test.

## FCM (Android)

Firebase console, Project settings, Service accounts, Generate new private key. The JSON must contain `project_id`, `client_email` and `private_key`. The Firebase Cloud Messaging API (V1) must be enabled for the project and the service account needs the FCM Admin role. The Android app must use the same project's `google-services.json`. Save with `kind: "fcm_service_account"` and `credentials` = the JSON (object or text). Google refusing the account refuses the save; if Google is unreachable it is stored with `verified: null`.

## Replacing a key

Replacing a stored key changes which credential Swiss-Knife uses. Apple does not let you download a `.p8` again, so preserve the existing local key unless the user asked to replace it.

Read the current credential metadata first. When replacement is part of the user's request, call `push_credentials_set` directly with native admin write access and omit `approval`. Transfer secrets from a user-provided local file or the private credential form. Never paste private keys into chat. Verify the saved provider status.

Only an actual approval refusal on a restricted connection needs `approval_request`; follow that returned human step without guessing or reusing codes.
