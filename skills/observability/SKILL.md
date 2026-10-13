---
name: observability
description: Answers analytics questions and manages crashes and alerts for an app through the Swiss-Knife MCP server (events, funnels, retention, subscription and crash-free stats, saved reports, cohorts, boards, conversion alerts, anomalies, notifications, error issues and their grouping, error alert rules, data filters, debug files). Use when the user says "what's my signup to purchase funnel", "how many users came back after a week", "what's crashing", "is the new release stable", "triage this crash", "mark this crash issue resolved or ignored", or "alert me on new crashes", "save this report", "build a cohort of paying users", "merge these two crash issues", "upload my dSYM". Not for paywalls and pricing (use monetization), push (use push), or adding SDK events to code (use setup-sdk).
---

# Read analytics, triage errors, manage alerts

Success: every number or state you report was read from a swiss-knife tool in this conversation, using event and property names that `swiss-knife:event_list` / `swiss-knife:property_list` returned, and every write (triage, alert) was re-read afterwards and matches what the user asked for.

## Rules that never bend

- Any tool answer with `human_action` (or a refusal that needs an approval): relay `who`, `what`, `url`, `why` to the user verbatim, then STOP. No retry, no polling, no workaround. When they say it is done, call the `verify_with` tool.
- Approval codes come from a signed-in human. If a call is refused for an approval, call `swiss-knife:approval_request` with the gated tool and your arguments, relay what it says, wait for the code, and send the same call again with it as `approval`. Never invent or reuse a code.
- Never guess an event or property name. Call `swiss-knife:event_list` (and `swiss-knife:property_list` for properties) BEFORE `swiss-knife:funnel_report`, `swiss-knife:report_run` or `swiss-knife:retention_report`. A name that does not exist does not error: it returns zeros, and a zero funnel is a wrong answer, not a finding.
- `swiss-knife:error_alert_create` is not idempotent: `swiss-knife:error_alert_list` first, then at most one create. A repeated create makes a second rule.
- Never invent a `notifyEmail`. Use the `email` from `swiss-knife:workspace_whoami` or one the user gave in this conversation; otherwise omit it and say the rule has no recipient yet.
- In ChatGPT and Codex only the reads exist: report with `swiss-knife:error_stats`, `swiss-knife:error_issue_list`, `swiss-knife:error_issue_detail`, `swiss-knife:error_issue_grouping`, `swiss-knife:error_feedback_list`, `swiss-knife:error_settings_get`, `swiss-knife:error_alert_list`, `swiss-knife:saved_report_list`, `swiss-knife:cohort_list`, `swiss-knife:board_get`, `swiss-knife:funnel_alert_list`, `swiss-knife:anomaly_list`, `swiss-knife:notification_list`, `swiss-knife:notification_settings_get` and the reports. Resolving, ignoring, assigning, merging and unmerging issues, alert rules, filters, debug files, saved reports, cohorts, boards, anomalies and notification settings (`swiss-knife:error_issue_update`, `swiss-knife:error_issue_archive`, `swiss-knife:error_issue_merge`, `swiss-knife:error_alert_create`, `swiss-knife:saved_report_save`, `swiss-knife:cohort_save`, `swiss-knife:board_save`) are refused there: say they are done from Claude Code, not from this chat, and do not look for another way.

## Checklist

Copy and tick:

```
- [ ] 1. App found (app_project_id)
- [ ] 2. Question classified: number, funnel/retention, issue, or alert
- [ ] 3. Names discovered (event_list / property_list) before any report
- [ ] 4. Read run, window stated with the numbers
- [ ] 5. Writes: list first, one call, no repeat
- [ ] 6. Verified by re-reading (saved item, alert, merge, filter or symbolicated frame)
```

## Steps

1. **Find the app.** `swiss-knife:workspace_whoami`, then `swiss-knife:app_list`; keep the `app_project_id`. If the prompt gives it, skip discovery.
2. **Pick the read.** One call answers a headline question: `swiss-knife:dashboard_summary` (standup snapshot), `swiss-knife:subscription_stats` (MRR, active, churn), `swiss-knife:crash_free_stats` (`days`, default 7), `swiss-knife:error_stats` (`days`, default 1). Use a report only when none of these fits.
3. **Reports.** Discover names first: `swiss-knife:event_list` (top ~200 events of the last 30 days), then `swiss-knife:property_list` (optionally scoped to one event). Then:
   - Funnel: `swiss-knife:funnel_report` with 2 to 10 `steps` (`eventName`, optional `filters`), `conversion_window` in seconds, `timeRange`.
   - Retention: `swiss-knife:retention_report` with `cohort_event`, optional `return_event`, `granularity`, `periods`, `timeRange`.
   - Anything else: `swiss-knife:report_run` with `measures`, `timeRange`, optional `filters`, `breakdown`, `formula`.
   - Reuse a saved one: `swiss-knife:saved_report_list`, `swiss-knife:saved_report_get`.

   Shapes and examples: read references/query-shapes.md.

4. **Errors.** `swiss-knife:error_issue_list` (filter `status`, `release`, `environment`, `search`), then `swiss-knife:error_issue_detail`; dig with `swiss-knife:error_issue_events`, `swiss-knife:error_event_get` (stack trace) and `swiss-knife:error_affected_users`. Triage with `swiss-knife:error_issue_update` (status, resolution, assignee; safe to repeat) or `swiss-knife:error_issue_archive` (not idempotent: read the issue first). Read references/errors-and-alerts.md before a write.
5. **Saved analysis.** Keep a question the user will ask again: `swiss-knife:saved_report_save`, `swiss-knife:cohort_save` (then `swiss-knife:cohort_materialize`), `swiss-knife:board_save`; daily-checked conversion alerts with `swiss-knife:funnel_alert_set`; what the detector found with `swiss-knife:anomaly_list`. List first, one write, dry run before any delete. Sharing a board publicly (`swiss-knife:board_share_set`) only when the user asks. Arguments and safe order: read references/saved-analysis-and-errors-admin.md.
6. **Error alerts.** `swiss-knife:error_alert_list` first. If a rule with that condition exists, change it with `swiss-knife:error_alert_update` (`rule_id`); `active: false` pauses it. Only if none exists: ONE `swiss-knife:error_alert_create`. Removing a rule is `swiss-knife:error_alert_delete`; ask the user first.
7. **Errors admin.** Merging issues, splitting a grouping hash, reading crash feedback, inbound data filters (they DISCARD events) and registering dSYMs, mapping files or source maps for symbolication: read references/saved-analysis-and-errors-admin.md before the first write.
8. **Verify.** Re-read what you wrote: `swiss-knife:error_alert_list` after an alert write (the rule is there once, `active`, with the condition and target asked for); `swiss-knife:error_issue_detail` after a triage (status and assignee match). For a report, state the window and the sample size next to the number; if a count is 0, say whether the event appeared in `swiss-knife:event_list`.

## When it breaks

- Funnel or report returns zeros -> the event name is wrong or never sent. Re-read `swiss-knife:event_list`; if the event is absent, the app does not send it: tell the user, and point to setup-sdk for instrumentation. Do not report 0% conversion.
- `unavailable` (retryable) -> the analytics store did not answer; send the same read again once. If it persists, say so.
- `precondition_failed` "ClickHouse not configured" -> analytics storage is off in this deployment; report it, do not retry.
- `error_issue_events` empty with total 0 -> either no events or the store is not answering; confirm with `swiss-knife:error_stats`.
- `forbidden` on alert create, update or delete -> alert rules need the workspace admin role; tell the user who can do it.
- `not_found` on an issue or rule id -> the id is wrong or belongs to another workspace; re-list, do not retry.
- `conflict` on a triage -> the issue was merged into another; triage the one it was merged into (`details.merged_into_id`).
- `invalid_request` -> the message names the field; fix the arguments, do not guess.
- A bulk triage refused with `not_found` -> nothing was written; fix the id list and send again.
