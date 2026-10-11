## Step 3 — Provision or adopt (control-plane tools)

### Contents

- 1. Workspace
- 2. App
- 3. Project token
- 4. Starter offering
- 5. Crash-alert rule

Run these in order. Capture each result; you need specific fields downstream.
**Only call a `*_create` tool for something the user chose to create.**

1. **Workspace.**
   - _Adopting_ → use the chosen workspace's `id`. Call nothing.
   - _Creating_ → `workspace_create({ name })` → keep `workspaceId`.
2. **App.**
   - _Adopting_ → use the chosen app's `id` as `appId`. **Do not call
     `app_create`.** Then recover the SDK key you need for
     `EXPO_PUBLIC_SWISSKNIFE_API_KEY`:

     ```
     app_reveal_key({ app_project_id: appId })
     ```

     → returns `{ appId, sdkKey, keyPrefix, platform }`. `sdkKey` is the same
     plaintext key `app_create` shows once; this is the supported way to get it
     back for an app you did not just create. It requires **owner or admin** of
     that app's workspace (checked live, on every call).
     - If it fails with a **forbidden / permission** error, you are a member or
       viewer there. Do **not** work around it by creating a duplicate app. Say so
       plainly: _"You're a member of this workspace, not an admin — ask an owner
       or admin to run `app_reveal_key` for this app (or to grant you admin), and
       I'll finish the wiring."_ Then offer the honest alternatives: pick a
       different workspace, or create a new app in one they own.

   - _Creating_ → `app_create({ workspace_id, name, platform, bundleId })`
     (for `both`, pass `iosBundleId` + `androidBundleId` instead of `bundleId`).
     - **Capture `sdkKey`** from the result — this is the plaintext SDK API key,
       shown **once**. It goes into `EXPO_PUBLIC_SWISSKNIFE_API_KEY`.
     - Also keep `appId`. (If you ever lose `sdkKey`, recover it later with
       `app_reveal_key({ app_project_id })`.)

3. **Project token.** `app_token_mint({ app_project_id: appId })` — same on both
   paths (a fresh per-project token is correct even for an adopted app; minting
   one does not invalidate any existing token).
   - **Capture `token`** (the `sk_mcp_…` value, shown once → `SWISS_KNIFE_MCP_TOKEN`)
     and **`mcpJson`** (a ready-to-paste `.mcp.json` snippet).
   - The token **expires** (90 days by default; `expires_in_days` takes up to
     365). Tell the user the `expiresAt` date from the result; when it lapses,
     re-mint it (below) and update only `SWISS_KNIFE_MCP_TOKEN`.
   - Also requires **owner or admin** of the workspace — same honest failure
     message as above if it is refused.
4. **Starter offering** (only if they picked "Seed a starter offering"). **On an
   adopted app, check before you seed:** call `offering_list({ app_project_id })`
   (and `product_list`) first. If it already has products or an offering, do
   **not** re-seed — report what's there and move on; duplicating a catalogue is
   worse than an empty one. Seed only into an empty catalogue. A monthly + annual
   "pro" so the paywall/offering isn't empty:

   ```
   monetization_setup({
     app_project_id: appId,
     products: [
       { product_id: "pro_monthly", name: "Pro Monthly", platform: <ios|android>, duration_days: 30,  price: 9.99,  currency: "USD" },
       { product_id: "pro_annual",  name: "Pro Annual",  platform: <ios|android>, duration_days: 365, price: 79.99, currency: "USD" }
     ],
     entitlements: [
       { key: "pro", name: "Pro", grants_product_ids: ["pro_monthly", "pro_annual"] }
     ],
     offering: {
       name: "default",
       packages: [
         { product_id: "pro_monthly" },
         { product_id: "pro_annual", badge_text: "Best value", is_highlighted: true }
       ],
       set_current: true
     }
   })
   ```

   Use one platform value (`ios` for `both` is fine for a starter seed — note to
   the user they can add the Android SKUs later). `set_current: true` is what
   makes `getOfferings().current` non-null.

5. **Crash-alert rule** — only if they ticked **Crash reporting**. Do this
   **silently**: announce it in ONE line ("Added a 'New crash issue' alert
   rule."), do **not** ask. A crash reporter nobody is watching is a log file.

   ```
   error_alert_create({
     app_project_id: appId,
     name: "New crash issue",
     condition: "new_issue"
   })
   ```

   That is the entire call — every other field on the tool is optional. The rule
   fires whenever a never-before-seen crash issue appears, and defaults to a
   60-minute `actionIntervalMinutes` so it cannot spam. (Other conditions exist:
   `regression`, `spike` with `threshold` + `windowMinutes`, and the crash-free
   metric rules `metric_crash_free_users` / `metric_crash_free_sessions` with
   `thresholdWarning` / `thresholdCritical` percentages. `new_issue` is the right
   default for a fresh app with no baseline.)

   **On the notify target: the account holder's own address, never a guess.** On
   an account (OAuth) connection, `workspace_whoami` returns `email` — the address
   of the signed-in user this connection acts as (the `workspace_whoami` answer
   from Step 1 already has it). Pass it as `notifyEmail` and say so in your
   one-line announcement ("Added a 'New crash issue' alert rule; it emails
   you@example.com."). `user_list` / `people_detail` describe the _app's
   end-users_, not the dashboard account — never take an address from them.

   If `workspace_whoami` returned **no `email`** (an `sk_mcp_` token never does:
   it is a credential meant for CI and other machines), omit `notifyEmail` and do
   not invent one — a rule pointed at the wrong inbox is worse than a rule with no
   inbox. Use an address only if the user has already given it to you in this
   conversation. A target can always be added afterwards **through MCP**, not the
   dashboard: `error_alert_list({ app_project_id })` shows the app's rules and
   their `id`, and `error_alert_update({ rule_id, notifyEmail })` sets (or, with
   `null`, clears) the target; `active: false` pauses a rule and
   `error_alert_delete({ rule_id })` removes it. On a re-run, call
   `error_alert_list` first and reuse the existing `new_issue` rule (update its
   `notifyEmail` if it has none) instead of creating a second one.

   Like `app_reveal_key` / `app_token_mint`, this needs **owner or admin** on the
   workspace. If it is refused, say so in one line and carry on — an alert rule is
   a nice-to-have, never a reason to fail the setup.
