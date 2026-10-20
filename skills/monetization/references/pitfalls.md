**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

# Monetization pitfalls

Read when a monetization or catalogue call is refused, or a push step fails.

## Contents

- Config errors
- Catalogue refusals and store errors
- Price points
- Approvals and the push switch
- Order of a first launch

## Config errors

Format: situation -> what to do.

| Situation                                                                                 | What to do                                                                                                                                                                                                 |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `conflict`: SKU or entitlement key "exists but is ARCHIVED"                               | `swiss-knife:product_unarchive` or `swiss-knife:entitlement_unarchive`, or choose a different SKU/key. Store SKUs cannot be reused.                                                                        |
| `conflict`: SKU or key "already exists"                                                   | Reuse the existing row (list first). In `swiss-knife:monetization_setup`, a `grants_product_ids` or `packages[].product_id` SKU that is not in `products` falls back to an existing product with that SKU. |
| `monetization_setup` error with `details.failed_step` and `rolled_back: true`             | Nothing was created. Fix the named step's cause and resend the call.                                                                                                                                       |
| Same `idempotency_key`, different request                                                 | `conflict`. Use a new key for a changed request.                                                                                                                                                           |
| `swiss-knife:offering_create` `conflict` on slug                                          | The offering exists. `swiss-knife:offering_list`, reuse it. Without `slug` a retry creates a second offering (`main-1`): always pass `slug`.                                                               |
| `validation`: no paywall template                                                         | Pass `template_id` to `swiss-knife:offering_create`.                                                                                                                                                       |
| `swiss-knife:offering_archive` or `swiss-knife:offering_update` status `archived` refused | It is the current offering. `swiss-knife:offering_set_current` another one first, or demote with status `draft` on purpose.                                                                                |
| `swiss-knife:package_add` `conflict`                                                      | The product is already a package of that offering.                                                                                                                                                         |
| `swiss-knife:package_reorder` `conflict`                                                  | Stale list. `swiss-knife:package_list` again and send exactly those ids.                                                                                                                                   |
| `swiss-knife:entitlement_attach_product` `conflict`                                       | Already mapped; nothing to do.                                                                                                                                                                             |

## Catalogue refusals and store errors

| Code                                                                                     | Meaning and action                                                                                                                                                                                                                      |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `first_product_requires_binary` (refusal) / `product_first_requires_binary` (store 403)  | Apple will not take an app's first in-app purchase or subscription through the API. Relay the `human_action` verbatim and stop. After the human did it: `swiss-knife:store_sync_now`, then `swiss-knife:catalog_submission_plan` again. |
| `product_version_state_invalid`                                                          | Product versions are append-only (create and read only); the version aimed at is in review, approved or replaced. Relay, wait for the human. Not retryable.                                                                             |
| `immutable_field`, or `swiss-knife:catalog_product_save` `rejected`                      | Product id, type, group and billing period are fixed by the store. The remedy is a new product with the new value, never an edit.                                                                                                       |
| `offer_id_already_used` (fatal) / `offer_id_conflict`                                    | One id space across introductory, promotional, win-back and offer-code offers. Choose a new offer id. Never adopt the colliding object: it is probably a different offer with different terms.                                          |
| `offer_already_on_store`                                                                 | The store already holds that introductory offer; re-sending would duplicate it. Do not push it.                                                                                                                                         |
| `missing_review_screenshot` / `product_incomplete_for_submission`                        | Apple leaves a product in MISSING_METADATA without saying why. Supply the screenshot or localization, preview again.                                                                                                                    |
| `store_drift` / `not_reconciled`                                                         | The store changed since the draft. `swiss-knife:catalog_sync_now`, read `swiss-knife:catalog_list`, preview again.                                                                                                                      |
| `console_only` / `capability_unsupported`                                                | No API exists. Relay the `human_action` or `consoleUrl`.                                                                                                                                                                                |
| `push_disabled` / `disarmed: true`                                                       | The per-app push switch is off. See Approvals.                                                                                                                                                                                          |
| `PRECONDITION_FAILED` "not the push that was rehearsed"                                  | The draft, store or a price point changed. Preview again and use the new `fingerprint`.                                                                                                                                                 |
| `play_regions_version_stale` (retryable)                                                 | Retry; the version is re-read.                                                                                                                                                                                                          |
| `play_catalogue_write_conflict`, `product_write_conflict` (retryable)                    | Two writes raced. Check `swiss-knife:catalog_push_status`, retry once.                                                                                                                                                                  |
| `play_immutable_field`, `play_activation_refused`, `product_definition_rejected` (fatal) | The body is wrong; the identical retry fails identically. Fix the product and push a new plan.                                                                                                                                          |

## Price points

An Apple price is an opaque price point id scoped to both the product and the territory. A base price snaps to the nearest band, so a price the user typed may not exist.

- `swiss-knife:catalog_price_points`: cache only, no store call. Empty means nothing resolved yet.
- `swiss-knife:catalog_price_preview`: asks the store, spends a request window a whole App Store Connect Team shares, needs a write token. Do not loop it.
- `price_point_territory_mismatch`, `price_point_product_mismatch`, `price_point_unresolved`: nothing was sent. Re-resolve for that territory and product, preview again.
- An Apple price increase cannot be undone once in effect. Say so when the new price is higher than the current one, and push only on the user's explicit go-ahead.

## Approvals and the push switch

- `swiss-knife:catalog_push_set` with `enabled: true` needs an approval code bound to the app; turning it off needs none and is the brake.
- Each `swiss-knife:catalog_push` needs its own code, bound to the exact plan. A code from an earlier plan stops working when the plan changes. Codes are single use and expire in 15 minutes.
- A refusal carries `error.details.human_action`: relay it, wait, send the same call again with `approval`.

## Order of a first launch

1. Config (`swiss-knife:monetization_setup`).
2. Draft and price (`swiss-knife:catalog_product_save`, price points).
3. If it is the app's first product on Apple: the human submits it with a binary first.
4. Preview, approvals, push, status.
5. Sync and read back from the store.
