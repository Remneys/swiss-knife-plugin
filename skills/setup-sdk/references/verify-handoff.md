## Step 9 — Verify and hand off (be honest)

### Contents

- 1. Typecheck
- 2. You cannot run the app here
- 4. Say which capabilities you wired
- 5. Summarize
- 6. State the analytics outcome just as explicitly

1. **Typecheck** if the project has one (`npm run typecheck`, or `npx tsc --noEmit`
   if there's a tsconfig). Fix anything you introduced. If there's no typecheck
   script, skip — don't invent one.
2. **You cannot run the app here**, and that's expected — the SDK has a **native
   module**, so **Expo Go will NOT work**. Clearly tell the user the real run
   path:
   - Prebuild native projects: `npx expo prebuild` — **unless** Step 5.4 flagged a
     committed, hand-edited `ios/` / `android/`; there, `cd ios && pod install`.
   - Launch a dev client on device/simulator: `npx expo run:ios`
     (or `npx expo run:android`) — first run compiles native, so it's slow.
   - The `[swiss-knife] configured…` log appears in the dev-client console on
     launch. That's the green light.
3. **Say the Session Start line out loud. This is the single sentence that
   prevents a false "the SDK doesn't work" bug hunt**, and it costs you one line:

   > _"Until something in your app calls `Analytics.track(...)`, the dashboard
   > will show exactly one event per launch — `[SwissKnife] Session Start`. That
   > is `Analytics.init` doing its job: `trackingSessionEvents` defaults to
   > `true`, so init emits it, and **nothing else is automatic**."_

   Say it whenever nothing in the project calls `Analytics.track` yet — which is
   the default state of a freshly wired app, and the state that reads as a broken
   integration. Say it **twice as loudly** if Step 7 landed on _"match the
   incumbent — dev sends nothing"_: their dev build is precisely that case, by
   their own choice, and they will otherwise spend an afternoon debugging a
   correct integration. If Step 7 wired the mirror with dev sending enabled, say
   the flip side instead — _"your existing events start arriving as soon as that
   code path runs; a launch that tracks nothing still shows only Session Start."_

   **The same sentence exists for the person, and it costs one more line.** Say it
   whenever Step 6 or Step 8 was skipped:

   > _"Until something calls `Identity.logIn(...)`, every person in the dashboard
   > is an anonymous device and every crash reads 'Affected users: none' — the
   > crash subject is pulled from `Identity`, so no `logIn` means no subject
   > anywhere. And until something calls `Attributes.set(...)`, that
   > person card carries an id and nothing else."_

4. **Say which capabilities you wired**, one line, mapped straight back to the
   Step 2 multi-select — e.g. _"Wired: subscriptions, analytics, crash reporting
   (+ `<SKErrorBoundary>` and a 'New crash issue' alert rule). Not wired:
   performance tracing, install attribution."_ Naming what you did **not** wire
   matters as much as naming what you did — it is how the user confirms the
   wizard honoured their answer, and how they know what to ask for next.
5. **Summarize** crisply, and **be precise about adopted vs created** — the user
   must never be left guessing whether you made a second app. Say
   _"adopted existing app 'Acme' (id …) in workspace 'Acme Inc'"_ or
   _"created app 'Acme' (id …) in a new workspace 'Acme Inc'"_, whether a starter
   offering was seeded (or was already present and left alone), which files you
   wrote vs left untouched, and the two commands above as the next step. Remind
   them the `sdkKey` and MCP `token` were shown once and now live only in `.env`
   (recover the SDK key with `app_reveal_key` if lost). **State the identity
   outcome explicitly**, naming the branch — "branch A: identity wired at
   `<file>`, `logOut()` on sign-out", "branch B: `logIn` wired at `<file>` on
   every app start, and sign-out re-identifies as the new anonymous user", "branch
   C: no server-side user, so everything is device-scoped for now", or "you asked
   to wire identity yourself" — so nobody discovers weeks later that their data is
   anonymous. Add the Step 8 outcome in the same breath, naming the fields:
   _"attached `plan` and `signup_date` to the person; no email or name, as you
   asked."_
6. **State the analytics outcome just as explicitly**, in the same breath as the
   dev-build decision — one line, e.g. _"Analytics: mirrored at
   `src/hooks/useTrack.ts` (88 call sites), sending from dev builds tagged
   `environment: development` — your Firebase/Mixpanel guard is untouched. Expect
   roughly double your current event volume."_ Or _"Analytics: screen views only
   (`Analytics.trackScreen` on the router). Nothing else calls `Analytics.track`
   yet."_ Or _"Analytics: nothing wired — the dashboard will show only
   `[SwissKnife] Session Start` until you add `Analytics.track(...)` calls."_
7. **If you wired branch B and the id can change, put the backend follow-up in
   the summary as an open item, not a footnote.** One line: _"Your backend still
   needs to `POST /api/sdk/users/identify` with `{ previous_user_id, user_id }`
   whenever sign-in changes the id — the client call alone will not move the
   anonymous user's history."_ It is the only part of the setup the wizard cannot
   do for them, and it is the part that loses data if skipped.
8. **If they kept an existing IAP library and you emitted `configureCore`, say
   so plainly** — _"Swiss-Knife is not touching purchases: no StoreKit observer,
   no BillingClient. `SwissKnife.isConfigured()` will report `false`; that's
   correct — use `isCoreConfigured()`."_ Otherwise the first thing they do is
   "fix" it by calling `configure`, which is the race you just avoided.
9. **iOS privacy manifest — say this only if Step 1.6 scored the project as
   shipped.** On a POC it is pure noise; on an app that goes through App Store
   review it is a rejection.

   The SDK ships **no** `PrivacyInfo.xcprivacy` and has no Expo config plugin to
   inject one, so nothing lands in their project automatically — the **host app's**
   manifest is where this gets declared. It must cover:
   - `NSPrivacyAccessedAPICategoryUserDefaults`, reason **`CA92.1`** — the SDK
     keeps session and config state in `UserDefaults`.
   - `NSPrivacyAccessedAPICategoryFileTimestamp`, reason **`C617.1`** — file
     timestamps on its own offline queue and crash payloads, inside the app
     container.
   - The **data types** they actually turned on: crash data, product interaction
     / other usage data, plus any identifier Step 8 attaches (user id, email).
   - **Only with install attribution ticked**: the SDK requests ATT and reads the
     IDFA, so the manifest must set `NSPrivacyTracking` true, list their origin in
     `NSPrivacyTrackingDomains`, and `Info.plist` needs
     `NSUserTrackingUsageDescription`.

   On managed Expo the file is generated from `expo.ios.privacyManifests` in
   `app.json` (SDK 51+); on a bare app it is a resource in the Xcode target. State
   it as an open item and offer to write it — don't silently edit a manifest that
   probably already carries their own entries.
