---
name: submit-release
description: Sends a build that is already in TestFlight or Play internal testing to App Review or Google Play production, with phased or staged rollout, release and rollout control, through the Swiss-Knife MCP server. Use when the user says "submit for App Review", "release to the App Store", "promote to production", "roll out to 10%", "halt the rollout" or "phased release". Not for getting the build onto TestFlight or Play internal (use ship-ios or ship-android) or store listings and screenshots.
---

# Submit a release for review and roll it out

**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

Success: the store itself says so. iOS: `swiss-knife:submission_status` after `swiss-knife:store_sync_now` shows the version in review, approved or released. Android: Google Play holds the production release at the fraction you set. A successful tool call is not proof; on Play it only means Google accepted the commit.

## Rules that never bend

- If a tool returns `human_action`, relay the required step and pause only the work that depends on it; continue independent authorized work. After the user completes it, call `verify_with`. Do not retry unchanged refusals.
- On restricted connections only, approval codes come from a signed-in human via `swiss-knife:approval_request`; never invent one. Call it with the gated tool and the same arguments, tell the user what `what_it_grants_in_plain_words` says, send them to `approve.url`, wait for the code, then repeat the SAME call with `dry_run: false` and `approval`. A code is single-use, bound to that exact action (a different `rollout_fraction: 0.5`, `fraction` or `replace_in_review` needs a new one) and expires.
- Always start with `dry_run: true` (the default) on every gated tool. Only the user's own words ("submit it", "ship it to 10%") start a real run; "check if it is ready" never does.
- Never set `replace_in_review`, `confirm_full_release`, `acknowledgements` or `provenance_override_reason` on your own. Each is the user's call; the override reason is their words, not yours.
- Never move to 100% (`rollout_fraction: 1`) unless the user asked for everyone at once.
- In ChatGPT the model reads `swiss-knife:submission_readiness`, `swiss-knife:submission_status`, `swiss-knife:store_overview` and `swiss-knife:testflight_overview`. Open `swiss-knife:dashboard_open` to inspect and use the in-chat app’s available release controls. A person confirms changes with the existing store switches, permissions and human approval; the model cannot call hidden writes. If a control requires action elsewhere or is not offered, relay its `human_action` and use the full MCP connection or the indicated human step.
- In Codex and Claude Code, use delegated write access as described above. Do not request an approval code proactively.

## Checklist

Copy and tick:

```
- [ ] 1. Build is processed in TestFlight / uploaded to Play internal (else ship-ios / ship-android)
- [ ] 2. submission_readiness has no blocking line; human_action lines relayed
- [ ] 3. submission_prepare done; submission_id kept
- [ ] 4. iOS: build attached, release type and phased release set. Android: fraction chosen
- [ ] 5. Dry run reviewed with the user
- [ ] 6. User-authorized real run sent using the connected permissions
- [ ] 7. Verified against the store
```

## Steps

1. **Find the release.** You need `release_id` and `app_project_id` (`swiss-knife:app_list`, `swiss-knife:release_list`). iOS: `swiss-knife:testflight_overview` must list a processed build. Android: `swiss-knife:submission_upload_status` (platform `android`) must say `uploaded`. If not, stop and use the ship skill.
2. **Readiness.** `swiss-knife:submission_readiness` (`release_id`, `platform`; Android production: `track: "production"`). Fix each blocking line and re-run. A line with `human_action` (App Privacy, a first in-app purchase, a draft review submission) is relayed verbatim and waits for the user. See references/pitfalls.md.
3. **Prepare.** `swiss-knife:submission_prepare`. iOS: optional `release_type: MANUAL`, `AFTER_APPROVAL` or `SCHEDULED` (the last with `earliest_release_date`); ask the user, do not choose. Keep the returned submission id (`swiss-knife:submission_list` shows it again).
4. **iOS: attach and configure.** `swiss-knife:submission_attach_build` (needs a processed build), then `swiss-knife:submission_configure_phased` with `enabled` as the user wants (7-day phased release). Set it before release: it cannot be started on a version that is already fully released.
5. **Android: choose the exposure.** The user names the fraction. For production there is no default: pass `rollout_fraction: 0.1` (or the user's number) to `swiss-knife:submission_promote_production`. Timed steps instead of manual bumps: `swiss-knife:rollout_plan_create`, then `swiss-knife:rollout_plan_control` with `start` (references/pitfalls.md says what it needs).
6. **Dry run.** iOS `swiss-knife:submission_submit`, Android `swiss-knife:submission_promote_production`, with `dry_run: true`. Show the user the readiness and payload.
7. **Real run.** If the user asked to submit or release and the preview matches that request, repeat with `dry_run: false`, omitting `approval` in Codex / Claude Code. Do not ask again. Only a restricted connection that actually requires a code uses `swiss-knife:approval_request`. A Play commit that Google already holds in review answers `alreadyInReview: true`: nothing was sent twice.
8. **Release controls.** For an approved iOS version held for manual release, use `swiss-knife:submission_release_now` when the user asks to publish. Android rollout: `swiss-knife:submission_control_rollout` (`halt`, `resume`, `bump` with `fraction`, or `complete`) within the requested exposure. Withdraw an iOS review with `swiss-knife:submission_cancel`. Android cannot un-ship: halt instead.
9. **Verify.** `swiss-knife:store_sync_now` (`mode: "full"` if the mirror looks incomplete), then `swiss-knife:submission_status` and `swiss-knife:store_sync_status`. Report what the store says: iOS `appVersionState`, Android track status, `userFraction` and `stateLabel`. "Sent to Google, not yet published" is not live. Never report success from the call alone.

## When it breaks

One line each; the full table is in references/pitfalls.md (read when any step fails).

- `forbidden` with `human_action` -> an approval code is missing: `swiss-knife:approval_request`, relay, wait.
- `fully_released_can_not_be_staged` -> the version is already 100% live; a phased release cannot be added. Tell the user; do not retry.
- `changes_not_sent_for_review` -> looks retryable, but a bare retry gets the same answer. A person sends or discards the waiting changes in Play Console, then repeat the call.
- `play_writes_blocked_after_rejection` -> Google refuses edits after a policy rejection. Human step in Play Console Policy status; relay and stop.
- `play_policy_declaration_incomplete` -> an App content declaration is open. Human step; relay `human_action` and stop.
- `play_release_in_review` -> wait, or the user chooses `replace_in_review`, which cancels the review and needs its own approval code.
