# Submit-release pitfalls

## Contents

- Authorization and previews
- Store errors by code
- Readiness lines that need a person
- Scheduled rollouts (Google Play)
- Rejection and review state

## Authorization and previews

An authenticated Codex or Claude Code connection with the required write scope and workspace role can execute the user's release request directly. Omit `approval`. Preview consequential actions against current store state, then perform the requested operation without asking for a second yes. Readiness, store locks, app restrictions and factual declarations still apply.

A request to submit a release does not automatically authorize canceling an unrelated release in review, changing the rollout audience, or enabling automatic progression. Use `replace_in_review`, a new rollout fraction or automation only when it matches the user's instructions. If that intent is already explicit, do not ask again.

Only use `approval_request` if this connection actually returns an approval refusal, such as on an older restricted server. If `approval_required` is false, continue with the action; an informational approval response is not a blocker. In ChatGPT, the person confirms the exact action once inside its card; the model must not imitate that click or obtain its private proof.

## Store errors by code

| Code or reason                                                                                       | Meaning                                                                                                                  | Action                                                                                                                                                                       |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `forbidden` + `human_action`                                                                         | Read the actual role, scope or approval requirement                                                                      | Resolve the stated access requirement; use `approval_request` only for an actual approval refusal                                                                            |
| `precondition_failed` on `swiss-knife:submission_configure_rollout`                                  | The submission is already on Google Play (waiting, in review, rolling out, halted or live): the fraction is Google's now | `swiss-knife:submission_control_rollout` with `bump`; native write connections omit approval                                                                                 |
| `fully_released_can_not_be_staged` (arrives as store_rejected)                                       | iOS version already 100% live; a phased release cannot start                                                             | Final. Say so; phased release has to be set before the version is released. Do not retry                                                                                     |
| `changes_not_sent_for_review` (arrives as `unavailable`, `retryable: true`)                          | Google will not send this app's changes for review by itself; a bare retry gets the same answer                          | Human: Play Console, Publishing overview, send the waiting changes for review or discard them (often after a rejection). Then repeat the call                                |
| `play_writes_blocked_after_rejection` (`needs_manual_action`)                                        | Google refuses edits after a policy rejection                                                                            | Human: Play Console Policy status, fix and resubmit or appeal. The block lifts when the mirror sees the build on its track: `swiss-knife:store_sync_now`                     |
| `play_policy_declaration_incomplete`, `play_app_content_declaration_missing` (`needs_manual_action`) | An App content declaration is open; no API                                                                               | Human: Play Console, Policy and programs, App content. Then repeat the same call. `swiss-knife:preflight_declaration_acknowledge` records the person's word, it is not proof |
| `play_release_in_review`                                                                             | A release is in review; a commit now would cancel it                                                                     | Wait, or with the user's say-so repeat with `replace_in_review: true`, `dry_run: false` within that explicit request; native write connections omit approval                 |
| `play_permission_denied`                                                                             | The service account lacks production release rights                                                                      | Human step in Play Console, Users and permissions                                                                                                                            |
| `asc_version_state_invalid`, `asc_state_error`                                                       | Unsigned agreement, incomplete listing or a version that can no longer be edited                                         | Human: fix in App Store Connect, then repeat                                                                                                                                 |
| `alreadyInReview: true` in an ok answer                                                              | Google already holds exactly this release in review                                                                      | Nothing was sent again; verify with `swiss-knife:submission_status`                                                                                                          |

Any `needs_manual_action` answer carries `details.human_action`: relay the required step and pause only dependent work; verify after it is completed.

## Readiness lines that need a person

- App Privacy (iOS): no API. The line carries `human_action` and an `acknowledge.token`. Relay; pass the token in `acknowledgements` on the real submit only after the user says it is published.
- First in-app purchase or subscription (iOS): has to be submitted with the binary from App Store Connect (`human_action`). Later ones go through MCP.
- Build not on the default branch (`provenance`, `override`): needs the user's own reason in `provenance_override_reason`, at least `override.minLength` characters. Never write it for them.
- Open review submissions (iOS): a draft holds something else. `swiss-knife:submission_draft_clear` with `dry_run: true` shows what it would withdraw; the real run must match the user’s intent and the current draft. Native write connections omit approval; re-preview changed drafts.
- Android production: the checklist includes how long the versionCode has been on the internal track.

## Scheduled rollouts (Google Play)

A plan raises a staged rollout on its own, step by step. `swiss-knife:rollout_plan_create` takes `steps` (rising fractions, last 1 completes) and `interval_minutes` (15 to 43200); `dry_run` defaults true and only renders it. `swiss-knife:rollout_plan_get` lists `blockers`. Nothing starts until the app's rollout automation is on (off by default): enable it with rollout_automation_set and start the requested plan with rollout_plan_control. Native admin connections omit approval for both; the user must have requested automatic progression. `pause` and `cancel` are the brake; Google keeps serving what it serves now. `swiss-knife:rollout_plan_list` shows `automationEnabled` and each plan's status.

## Rejection and review state

App Review rejection (submission waits for an answer): the Resolution Center has no API. The human reads and replies; then `swiss-knife:store_sync_now`, `swiss-knife:submission_status`, and for a new build `swiss-knife:submission_attach_build` then `swiss-knife:submission_submit` again (dry run first).

`swiss-knife:submission_status` also carries `upload` (where the test-channel upload got to). A refused Play commit shows there, for example `play_release_in_review`. Repeating `swiss-knife:submission_upload_build` is a retry, not a status check.
