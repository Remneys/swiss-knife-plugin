# Errors and alerts: arguments and safe order

Read before triaging issues or creating, changing or deleting an alert rule.

## Issue triage

- `swiss-knife:error_issue_update`: send exactly one of `issue_id` or `issue_ids` (bulk, up to 50, all in one app), and `status` and/or `assignee_id` (one issue at a time; `null` unassigns). `resolution` only with `status: "resolved"`: `{ "mode": "immediate" }`, `{ "mode": "next_release" }` or `{ "mode": "release", "release": "1.4.2" }`. Repeating the same call leaves the same state.
- `swiss-knife:error_issue_archive`: `variant` is `until_escalating`, `for_7d`, `for_30d`, `until_100_events` or `until_10_users`. Not idempotent for the durations and count variants: a repeat re-baselines and logs a second archive. Read the issue with `swiss-knife:error_issue_detail` first.
- Any triage marks the issue reviewed.

## Alert conditions

- `new_issue`: a never-seen issue appears. The right default for an app with no baseline.
- `regression`: a resolved issue comes back.
- `spike`: needs `threshold` and `windowMinutes`.
- `metric_crash_free_users` / `metric_crash_free_sessions`: `thresholdWarning` and `thresholdCritical` as percentages, `metricWindowHours`, `minSessions`.
- `actionIntervalMinutes` is the cooldown between notifications; leave it unset unless the user asks.
- Targets: `notifyEmail`, `notifyWebhookUrl`. Both optional; `null` clears one on update.

## Create flow

1. `swiss-knife:error_alert_list` with `app_project_id`.
2. A rule with the same `condition` exists: reuse it. If it has no `notifyEmail`, `swiss-knife:error_alert_update` sets it.
3. None: one `swiss-knife:error_alert_create` (`name`, `condition`, and `notifyEmail` from `swiss-knife:workspace_whoami` `email` when present).
4. Never resend a create after a lost answer: list again. If a duplicate exists, tell the user, then `swiss-knife:error_alert_delete` the extra.
5. Re-read with `swiss-knife:error_alert_list`.

## What a rule read shows

`notifyEmail` as stored; `notifyWebhook` only as the webhook's origin, never its secret path. Do not ask for or echo a stored webhook URL. `state` and `lastTriggeredAt` tell whether the rule has fired.
