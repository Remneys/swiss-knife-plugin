---
name: push
description: Sends push notifications to an app's users directly through APNs and FCM with the Swiss-Knife MCP server (APNs key and FCM service account, device registration, dry run, send, delivery check, token hygiene). Use when the user says "send a push notification", "notify all my users", "set up push", "add my APNs key", "why didn't the push arrive", or "cancel that push". Not for installing the SDK (use setup-sdk), shipping builds (use ship-ios / ship-android), or support chat and email.
---

# Send push notifications

Success: `swiss-knife:push_message_get` shows the message `completed` with `sent` above zero and no `error`, and `swiss-knife:push_device_list` shows the target devices `active`. A `message_id` alone is not proof; delivery counts are.

## Rules that never bend

- Any tool answer with `human_action` (or a refusal that needs an approval): relay `who`, `what`, `url`, `why` to the user verbatim, then STOP. No retry, no polling, no workaround. When they say it is done, call the `verify_with` tool.
- Approval codes come from a signed-in human. Call `swiss-knife:approval_request` with the gated tool and your arguments, relay what it says, wait for the code, and send the same call again with it as `approval`. Never invent or reuse a code.
- Always `swiss-knife:push_send` with `dry_run: true` first, then show the user the audience (`targeted_devices`, `devices_by_platform`) and the exact content. A push is not recallable once delivered.
- `swiss-knife:push_send` is not approval-gated, so the confirmation is yours: for `target: {"all": true}`, any `kind: "marketing"` send, or more than a handful of devices, get an explicit yes from the user before the real send. Never widen the target (from `target.user_ids` to `all`) without asking.
- Never invent a key. The APNs `.p8` is issued by a human in the Apple Developer portal (the App Store Connect API cannot create one). Never replace a stored credential without an approval code.

## Checklist

In ChatGPT, open the in-chat app with `swiss-knife:dashboard_open`,
`section: "support"`, `tab: "push"` and the app id. The person composes and
reviews the notification there; the preview checks the payload and shows the
current audience before the admin approves the send. History, delivery counts
and cancellation are in the same view. The model is not given `push_send` and
must never try calling it through a hidden tool. The person adds APNs or Firebase credentials with the private form in this view;
replacing a stored key retains the administrator approval step.

Copy and tick:

```
- [ ] 1. App found (app_project_id)
- [ ] 2. Credentials: in Support > Push, the person uploads their APNs `.p8` or Firebase JSON using the private form. Only metadata enters MCP; replacing a key requires administrator approval. Provider status shows what is saved and verified.
- [ ] 3. Devices registered: push_device_list has active devices for the audience
- [ ] 4. Dry run sent, audience and content shown to the user
- [ ] 5. User confirmed (always for all / marketing / large audiences)
- [ ] 6. Real send, message_id kept
- [ ] 7. Verified with push_message_get and push_device_list
```

## Steps

1. **Find the app.** `swiss-knife:workspace_whoami`, then `swiss-knife:app_list`; keep the `app_project_id`. Sending and saving credentials need the admin role.
2. **Credentials.** `swiss-knife:push_credentials_status`. Each of `apns` and `fcm` says `configured`. Save what is missing with `swiss-knife:push_credentials_set` (`kind` `apns_key` or `fcm_service_account`); recipes and the human steps are in references/credentials.md. The first key of a kind is free; replacing one needs an approval code.
3. **Devices.** `swiss-knife:push_device_list` (filter `user_id`, the external user id, and `platform`). Only `active` devices are sent to. No devices: the app has no SDK registration yet; hand off to setup-sdk and read references/sdk-registration.md for the exact `Notifications` calls.
4. **Dry run.** `swiss-knife:push_send` with `dry_run: true`, the same `target`, `kind` (`transactional` or `marketing`) and `content` as the real send. It validates the payload and credentials, sends nothing, creates nothing, and returns `targeted_devices` and `devices_by_platform` (plus `warnings` if a provider has no credential). An alert needs a `title` or `body`; `data` keys `aps`, `sk` and `sk_*` are reserved.
5. **Confirm.** Show the user the audience numbers and the content. Wait for a yes where the rules above require it.
6. **Send.** The same call without `dry_run`. Add an `idempotency_key` so a retry returns the original message instead of sending twice. `scheduled_at` (ISO-8601 with offset) delivers later; targets are resolved when it fires. Keep `message_id`.
7. **Verify.** `swiss-knife:push_message_get` until `status` is `completed`, `failed` or `canceled`: read `sent`, `failed`, `invalidated_tokens`, `error` and `delivery_stats` (`received` and `opened` are reported back by the app's SDK, so they lag and may stay 0). Then `swiss-knife:push_device_list` for the same audience: devices the provider reported dead are no longer `active`. `swiss-knife:push_message_list` shows recent messages. To stop one that has not finished: `swiss-knife:push_message_cancel`; pushes already delivered stay delivered.

## Logout and the user link

The SDK leaves `user_id` out to keep a device linked (relaunch, refresh) and sends `user_id: null` to unlink it on logout, so the previous user stops receiving pushes on that phone. `target.user_ids` targets only linked devices: an empty audience on the dry run usually means the app never called `Identity.logIn`.

## When it breaks

Full table in references/pitfalls.md (read when a send fails).

- `credentials_missing` (422) on send -> no key for any targeted device's provider; save it (Step 2).
- Dry run `targeted_devices: 0` -> no active device for that audience; run `swiss-knife:push_device_list` without filters, then fix registration (setup-sdk).
- Message `failed` with an `error` about credentials -> the stored key is wrong (revoked `.p8`, wrong `bundle_id`, wrong Firebase project). Stop; the human issues a new key and `swiss-knife:push_credentials_set` replaces it with an approval code.
- Many `invalidated_tokens` -> dead tokens, pruned by the server; nothing to fix, report the count.
- `conflict` on cancel -> the message already finished.
- `human_action` or an approval refusal -> the rules above.
