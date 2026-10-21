---
name: push
description: Sends push notifications to an app's users directly through APNs and FCM with the Swiss-Knife MCP server (APNs key and FCM service account, device registration, dry run, send, delivery check, token hygiene). Use when the user says "send a push notification", "notify all my users", "set up push", "add my APNs key", "why didn't the push arrive", or "cancel that push". Not for installing the SDK (use setup-sdk), shipping builds (use ship-ios / ship-android), or support chat and email.
---

# Send push notifications

**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

Success: `swiss-knife:push_message_get` shows the message `completed` with `sent` above zero and no `error`, and `swiss-knife:push_device_list` shows the target devices `active`. A `message_id` alone is not proof; delivery counts are.

## Rules that never bend

- If a tool returns `human_action`, relay the required step and pause only the work that depends on it; continue independent authorized work. After the user completes it, call `verify_with`. Do not retry unchanged refusals.
- On restricted connections only, approval codes come from a signed-in human. Call `swiss-knife:approval_request` with the gated tool and your arguments, relay what it says, wait for the code, and send the same call again with it as `approval`. Never invent or reuse a code.
- Always `swiss-knife:push_send` with `dry_run: true` first, then show the user the audience (`targeted_devices`, `devices_by_platform`) and the exact content. A push is not recallable once delivered.
- The exact content and audience must be authorized by the user. If the existing request specifies them, the dry run matches, and there are no new consequential differences, send without asking again. Ask only for missing content, an unexpected audience or a change in scope. Never widen the target from `target.user_ids` to `all` on your own.
- Never invent a key. The APNs `.p8` is issued by a human in the Apple Developer portal (the App Store Connect API cannot create one). Replace a stored credential only when the user authorized replacement; request a code only if the connection requires it.

## Checklist

In ChatGPT, open the in-chat app with `swiss-knife:dashboard_open`,
`section: "support"`, `tab: "push"` and the app id. The person composes and
reviews the notification there; the preview checks the payload and shows the
current audience before the admin approves the send. History, delivery counts
and cancellation are in the same view. The model is not given `push_send` and
must never try calling it through a hidden tool. The person adds APNs or Firebase credentials with the private form in this view;
replacing a stored key retains the admin role and one in-card confirmation.

Copy and tick:

```
- [ ] 1. App found (app_project_id)
- [ ] 2. Credentials: in Support > Push, the person uploads their APNs `.p8` or Firebase JSON using the private form. Only metadata enters MCP; replacing a key uses the admin role and in-card confirmation. Provider status shows what is saved and verified.
- [ ] 3. Devices registered: push_device_list has active devices for the audience
- [ ] 4. Dry run sent, audience and content shown to the user
- [ ] 5. Content and audience match the user's explicit request
- [ ] 6. Real send, message_id kept
- [ ] 7. Verified with push_message_get and push_device_list
```

## Steps

1. **Find the app.** `swiss-knife:workspace_whoami`, then `swiss-knife:app_list`; keep the `app_project_id`. Sending and saving credentials need the admin role.
2. **Credentials.** `swiss-knife:push_credentials_status`. Each of `apns` and `fcm` says `configured`. Save what is missing with `swiss-knife:push_credentials_set` (`kind` `apns_key` or `fcm_service_account`); recipes and the human steps are in references/credentials.md. Replacing a key must be part of the user’s requested change; native write connections omit approval.
3. **Devices.** `swiss-knife:push_device_list` (filter `user_id`, the external user id, and `platform`). Only `active` devices are sent to. No devices: the app has no SDK registration yet; hand off to setup-sdk and read references/sdk-registration.md for the exact `Notifications` calls.
4. **Dry run.** `swiss-knife:push_send` with `dry_run: true`, the same `target`, `kind` (`transactional` or `marketing`) and `content` as the real send. It validates the payload and credentials, sends nothing, creates nothing, and returns `targeted_devices` and `devices_by_platform` (plus `warnings` if a provider has no credential). An alert needs a `title` or `body`; `data` keys `aps`, `sk` and `sk_*` are reserved.
5. **Check intent.** Show the audience numbers and content. Continue when they match the user's request; clarify only missing or changed scope.
6. **Send.** The same call without `dry_run`. Add an `idempotency_key` so a retry returns the original message instead of sending twice. `scheduled_at` (ISO-8601 with offset) delivers later; targets are resolved when it fires. Keep `message_id`.
7. **Verify.** `swiss-knife:push_message_get` until `status` is `completed`, `failed` or `canceled`: read `sent`, `failed`, `invalidated_tokens`, `error` and `delivery_stats` (`received` and `opened` are reported back by the app's SDK, so they lag and may stay 0). Then `swiss-knife:push_device_list` for the same audience: devices the provider reported dead are no longer `active`. `swiss-knife:push_message_list` shows recent messages. To stop one that has not finished: `swiss-knife:push_message_cancel`; pushes already delivered stay delivered.

## Logout and the user link

The SDK leaves `user_id` out to keep a device linked (relaunch, refresh) and sends `user_id: null` to unlink it on logout, so the previous user stops receiving pushes on that phone. `target.user_ids` targets only linked devices: an empty audience on the dry run usually means the app never called `Identity.logIn`.

## When it breaks

Full table in references/pitfalls.md (read when a send fails).

- `credentials_missing` (422) on send -> no key for any targeted device's provider; save it (Step 2).
- Dry run `targeted_devices: 0` -> no active device for that audience; run `swiss-knife:push_device_list` without filters, then fix registration (setup-sdk).
- Message `failed` with an `error` about credentials -> the stored key is wrong (revoked `.p8`, wrong `bundle_id`, wrong Firebase project). Obtain a valid key through a private upload or local file and replace it with push_credentials_set when requested; native write connections omit approval.
- Many `invalidated_tokens` -> dead tokens, pruned by the server; nothing to fix, report the count.
- `conflict` on cancel -> the message already finished.
- `human_action` or an approval refusal -> the rules above.
