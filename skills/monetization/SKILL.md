---
name: monetization
description: Sets up and runs an app's monetization through the Swiss-Knife MCP server - products, entitlements, offerings (the paywall the SDK serves) and packages, the priced store catalogue pushed to App Store Connect or Google Play (new products included), paywall drafts and AI redesigns, which paywall each user is served (offering rules, audiences, paywall experiments), and one customer's subscriptions, free grants, extensions and refunds. Use when the user says "add a subscription", "set up my paywall", "change the price", "push my products to the store", "show this user's subscription", "give this user a free month", "refund this purchase", "extend their subscription", "show a different paywall in Germany", "pause the paywall test", "redesign the paywall with AI". Not for SDK code (use setup-sdk), builds and review submission (use ship-ios, ship-android) or revenue charts over time (use observability).
---

# Set up monetization, serve paywalls, help customers

**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

Success: the store itself lists the product with the intended price (`swiss-knife:catalog_sync_now`, then `swiss-knife:catalog_list`), and `swiss-knife:offering_get` shows the current offering with its packages. Our own push status is not proof.

Three layers, do not mix them up:

- Config the SDK serves: products, entitlements, offerings, packages (`swiss-knife:product_create` and friends). Writes only our database.
- The store catalogue: `swiss-knife:catalog_*`. Only `swiss-knife:catalog_push` can write to a store.
- A customer's state: subscriptions, purchases and grants, as the stores and the SDK reported them. Reads never call a store; only `swiss-knife:subscription_extend`, `swiss-knife:subscription_refund` and `swiss-knife:purchase_refund` write to one.

A config product and a catalogue product meet by platform plus store SKU (`product_id`), so use the same SKU in both.

An offering is the paywall. Native clients edit its `config` or `config_patch`; `offering_get` returns its schema, preview URL, opaque `content_version` and `updatedAt`. Send `content_version` as `expected_version` to refuse a concurrent change; `expected_updated_at` remains supported for older clients. The in-chat app's Paywalls > Design & packages view offers one editor for content, plan display labels, style and custom HTML source, with reviewed saving. Its local preview covers structured text, package labels and styling; images and countdown behavior appear in the saved preview. Custom HTML is edited as text and never executed in the chat; the review identifies its old and new byte length and hash. Reading the offering in a mobile app is `setup-sdk` territory.

## Pick the job

| The user wants to                                                                                               | Do                                                           |
| --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Products, prices, the offering, a new store product, the store push                                             | the Checklist and Steps below                                |
| Look up one customer's subscription or purchase, give free access, extend or refund                             | read references/customers.md                                 |
| Change which paywall users see (current offering, rules, audiences), a served paywall's content, an AI redesign | read references/paywalls-and-targeting.md                    |
| Run, pause, resume or change a paywall experiment                                                               | read references/paywalls-and-targeting.md                    |
| Test purchases locally in Xcode                                                                                 | Step 12                                                      |
| Revenue over time (MRR, churn, trials by month)                                                                 | `swiss-knife:subscription_chart`, in the observability skill |

## Rules that never bend

- If a tool returns `human_action`, relay the required step and pause only the work that depends on it; continue independent authorized work. After the user completes it, call `verify_with`. Do not retry unchanged refusals.
- On restricted connections only, approval codes come from a signed-in human via `swiss-knife:approval_request`; never invent, guess or reuse one. Call it with the gated tool and your arguments, relay what it says, wait for the code, send the same call again with it as `approval`.
- Store-side subscription actions move real money and access. Preview with `dry_run` (the default), show the plan, then send with its `fingerprint`. A refund send, and turning `swiss-knife:subscription_actions_set` on, need a person's approval code on every connection, Codex and Claude Code included: relay the `human_action`, wait for the code, send the same call with it as `approval`. These codes do not come from `swiss-knife:approval_request`. Details in references/customers.md.
- A write that changes what real users are served (an active offering rule, a paywall publish, an experiment pause or resume) answers its plan first; send `confirm: true` with that plan's fingerprint when it matches the user's request.
- `swiss-knife:catalog_push_preview` before every `swiss-knife:catalog_push`; pass its `fingerprint` as `rehearsed`. Change anything after the preview and you preview again.
- Never push a price you have not matched to a store price point (Step 5). A price is an integer in USD micros: `4_990_000` is $4.99.
- Enable `swiss-knife:catalog_push_set` through MCP when the user authorized store setup and the connection has admin access. Never use `activate` or `submit_for_review` unless the user asked for that.
- `swiss-knife:offering_create` is not idempotent. List first.
- In ChatGPT the model reads with `swiss-knife:entitlement_list`, `swiss-knife:offering_list`, `swiss-knife:offering_get` and `swiss-knife:store_events_health`. Open `swiss-knife:dashboard_open` for the in-chat catalogue, offerings, packages and available ledger views. A person uses the app’s offered controls with their permissions and required approval; a model cannot call hidden write tools. For an operation the app does not offer, relay its `human_action` and use a separate connected Codex or Claude Code session.
- In Codex and Claude Code, use delegated write access as described above. Do not request an approval code proactively.

## Checklist

```
- [ ] 1. App found; existing products/entitlements/offerings listed
- [ ] 2. Config created (setup macro or manual chain)
- [ ] 3. Paywall copy/config changed (draft on a served offering), preview_url shown
- [ ] 4. One offering is current, packages in order
- [ ] 5. Catalogue read, draft saved (a new product: created true)
- [ ] 6. Price matched to a store price point
- [ ] 7. catalog_push_preview read, blockers handled
- [ ] 8. Push approved by a human and sent
- [ ] 9. catalog_push_status finished
- [ ] 10. Store checked
```

## Steps

1. **Look first.** `swiss-knife:app_list` for the `app_project_id`. Then `swiss-knife:product_list`, `swiss-knife:entitlement_list` (each entitlement carries its `products`), `swiss-knife:offering_list` (`templates: true` lists the paywall templates, so `template_id` is discovered, not guessed) and `swiss-knife:offering_get` (`preview_url`, `share_url`, `isCurrent` and `config_schema`, the shape `config` is validated against). Reuse what exists. An existing SKU or key that is ARCHIVED comes back as `conflict`: restore it with `swiss-knife:product_unarchive` or `swiss-knife:entitlement_unarchive`.
2. **Create the config.** Default: one `swiss-knife:monetization_setup` call with `products`, `entitlements` (`grants_product_ids` are the SKUs) and `offering` (with `packages` and `set_current`), always with an `idempotency_key`. It is atomic: on error nothing exists, fix the cause and resend. The same key and request replays (`replayed: true`); the same key with a different request is a `conflict`. If the app already has part of it, use the manual chain instead: `swiss-knife:product_create`, `swiss-knife:entitlement_create`, `swiss-knife:entitlement_attach_product`, `swiss-knife:offering_create` (pass `slug`), `swiss-knife:package_add`. Entitlement keys are `[a-z0-9_]+`; `duration_days` 0 means lifetime.
3. **Paywall copy and config.** On an offering no real user is served yet, change it with `swiss-knife:offering_update`: `config_patch` (a JSON merge patch checked against `config_schema`; field errors come back as `details.field_errors` and nothing changes) instead of rewriting `config`. On an offering users are served (the current one, an offering rule's target or an experiment's variant), stage the change as a draft and publish it on purpose: `swiss-knife:paywall_draft_save`, then `swiss-knife:paywall_publish` (references/paywalls-and-targeting.md, which also covers AI redesigns and undo). Change what a package prints with `swiss-knife:package_update` (`label`, `price_label`, `badge_text`, `is_highlighted`, `position`); on a served offering that reaches real users at once: state that effect and proceed when the target and change match the user's request. Give the user the `preview_url` to look at.
4. **Make it current.** `swiss-knife:offering_set_current` with `dry_run: true` first, read `plan.demoted_offering_ids`, then the real call. Retire an offering with `swiss-knife:offering_archive`; the current one is refused, make another current first. Reorder with `swiss-knife:package_reorder` (the full list of package ids). `swiss-knife:offering_verify` checks what an offering serves (packages, products, store state, entitlements) before you make it current.
5. **Read the catalogue.** `swiss-knife:catalog_list`. Stale `lastSyncedAt`: `swiss-knife:catalog_sync_now` (answers STARTED; a second call within a minute is `conflict`), then read again. Save the draft with `swiss-knife:catalog_product_save` (`platform`, `product_id`, `base_price_usd_micros`, `localizations`) and read its `rejected` array: fields a store has fixed (product id, type, group, billing period) are refused by name, the remedy is a new product. **A new product** (a SKU from `swiss-knife:product_create` that no store holds yet) is drafted by the same call from its config row and answers `created: true`. While no store has accepted it you may set `kind`, `group_key` (the App Store subscription group's reference name: required when the app has no single group, and a new name creates the group), `group_level` and `billing_period`. On Google Play the listing must include the app's default language, and a one-time product needs a description.
6. **Check the price.** `swiss-knife:catalog_price_points` with `parent_external_id` from `swiss-knife:catalog_list` (`apple.subscriptionExternalId` or `apple.iapExternalId`) shows the bands already resolved. Empty, or no band at the price: `swiss-knife:catalog_price_preview` shows what the amount snaps to in every territory. If the snap differs from what the user asked for, tell them the real price and ask; do not push the original. A new Google Play product is priced from its USD base price in every region Google converts it to; the create step's summary in the preview says so.
7. **Rehearse.** `swiss-knife:catalog_push_preview` with the same `activate` / `submit_for_review` you intend. Read `refusals`, `readiness.blockers`, `summary.requestCount` (an Apple subscription price is about 175 requests) and `disarmed`. A plan with `blocked: true` or `readiness.canPush: false` is not pushed.
8. **Push.** If `disarmed: true`, enable `swiss-knife:catalog_push_set` when the user authorized store setup. Call `swiss-knife:catalog_push` with the preview's `rehearsed` fingerprint. Codex and Claude Code omit `approval`; only a restricted connection's actual refusal uses `swiss-knife:approval_request`. The answer means queued, not done. A new App Store product whose draft named no price point is created without a price: once that push finishes, `swiss-knife:catalog_sync_now`, then a second preview and push set its price.
9. **Follow it.** `swiss-knife:catalog_push_status` with the `push_id` until every step is done; for a failed step read references/pitfalls.md. Before sending a product to App Review, `swiss-knife:catalog_submission_plan`. Its `missing_review_screenshot` blocker is fixed with `swiss-knife:catalog_review_screenshot_upload` (`product_id`; a PNG or JPEG of what the purchase unlocks as `bytes_base64`, or an `upload_key` from `swiss-knife:media_upload` step `presign`). Apple must already hold the product, and the catalogue switch must be on. Then sync and read the plan again.
10. **Verify.** `swiss-knife:catalog_sync_now`, wait, `swiss-knife:catalog_list`: the store's own version shows the product, its state and the price per territory. If it does not, say so; never report success from the push status.
11. **Is revenue data arriving?** `swiss-knife:store_events_health` (`view` health by default) shows whether Apple server notifications, Google RTDN and RevenueCat webhooks are connected, when the last one arrived, failures in the last 7 days, the URL a person pastes into App Store Connect or Pub/Sub, and the Sales and Trends ingestion state. Page the inbound event log with its `status`, `platform` or `app_user_id` filters when a purchase did not show up. One-time purchases and consumables are in `swiss-knife:store_transaction_list` (`include_balances` adds consumable balances; refunded rows are out of revenue). A webhook that is not connected is a `human_action` for the user to paste a URL: relay it, do not call it fixed. RevenueCat's row also says which Authorization secret the webhook takes (`authorization`). When a person sets up the RevenueCat webhook, give the app its own secret: `swiss-knife:revenuecat_forward_set` with action webhook_secret (admin) returns it once as `authorization_header_value`, with the webhook URL and a `human_action` for the person to save it as the webhook's Authorization header value in RevenueCat; relay both and confirm with `store_events_health` (`authorization` app_secret, a fresh `last_received_at`). The value cannot be read again, and calling the action again replaces it. On an app that already receives RevenueCat events, minting stops them (401) until the new value is saved, and RevenueCat retries a refused delivery only for a few hours, so mint only while that person can save it right away; if events were missed, `swiss-knife:revenuecat_import` `action` `run` re-reads RevenueCat's state.
12. **Test purchases locally (iOS).** `swiss-knife:storekit_config_export` builds an Xcode StoreKit configuration (`.storekit`) from the catalogue, with no store call: write `storekit` as-is to `fileName` in the Xcode project, and name it as the scheme's StoreKit Configuration (Edit Scheme, Run, Options). Read `warnings` (facts the catalogue lacked) and `skipped` (products that are not App Store products) and tell the user about them.

## When it breaks

Full table in references/pitfalls.md (read when any step fails or a refusal appears). For a customer action or a paywall draft, rule or experiment, the reference for that job has its own list.

- `first_product_requires_binary` -> a human must submit the app's first product together with a binary in App Store Connect. Relay `human_action`, stop. Later products go through MCP.
- `price_point_territory_mismatch` -> wrong price point for the territory or product. Nothing was sent. Resolve again with `swiss-knife:catalog_price_preview`, preview again.
- `product_incomplete_for_submission` -> a screenshot or localization is missing. Fix the draft, preview, push again.
- `product_version_state_invalid` -> product versions are append-only and the target left the editable state. Relay, do not retry.
- `offer_id_already_used` / `offer_id_conflict` -> offer ids are unique across every offer kind. Pick a new id; never adopt the existing one.
- `conflict` from `swiss-knife:offering_create` -> the slug exists. List and reuse; do not create again.
- `fingerprint_mismatch` from a store-side subscription action -> the subscription changed since the preview. Dry run again and show the new plan.

For the app side (reading offerings and entitlements in code) use `setup-sdk`. For builds and review use `ship-ios` / `ship-android`.
