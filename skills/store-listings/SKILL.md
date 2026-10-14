---
name: store-listings
description: Writes, translates and publishes store listing text, screenshots and the Apple age rating on the App Store and Google Play through the Swiss-Knife MCP server, then checks what the store answered. Use when the user says "update my store listing", "translate my Play description into Spanish", "upload new screenshots", "import my fastlane metadata", "fix the age rating". Not for submitting a version to review (use submit-release), builds (ship-ios, ship-android) or in-app products and prices (use monetization).
---

# Store listings, screenshots and age rating

Success: the push status says `applied` with `pending: false`, and the store-side read (`swiss-knife:listing_get` published values, `swiss-knife:media_list`, `swiss-knife:age_rating_get` published answers) after a store sync matches what you meant to publish. A queued push is not done.

## Rules that never bend

- Any tool answer with `human_action` (or a refusal that needs an approval): relay `who`, `what`, `url`, `why` to the user verbatim, then STOP. No retry, no polling, no workaround. When they say it is done, call the `verify_with` tool.
- Approval codes come from a signed-in human. Call `swiss-knife:approval_request` with the gated tool and your arguments, relay what it says, wait for the code, and send the same call again with it as `approval`. Never invent or reuse a code.
- Drafts (`swiss-knife:listing_update`, `swiss-knife:listing_translate`, `swiss-knife:listing_import`, `swiss-knife:media_upload`, `swiss-knife:age_rating_update`) reach no store. Only the three pushes publish. Every push runs as a dry run first (`confirm` false, the default); read the plan, then repeat with `confirm: true`.
- Never answer a `conflict` yourself. No tool resolves one; the store's text and the draft differ and a human picks which wins.
- Google Play language codes are Play's own, not Apple's. Never send a bare `es`, `ru`, `pt` or `zh` to Play (references/pitfalls.md has the list).
- Pushing a listing, screenshots or age rating while an iOS version waits for review needs an approval code; a version in review refuses them, and a live version's screenshots are frozen.
- In ChatGPT only the reads exist (`swiss-knife:listing_get`, `swiss-knife:media_list`, `swiss-knife:media_validate`, `swiss-knife:age_rating_get`, the push status tools, `swiss-knife:store_overview`): drafts and pushes are refused there. Read and explain, say that editing and publishing are done from Claude Code, not from this chat, and do not look for another way.
- In Codex, as in Claude Code, you call the write tools yourself, including `swiss-knife:build_start` and `swiss-knife:build_retry`, which queue a build at once; a gated write (a store submit, a release, a signing, credential or catalogue change) stops and waits for a human approval code an admin issues, so call `swiss-knife:approval_request`, relay what it says and stop. Never make up a code.

## Checklist

```
- [ ] 1. App, platform and locales settled (Play codes valid)
- [ ] 2. listing_get read; limits known per field
- [ ] 3. Drafts written (update / translate / import) and over-limit issues fixed
- [ ] 4. Media validated (media_validate has no errors)
- [ ] 5. Dry run of each push read
- [ ] 6. Confirmed push sent, pushId kept
- [ ] 7. *_push_status polled until pending is false
- [ ] 8. Verified against the store
```

## Steps

1. **Scope.** `swiss-knife:workspace_whoami`, `swiss-knife:app_list` for the `app_project_id`. Settle the platform (`ios` or `android`) and the locales. On Android map every locale to a Play code first.
2. **Read.** `swiss-knife:listing_get` (add `locale` to narrow). It returns published values beside drafts and the limit table. Fields differ by store: iOS has subtitle, keywords and promotional text; Android has short description and video URL. Writing a field the platform lacks is an error.
3. **Draft text.** `swiss-knife:listing_update` per locale with only the fields to change. A `null` discards that draft (reverts to the published value); it never blanks the store. Over-limit copy still saves and the issue shows at the push dry run, so check lengths against the limit table yourself. Many locales from one source: `swiss-knife:listing_translate` (`from_locale`, `to_locales`; costs money per locale). A fastlane `metadata/` tree: `swiss-knife:listing_import` with `dry_run: true` first; `swiss-knife:listing_export` writes one back out.
4. **Draft media.** `swiss-knife:media_import` (`dry_run: true` first) or `swiss-knife:media_upload` (presign, PUT, commit), then `swiss-knife:media_list`. One master per device class; `swiss-knife:media_derive` (`confirm` false is the plan) only when per-size control is wanted. `swiss-knife:media_validate` must show no `error` issue before pushing.
5. **Age rating (Apple only).** `swiss-knife:age_rating_get` for the questions and allowed values, `swiss-knife:age_rating_update` for answers (answer `ageRatingOverrideV2`, never `ageRatingOverride`). Play's content rating is a console-only questionnaire; tell the user so.
6. **Dry run, then push.** `swiss-knife:listing_push`, `swiss-knife:media_push`, `swiss-knife:age_rating_push`, each with `confirm` false first. Listing targets: "app_info" (name, subtitle), `version` (needs `version_id`), `play_listing`, `play_release_notes` (needs `version_code`). Media targets: `ios_screenshots` (needs `version_id`), `play_images`. Fix everything the plan lists under issues, invalid or conflict locales, then repeat with `confirm: true`. Keep the returned `pushId`.
7. **Check the answer.** Poll `swiss-knife:listing_push_status`, `swiss-knife:media_push_status` or `swiss-knife:age_rating_push_status` with the `pushId` until `pending` is false. Do this after EVERY confirmed push. `applied` is success; `failed` or `conflict` carries a reason (below). An iOS listing push is one mutation per locale and can land partly: read `appliedLocales`.
8. **Verify.** The push status is our own record. Run `swiss-knife:store_sync_now` (the platform; `mode: "full"` if the mirror looks incomplete), then `swiss-knife:listing_get`, `swiss-knife:media_list` or `swiss-knife:age_rating_get` and confirm the published side equals what you pushed. If it does not, say so; never report success from the push status alone.

## When it breaks

Full table, limits and the Play code list: read references/pitfalls.md when any step fails.

- `unsupported_locale` (dry-run issue) or `play_unsupported_language` -> not a Play code. Use the suggested one (`es` -> `es-ES`, `es-419` or `es-US`, ask which; `ru` -> `ru-RU`), redraft under it with `swiss-knife:listing_update`, dry run again.
- "listing_version_not_editable" / "media_version_not_editable" -> the version entered review or went live. Do not retry. A live version accepts only promotional text, and no screenshots; name and subtitle go through "app_info". Offer the user: wait for the review result, or prepare the next version.
- "media_asset_rejected" / `play_image_rejected` -> final, the same bytes fail again. Read the `swiss-knife:media_validate` issue (wrong size, format or alpha), fix the image, `swiss-knife:media_upload` again.
- "listing_push_conflict" / status `conflict` -> relay the `human_action` verbatim and stop. After they resolve it: `swiss-knife:listing_get`, then the dry run again.
- "listing_field_rejected" / `play_invalid_listing` -> the store refused the text; fix it with `swiss-knife:listing_update`, push again.
- Refusal that needs an approval (version waiting for review) -> `swiss-knife:approval_request`, relay, wait for the code, repeat the call with `approval`.
