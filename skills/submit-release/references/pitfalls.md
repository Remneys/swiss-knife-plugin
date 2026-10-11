# Submit-release pitfalls

## Contents

- Approval handshake
- Store errors by code
- Readiness lines that need a person
- Scheduled rollouts (Google Play)
- Rejection and review state

## Approval handshake

Gated: `swiss-knife:submission_submit` (real run), `swiss-knife:submission_promote_production` (real run), `swiss-knife:submission_release_now`, `swiss-knife:submission_cancel`, `swiss-knife:submission_draft_clear`, `swiss-knife:submission_control_rollout` (`resume`, `bump`, `complete`, and `halt` only with `replace_in_review`), `swiss-knife:rollout_plan_control` (`start`, `resume`), `swiss-knife:rollout_automation_set` (turning it on).

Not gated: every `dry_run: true`, `swiss-knife:submission_readiness`, `swiss-knife:submission_prepare`, `swiss-knife:submission_attach_build`, `swiss-knife:submission_configure_phased`, `swiss-knife:submission_configure_rollout`, a plain `halt`, `pause` and `cancel` of a plan, and an Android submit to the internal track.

Without a valid code the call is refused as `forbidden` with `details.human_action`. Calling `swiss-knife:approval_request` first saves the round trip. Arguments for it: `submission_id` for the submission tools (+ rollout_fraction for promote, + `action` and `fraction` for control), `app_project_id` + `enabled` for `swiss-knife:rollout_automation_set`. The code is bound to the action and its arguments: a code issued for 10% cannot be replayed at 50%, and one issued without `replace_in_review` cannot be replayed with it. If the call is refused after the code was given, ask for a new one; never reuse or guess.

## Store errors by code

| Code or reason                                                                                       | Meaning                                                                                                                  | Action                                                                                                                                                                       |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `forbidden` + `human_action`                                                                         | No or wrong approval code                                                                                                | `swiss-knife:approval_request`, relay, wait, repeat the same call with the code                                                                                              |
| `precondition_failed` on `swiss-knife:submission_configure_rollout`                                  | The submission is already on Google Play (waiting, in review, rolling out, halted or live): the fraction is Google's now | `swiss-knife:submission_control_rollout` with `bump` and an approval code                                                                                                    |
| `fully_released_can_not_be_staged` (arrives as store_rejected)                                       | iOS version already 100% live; a phased release cannot start                                                             | Final. Say so; phased release has to be set before the version is released. Do not retry                                                                                     |
| `changes_not_sent_for_review` (arrives as `unavailable`, `retryable: true`)                          | Google will not send this app's changes for review by itself; a bare retry gets the same answer                          | Human: Play Console, Publishing overview, send the waiting changes for review or discard them (often after a rejection). Then repeat the call                                |
| `play_writes_blocked_after_rejection` (`needs_manual_action`)                                        | Google refuses edits after a policy rejection                                                                            | Human: Play Console Policy status, fix and resubmit or appeal. The block lifts when the mirror sees the build on its track: `swiss-knife:store_sync_now`                     |
| `play_policy_declaration_incomplete`, `play_app_content_declaration_missing` (`needs_manual_action`) | An App content declaration is open; no API                                                                               | Human: Play Console, Policy and programs, App content. Then repeat the same call. `swiss-knife:preflight_declaration_acknowledge` records the person's word, it is not proof |
| `play_release_in_review`                                                                             | A release is in review; a commit now would cancel it                                                                     | Wait, or with the user's say-so repeat with `replace_in_review: true`, `dry_run: false` and a code that names the replace                                                    |
| `play_permission_denied`                                                                             | The service account lacks production release rights                                                                      | Human step in Play Console, Users and permissions                                                                                                                            |
| `asc_version_state_invalid`, `asc_state_error`                                                       | Unsigned agreement, incomplete listing or a version that can no longer be edited                                         | Human: fix in App Store Connect, then repeat                                                                                                                                 |
| `alreadyInReview: true` in an ok answer                                                              | Google already holds exactly this release in review                                                                      | Nothing was sent again; verify with `swiss-knife:submission_status`                                                                                                          |

Any `needs_manual_action` answer carries `details.human_action`: relay it verbatim and stop.

## Readiness lines that need a person

- App Privacy (iOS): no API. The line carries `human_action` and an `acknowledge.token`. Relay; pass the token in `acknowledgements` on the real submit only after the user says it is published.
- First in-app purchase or subscription (iOS): has to be submitted with the binary from App Store Connect (`human_action`). Later ones go through MCP.
- Build not on the default branch (`provenance`, `override`): needs the user's own reason in `provenance_override_reason`, at least `override.minLength` characters. Never write it for them.
- Open review submissions (iOS): a draft holds something else. `swiss-knife:submission_draft_clear` with `dry_run: true` shows what it would withdraw; the real run needs approval, and a draft that changed since needs a new code.
- Android production: the checklist includes how long the versionCode has been on the internal track.

## Scheduled rollouts (Google Play)

A plan raises a staged rollout on its own, step by step. `swiss-knife:rollout_plan_create` takes `steps` (rising fractions, last 1 completes) and `interval_minutes` (15 to 43200); `dry_run` defaults true and only renders it. `swiss-knife:rollout_plan_get` lists `blockers`. Nothing starts until the app's rollout automation is on (off by default): `swiss-knife:rollout_automation_set` needs an approval code, then each plan's `swiss-knife:rollout_plan_control` `start` needs its own. `pause` and `cancel` are the brake; Google keeps serving what it serves now. `swiss-knife:rollout_plan_list` shows `automationEnabled` and each plan's status.

## Rejection and review state

App Review rejection (submission waits for an answer): the Resolution Center has no API. The human reads and replies; then `swiss-knife:store_sync_now`, `swiss-knife:submission_status`, and for a new build `swiss-knife:submission_attach_build` then `swiss-knife:submission_submit` again (dry run first).

`swiss-knife:submission_status` also carries `upload` (where the test-channel upload got to). A refused Play commit shows there, for example `play_release_in_review`. Repeating `swiss-knife:submission_upload_build` is a retry, not a status check.
