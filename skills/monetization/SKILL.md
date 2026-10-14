---
name: monetization
description: Sets up an app's products, entitlements, offerings (the paywall the SDK serves) and packages through the Swiss-Knife MCP server, then pushes the priced catalogue to App Store Connect or Google Play and checks the store. Use when the user says "add a subscription", "create a monthly and yearly plan", "set up my paywall", "add an entitlement", "change the price", "push my products to the store". Not for SDK code (use setup-sdk) or builds and review submission (use ship-ios, ship-android).
---

# Set up monetization and push it to the store

Success: the store itself lists the product with the intended price (`swiss-knife:catalog_sync_now`, then `swiss-knife:catalog_list`), and `swiss-knife:offering_get` shows the current offering with its packages. Our own push status is not proof.

Two layers, do not mix them up:

- Config the SDK serves: products, entitlements, offerings, packages (`swiss-knife:product_create` and friends). Writes only our database.
- The store catalogue: `swiss-knife:catalog_*`. Only `swiss-knife:catalog_push` can write to a store.

A config product and a catalogue product meet by platform plus store SKU (`product_id`), so use the same SKU in both.

An offering is the paywall. Its `template_id` and `config` are the only paywall controls in MCP; there is no tool that designs or previews paywall screens, so do not promise that. Reading the offering in the app is `setup-sdk` territory.

## Rules that never bend

- Any tool answer with `human_action` (or a refusal that needs an approval): relay `who`, `what`, `url`, `why` to the user verbatim, then STOP. No retry, no polling, no workaround. When they say it is done, call the `verify_with` tool.
- Approval codes come from a signed-in human via `swiss-knife:approval_request`; never invent, guess or reuse one. Call it with the gated tool and your arguments, relay what it says, wait for the code, send the same call again with it as `approval`.
- `swiss-knife:catalog_push_preview` before every `swiss-knife:catalog_push`; pass its `fingerprint` as `rehearsed`. Change anything after the preview and you preview again.
- Never push a price you have not matched to a store price point (Step 5). A price is an integer in USD micros: `4_990_000` is $4.99.
- Never turn `swiss-knife:catalog_push_set` on without a human's approval code. Never use `activate` or `submit_for_review` unless the user asked for that.
- `swiss-knife:offering_create` is not idempotent. List first.
- In ChatGPT only the reads exist: `swiss-knife:entitlement_list`, `swiss-knife:offering_list`, `swiss-knife:offering_get` and `swiss-knife:store_events_health`. The one-time purchase ledger and consumable balances are Claude Code's, not this chat's. Creating or changing products, offerings, packages and the store catalogue is refused there: say it is done from Claude Code, not from this chat, and do not look for another way.
- In Codex, as in Claude Code, you call the write tools yourself, including `swiss-knife:build_start` and `swiss-knife:build_retry`, which queue a build at once; a gated write (a store submit, a release, a signing, credential or catalogue change) stops and waits for a human approval code an admin issues, so call `swiss-knife:approval_request`, relay what it says and stop. Never make up a code.

## Checklist

```
- [ ] 1. App found; existing products/entitlements/offerings listed
- [ ] 2. Config created (setup macro or manual chain)
- [ ] 3. Paywall copy/config changed with config_patch / package_update, preview_url shown
- [ ] 4. One offering is current, packages in order
- [ ] 5. Catalogue read, draft saved
- [ ] 6. Price matched to a store price point
- [ ] 7. catalog_push_preview read, blockers handled
- [ ] 8. Push approved by a human and sent
- [ ] 9. catalog_push_status finished
- [ ] 10. Store checked
```

## Steps

1. **Look first.** `swiss-knife:app_list` for the `app_project_id`. Then `swiss-knife:product_list`, `swiss-knife:entitlement_list` (each entitlement carries its `products`), `swiss-knife:offering_list` (`templates: true` lists the paywall templates, so `template_id` is discovered, not guessed) and `swiss-knife:offering_get` (`preview_url`, `share_url`, `isCurrent` and `config_schema`, the shape `config` is validated against). Reuse what exists. An existing SKU or key that is ARCHIVED comes back as `conflict`: restore it with `swiss-knife:product_unarchive` or `swiss-knife:entitlement_unarchive`.
2. **Create the config.** Default: one `swiss-knife:monetization_setup` call with `products`, `entitlements` (`grants_product_ids` are the SKUs) and `offering` (with `packages` and `set_current`), always with an `idempotency_key`. It is atomic: on error nothing exists, fix the cause and resend. The same key and request replays (`replayed: true`); the same key with a different request is a `conflict`. If the app already has part of it, use the manual chain instead: `swiss-knife:product_create`, `swiss-knife:entitlement_create`, `swiss-knife:entitlement_attach_product`, `swiss-knife:offering_create` (pass `slug`), `swiss-knife:package_add`. Entitlement keys are `[a-z0-9_]+`; `duration_days` 0 means lifetime.
3. **Paywall copy and config.** Change an offering with `swiss-knife:offering_update`: `config_patch` (a JSON merge patch checked against `config_schema`; field errors come back as `details.field_errors` and nothing changes) instead of rewriting `config`. Change what a package prints with `swiss-knife:package_update` (`label`, `price_label`, `badge_text`, `is_highlighted`, `position`). On the current offering these changes reach real users at once: say so and confirm before editing it. Give the user the `preview_url` to look at.
4. **Make it current.** `swiss-knife:offering_set_current` with `dry_run: true` first, read `plan.demoted_offering_ids`, then the real call. Retire an offering with `swiss-knife:offering_archive`; the current one is refused, make another current first. Reorder with `swiss-knife:package_reorder` (the full list of package ids).
5. **Read the catalogue.** `swiss-knife:catalog_list`. Stale `lastSyncedAt`: `swiss-knife:catalog_sync_now` (answers STARTED; a second call within a minute is `conflict`), then read again. Save the draft with `swiss-knife:catalog_product_save` (`base_price_usd_micros`, `localizations`) and read its `rejected` array: locked fields are refused by name, the remedy is a new product.
6. **Check the price.** `swiss-knife:catalog_price_points` with `parent_external_id` from `swiss-knife:catalog_list` (`apple.subscriptionExternalId` or `apple.iapExternalId`) shows the bands already resolved. Empty, or no band at the price: `swiss-knife:catalog_price_preview` shows what the amount snaps to in every territory. If the snap differs from what the user asked for, tell them the real price and ask; do not push the original.
7. **Rehearse.** `swiss-knife:catalog_push_preview` with the same `activate` / `submit_for_review` you intend. Read `refusals`, `readiness.blockers`, `summary.requestCount` (an Apple subscription price is about 175 requests) and `disarmed`. A plan with `blocked: true` or `readiness.canPush: false` is not pushed.
8. **Push.** `disarmed: true` means pushes are off: `swiss-knife:approval_request` for `swiss-knife:catalog_push_set`, wait for the code, send it. Then `swiss-knife:approval_request` for `swiss-knife:catalog_push` (the code is bound to this exact plan), and send `swiss-knife:catalog_push` with `rehearsed` and `approval`. The answer means queued, not done.
9. **Follow it.** `swiss-knife:catalog_push_status` with the `push_id` until every step is done; for a failed step read references/pitfalls.md. Before sending a product to App Review, `swiss-knife:catalog_submission_plan`.
10. **Verify.** `swiss-knife:catalog_sync_now`, wait, `swiss-knife:catalog_list`: the store's own version shows the product, its state and the price per territory. If it does not, say so; never report success from the push status.
11. **Is revenue data arriving?** `swiss-knife:store_events_health` (`view` health by default) shows whether Apple server notifications, Google RTDN and RevenueCat webhooks are connected, when the last one arrived, failures in the last 7 days, the URL a person pastes into App Store Connect or Pub/Sub, and the Sales and Trends ingestion state. Page the inbound event log with its `status`, `platform` or `app_user_id` filters when a purchase did not show up. One-time purchases and consumables are in `swiss-knife:store_transaction_list` (`include_balances` adds consumable balances; refunded rows are out of revenue). A webhook that is not connected is a `human_action` for the user to paste a URL: relay it, do not call it fixed.

## When it breaks

Full table in references/pitfalls.md (read when any step fails or a refusal appears).

- `first_product_requires_binary` -> a human must submit the app's first product together with a binary in App Store Connect. Relay `human_action`, stop. Later products go through MCP.
- `price_point_territory_mismatch` -> wrong price point for the territory or product. Nothing was sent. Resolve again with `swiss-knife:catalog_price_preview`, preview again.
- `product_incomplete_for_submission` -> a screenshot or localization is missing. Fix the draft, preview, push again.
- `product_version_state_invalid` -> product versions are append-only and the target left the editable state. Relay, do not retry.
- `offer_id_already_used` / `offer_id_conflict` -> offer ids are unique across every offer kind. Pick a new id; never adopt the existing one.
- `conflict` from `swiss-knife:offering_create` -> the slug exists. List and reuse; do not create again.

For the app side (reading offerings and entitlements in code) use `setup-sdk`. For builds and review use `ship-ios` / `ship-android`.
