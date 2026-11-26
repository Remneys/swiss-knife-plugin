# Serve, change and test paywalls

Read when the user wants to change which paywall (offering) users see, change what a served paywall says, redesign it with AI, or run a paywall experiment. `swiss-knife:help` with goal `change_current_offering` gives the same order of calls.

Success: `swiss-knife:offering_preview` for a user the change is meant for answers the intended offering, and a published paywall shows in `swiss-knife:offering_get`. Reads are viewer; every write here is admin, except experiments (member; concluding and deleting are admin).

## Contents

- Check before you serve
- Serve it to everyone or to some users
- Change a paywall through a draft
- Paywall experiments
- When it breaks

## Check before you serve

1. `swiss-knife:offering_list` for the ids and which one `isCurrent`.
2. `swiss-knife:offering_verify` (`offering_id`, or none for every offering) walks offering, packages, products, store state and the entitlements they grant, and returns `issues`, each with a stable code and a severity. `ok: false` means an error: fix those first (packages with `swiss-knife:package_add`, a product missing from the store through the catalogue steps in SKILL.md). `store_checked: false` means the store state was not checked.
3. `swiss-knife:offering_preview` with `app_user_id`, `placement`, `platform`, `country` and `app_version` as the app would send them: which offering that user is served now and why (`reason`, `rules_evaluated`), with both SDK answers (`sdk_response`, `rc_response`). Nothing is recorded.

## Serve it to everyone or to some users

- **Everyone no rule matches:** `swiss-knife:offering_set_current`, `dry_run: true` first (Step 4 in SKILL.md). An active rule that matches a user wins over the current offering.
- **Some users: an offering rule.** `swiss-knife:offering_rule_create` with `name`, `offering_id` and who it is for: an `audience_id`, inline `conditions`, or both (a list is ANDed; fields platform, country, app_version, app_user_id, attribute, first_seen, entitlement and percentage), optionally a `placement`, a schedule (`starts_at`, `ends_at`) and a `priority` (lowest first). A rule is created inactive. Activating it (`state: "active"` on create, or `swiss-knife:offering_rule_update`) changes what the SDK serves, so that call answers `applied: false` with the `plan` and `plan_fingerprint`; send it again with `confirm: true` and that fingerprint when the plan matches the user's request. `swiss-knife:offering_rule_update` takes `expected_revision` from `swiss-knife:offering_rule_get`. To stop a rule reversibly set it inactive; `swiss-knife:offering_rule_delete` is for good.
- These rules choose offerings only. A feature flag's targeting is `swiss-knife:flag_update`.
- **Saved condition sets:** `swiss-knife:audience_create` (`name`, `conditions`), `swiss-knife:audience_list`, `swiss-knife:audience_get` (`stats: true` counts matching app users over the most recently seen ones; conditions only a request knows are left out, so it is then an upper bound), `swiss-knife:audience_update` and `swiss-knife:audience_delete`.
- Rule and audience creates are not idempotent: a name the app already has is `conflict`. After a lost answer, list before creating again.
- **Verify:** `swiss-knife:offering_preview` for one user the rule should match and one it should not.
- **RevenueCat SDK clients.** An app in `legacy_default` mode (`rcOfferingsMode` on `swiss-knife:app_list`; `swiss-knife:offering_verify` reports it) answers RevenueCat's offerings route with one synthetic `default` offering of every product, ignoring rules. `swiss-knife:app_update` with `rcOfferingsMode: "configured"` serves the real offerings and rules there instead. That changes what every installed RevenueCat client receives: do it only on the user's request, after `swiss-knife:offering_preview` shows the `rc_response` they will get.

## Change a paywall through a draft

A draft stages a change over the live config and is published on purpose. Every state is kept as a revision.

1. `swiss-knife:paywall_draft_get` (`offering_id`): the draft if there is one, its `draft_version`, `changed_paths`, `live_changed_since_base` and a one-hour `preview_url`.
2. Edit by hand with `swiss-knife:paywall_draft_save`: a `config_patch` (a JSON merge patch, checked as `swiss-knife:offering_update` checks it) or a whole `config`. With AI: `swiss-knife:paywall_generate` (`prompt`; `style` sections or custom_html; designs it anew and keeps the offering's products and their order) or `swiss-knife:paywall_edit` (`prompt`; changes copy, layout and colours of the draft). Both run the workspace's AI model, cost a model call each time, save the result as the draft and never publish. Pass `expected_version` so a retry after a lost answer is a `conflict` rather than a second model run. Read `explanation` and `changed_paths`.
3. Show the user the `preview_url`.
4. Publish with `swiss-knife:paywall_publish` and `expected_version` set to the `draft_version` the user reviewed. When real users are served the offering (the current one, a rule's target, an experiment's variant), the first call answers the plan (`published: false`, `served_by`); send `confirm: true` when it matches the user's request. If `live_changed_since_base` is true, publishing replaces a live change made after the draft started: say so first.
5. Undo: `swiss-knife:paywall_revision_list` lists every config the paywall has had (`revision` for one, with its config and a preview link); `swiss-knife:paywall_rewind` copies one back into the draft, then publish again. `swiss-knife:paywall_draft_discard` drops the draft; its states stay in the revisions.
6. Verify: `swiss-knife:offering_get` shows the new live config, and `swiss-knife:paywall_draft_get` answers `has_draft: false`.

## Paywall experiments

- **Read.** `swiss-knife:experiment_list` (`status` draft, running, paused or concluded), `swiss-knife:experiment_get`, `swiss-knife:experiment_results` (optionally `platform` or `country` for a slice). `trending` is not a result: report a winner only at `significant_winner`, with the sample sizes, and say when `needs_more_users` is above 0.
- **Create and start.** `swiss-knife:experiment_create` with `key`, `name` and 2 to 10 `variants` (traffic summing to 100, exactly one control; in a paywall experiment each variant's payload names one of this app's paywalls) and `enrollment_percentage` (1 to 100, the share of users enrolled at all). The same definition again answers the existing experiment. Then `swiss-knife:experiment_start`.
- **Pause and resume.** `swiss-knife:experiment_pause` stops new enrollment; assigned users keep their variant and stay in the results. `swiss-knife:experiment_resume` restarts it. Both answer a review first (assignments, effect, `plan_fingerprint`); send `confirm: true` with that fingerprint when it matches the user's request. Repeating the current state answers `changed: false`.
- **Change.** `swiss-knife:experiment_update` changes only the fields sent. A draft may change everything; once started, only `name`, `description` and a higher `enrollment_percentage`. Anything else is a `conflict` naming why: conclude it and create a new experiment. Pass `expected_updated_at` from the read.
- **End.** `swiss-knife:experiment_conclude` from running or paused, `dry_run: true` first to read the winner's effects (a paywall experiment activates the winning paywall and archives the others), then the real call. `swiss-knife:experiment_delete` refuses a running or paused experiment.

## When it breaks

| Situation                                                                                                            | What to do                                                                                                                                                          |
| -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `conflict` with a stale `plan_fingerprint`, `expected_revision`, `expected_version` or `expected_updated_at`         | It changed since you read it. Read again, show the new state, retry with the new value.                                                                             |
| `validation` from a draft save, an AI call, a rewind or a publish                                                    | The config would be invalid (`details.field_errors`), or it names a product the app no longer has. Nothing was saved. An AI answer that failed may be retried once. |
| `precondition_failed` with `human_action` from `swiss-knife:paywall_generate` or `swiss-knife:paywall_edit`          | The workspace has no AI key. Relay the step.                                                                                                                        |
| `rate_limited` from an AI call                                                                                       | The shared allowance is used up. Wait `retry_after_seconds`, then retry once.                                                                                       |
| `precondition_failed` from `swiss-knife:paywall_publish`                                                             | There is no draft to publish.                                                                                                                                       |
| `conflict` from `swiss-knife:offering_rule_create` naming an archived offering                                       | Serve an active offering, or restore that one first.                                                                                                                |
| `conflict` from `swiss-knife:audience_delete`                                                                        | A rule still uses it. Repoint or delete that rule first, with the user's go-ahead.                                                                                  |
| `conflict` from `swiss-knife:experiment_pause`, `swiss-knife:experiment_resume` or `swiss-knife:experiment_conclude` | The experiment is a draft or already concluded; `swiss-knife:experiment_get` shows its state.                                                                       |
