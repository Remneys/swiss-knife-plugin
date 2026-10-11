## Step 1 — Preflight: read the project, then discover what already exists

### Contents

- 1. Confirm the control plane is connected
- 2. Detect the project stack
- 3. Read the project's identity signals
- 4. Take stock of the local wiring
- 5. Enumerate the account (the adoption search)
- 6. Score the project's stage — from repo evidence, silently. Do NOT ask

**This step writes nothing.** You are building a picture of (a) the local project
and (b) what the user's Swiss-Knife account **already contains**, so that Step 2
offers real choices instead of blindly provisioning a duplicate.

> **The cardinal rule of this wizard: DISCOVER, THEN ADOPT, AND ONLY THEN CREATE.**
> Most users arrive here already having something — they signed up on the
> dashboard (which creates a workspace **and** an app), or a teammate did, or they
> ran this wizard before. Calling `workspace_create` / `app_create` unconditionally
> silently forks their SDK data across two app projects, and the split is invisible
> until their dashboard shows half the events. Creating is the **fallback**, never
> the default.

1. **Confirm the control plane is connected.** Call `workspace_whoami`.
   - If it errors with an auth / unauthorized message, the account MCP connection
     isn't authorized yet. Tell the user: _"I need to connect to Swiss-Knife
     first — approve the browser sign-in that just opened (or run
     `/mcp` → swiss-knife → Authenticate)."_ This plugin already added the
     `swiss-knife` MCP server, so authorizing is a one-time browser step. Then
     re-run `workspace_whoami`.
   - On success it returns your identity and the `workspaces` you belong to.
2. **Detect the project stack** in the current directory (use Read / `ls`, don't
   guess). Check in this order and take the FIRST match:
   - `app.json` / `app.config.js` with an `expo` key, or `expo` in
     `package.json` dependencies → **Expo** (the fully-automated path).
   - `package.json` with `react-native` but no `expo` → **bare React Native**
     (mostly automated; note the manual `pod install`).
   - No `package.json` (or one with neither `expo` nor `react-native`) **and** an
     Xcode project (`*.xcodeproj` / `*.xcworkspace`) or a `Package.swift` →
     **native iOS** → the `swiss-knife:setup-native` skill (see the pointer at the
     end of this file).
   - A `settings.gradle` / `build.gradle` (`*.gradle.kts` or Groovy, with Kotlin)
     and no React Native → **native Android** → same sibling skill.
   - None of the above → **web / unknown** → same sibling skill's _Other stacks_.

   All stacks run Steps 1–4 (provisioning + `.mcp.json` config) first, and all
   stacks finish with Step 6 (identity), Step 8 (the user's data) and Step 9
   (verify) — those are stack-agnostic. Step 5 (SDK integration) branches by stack,
   and Step 7 (the analytics boundary) applies to the React-Native stacks only.

3. **Read the project's identity signals** — you need these both to pre-fill the
   questions AND to match against existing apps:
   - **Bundle id** — `expo.ios.bundleIdentifier` / `expo.android.package` from
     `app.json` (native iOS: `PRODUCT_BUNDLE_IDENTIFIER`; native Android:
     `applicationId` / `namespace` in `build.gradle`).
   - **App name** — `expo.name`, else `package.json` `name`, else the directory
     name.

4. **Take stock of the local wiring** (idempotency — _observe_, don't act yet).
   Build a **local ledger** of what is already wired, and carry it into Step 4:
   - `.mcp.json` — is there a `swiss-knife` server entry?
   - `package.json` — is `@app-swiss-knife/react-native-sdk` a dependency? (native:
     the SPM / Gradle dependency instead.)
   - `src/swiss-knife.ts` (native: `SwissKnifeConfig.swift` / `secrets.properties`)
     — does it exist?
   - Root component — is `initSwissKnife()` / `SwissKnife.configure` /
     `SwissKnife.configureCore` actually called? **Grep for `Purchases.configure`
     and `Purchases.configureCore` too** — those are the same functions under the
     RevenueCat-compatible names, and anything wired before SDK 0.3.0 spells it
     that way. Note **which** bootstrap it is — `configure` (core + store) and
     `configureCore` (core only) are not interchangeable, and Step 5 needs to
     know which one is already there.
   - **A competing IAP library** — grep `package.json` for
     `react-native-purchases`, `react-native-iap`, `expo-in-app-purchases`,
     `adapty`, `qonversion`, `glassfy`. If one is present, Step 2's store-conflict
     warning is not hypothetical for this project; carry the finding forward.
   - **An incumbent crash reporter** — grep `package.json` for
     `@sentry/react-native`, `@react-native-firebase/crashlytics`,
     `@bugsnag/react-native`, `@datadog/mobile-react-native`, and cross-check
     `ios/Podfile` (`Sentry`, `Firebase/Crashlytics`, `Bugsnag`) plus
     `google-services.json` / `GoogleService-Info.plist` (Crashlytics ships with
     Firebase). If one is present, Step 2 leaves **Crash reporting unticked** and
     explains why — see the coexistence note there.
   - **Capabilities already wired** — grep for `Analytics.init(`,
     `CrashReporting.init(`, `Performance.init(`, `SKErrorBoundary`, and
     `Attribution.`. Step 5 adds only the ones that are missing; it never removes
     or rewrites a capability someone already wired by hand.
   - **Identity** — is `Identity.logIn(` called anywhere (grep the project)?
     Record **where**: a call at an app-start / bootstrap site and a call only
     inside a sign-in success handler mean very different things (Step 6 branches
     B and A). A sign-in-only call site in an app that has anonymous server users
     is a gap to repair, not a box already ticked. Also note whether
     `Identity.logOut(` is called on sign-out — in branch B that is usually the
     wrong call.
   - **What the app already KNOWS about the user at that boundary.** Open the
     user / session type it hands you (`User`, `Profile`, `Session`, the
     `useAuth()` return) and **write down the field names**. Step 8 attaches some
     of them to the person, and it may only ever name fields it has actually
     seen. Note in particular which fields are **credentials** —
     `accessToken`, `refreshToken`, `idToken`, `apiKey`, `password*`, `secret*`,
     `session` — Step 8's hard rule is that those are never sent, ever.
   - **An incumbent people / user-properties API** — the app may already be
     attaching these somewhere. Grep for `getPeople().set`, `.people.set`,
     `setUserProperties(`, `setUserProperty(`, `identify(`, `setUserId(`,
     `Sentry.setUser(`, `setUserAttributes(`. If you find one, Step 8 mirrors
     **the fields that call already sends** rather than inventing a new set —
     the app owner has already made this decision once.
   - **Incumbent analytics — and, more importantly, its central `track` wrapper.**
     Almost every real app already sends events somewhere, and that wrapper is the
     seam Step 7 wires into, so find it now. Grep `package.json` for at least
     `mixpanel-react-native`, `@amplitude/analytics-react-native`,
     `posthog-react-native`, `@react-native-firebase/analytics`,
     `@segment/analytics-react-native` — and for the ones that show up almost as
     often: `@rudderstack/rudder-sdk-react-native`, `@datadog/mobile-react-native`,
     `countly-sdk-react-native-bridge`, `@aptabase/react-native`,
     `@heap/react-native-heap`, plus the legacy names still sitting in older
     manifests (`@amplitude/react-native`, `react-native-mixpanel`,
     `expo-firebase-analytics`, `expo-analytics-amplitude`, `appcenter-analytics`).
     Two near-misses **not** to count: `@sentry/react-native` / `bugsnag` are crash
     reporters, not event analytics; and AppsFlyer / Adjust / Branch / Singular are
     attribution SDKs (Step 2 already handles those) even though they expose a
     `logEvent`.
   - **Then find the wrapper — the dependency alone tells you nothing.** Grep for
     the import specifier of whatever you found (`from "mixpanel-react-native"`,
     `@react-native-firebase/analytics`, …) and **count the files that import it**:
     - **1–3 files** → that IS the seam. Open them; whatever they export
       (`useTrack`, `track`, `trackEvent`, `logEvent`, `capture`) is the wrapper
       every call site goes through.
     - **Many files** → there is no central wrapper; the calls are scattered. Record
       that — Step 7 handles it differently and must not offer a one-line fix.

     Cross-check by filename (`analytics.*`, `tracking.*`, `telemetry.*`,
     `use-track*` / `useTrack*`, `events.*` under `src/`, `lib/`, `hooks/`,
     `utils/`, `services/`), then **count the call sites** of the exported symbol.
     The number is what makes Step 7's question concrete — _"88 `track()` calls
     across 35 files, all through `useTrack`"_ is a decision; _"you use Mixpanel"_
     is not.

   - **Read the wrapper's body for a dev guard — this is the trap.** Look for an
     early return keyed on `__DEV__`, `isDev`, `process.env.NODE_ENV`,
     `Constants.appOwnership`, `Updates.channel`, `!isProduction`, or an
     `EXPO_PUBLIC_*` flag, and note **where it sits relative to the fan-out
     calls**. `if (isDev) return;` at the top of the wrapper is a very common and
     perfectly sensible pattern (don't pollute production analytics while
     debugging) — and it means a correctly-built Swiss-Knife bridge still sends
     nothing from the dev build the user is about to test on. Carry the finding
     into Step 7 question 2; do not let them discover it themselves.
   - `.env` — are `EXPO_PUBLIC_SWISSKNIFE_API_KEY` / `SWISS_KNIFE_MCP_TOKEN` set to
     **real** values, or still `…_replace_me` placeholders?

   If `.env` holds a real `sk_live_…` key, note its **prefix only** (the leading
   `sk_live_` + first few chars). Never print or echo the full key. That prefix
   matches `keyPrefix` in `app_list` output and identifies the app this project is
   **already bound to** — an exact, zero-ambiguity match.

   A non-empty ledger does **not** mean "abort". It means Step 4 must
   **reconcile** rather than overwrite — see _Reconciling an existing setup_.

5. **Enumerate the account (the adoption search).** Read-only, cheap, always do it:
   - `workspace_list` → the workspaces the user can act on.
   - **Zero workspaces** → nothing exists to adopt. Skip straight to the
     brand-new-user path: no adoption questions at all (see Step 2).
   - Otherwise call `app_list({ workspace_id })` for each workspace (cap at the
     first ~5 workspaces so you don't fan out absurdly; if there are more, ask the
     user which workspace first and list only that one).
   - **Rank every app you found** against the local signals, best first:
     1. **Exact — key prefix.** The app's `keyPrefix` matches the `sk_live_…`
        prefix in the local `.env`. This project IS that app. Adopt it; do not
        even offer "create new" as the recommended option.
     2. **Exact — bundle id.** The local bundle id equals the app's `bundleId`,
        `iosBundleId`, or `androidBundleId` (compare case-insensitively; check all
        three — a cross-platform app stores per-platform ids and leaves `bundleId`
        null).
     3. **Strong — name.** App `name` matches the detected app name
        (case-insensitive) and the platform is compatible.
     4. **Weak — unfinished onboarding.** `setupCompletedStep` < 3 means someone
        started this app in the dashboard and never finished. Very likely the
        intended target, especially if it is the workspace's only app.
     5. **None.** No signal matches.

   Carry the ranked list into Step 2. Say what you found in one line, e.g.
   _"Found an existing app 'Acme' (com.acme.app) in workspace 'Acme Inc' — I'll
   adopt it rather than create a second one."_

6. **Score the project's stage — from repo evidence, silently. Do NOT ask.** A
   weekend prototype and a shipping app need different warnings, and asking "is
   this in production?" is a question the files already answer. Score one point
   each:
   - `ios/` or `android/` **tracked in git** (`git ls-files ios android | head -1`
     — a gitignored prebuild output does not count; a committed one is a bare
     workflow someone maintains by hand).
   - `eas.json` with a **production** profile.
   - `expo-updates` in dependencies.
   - A crash reporter, a consent/CMP, or an attribution SDK in `package.json`.
   - A `CHANGELOG*` file or a `fastlane/` directory.
   - `app.config.ts` / `app.config.js` branching on an env var, or
     `productFlavors` in `android/app/build.gradle`.

   **3 or more → shipped. 0 → POC.** 1–2 → in between: say the warnings whose
   file you actually found, skip the rest. **Announce the conclusion in one line**
   so the extra care reads as earned rather than as nagging — _"This looks like a
   shipped app (committed `ios/`, `eas.json` with a production profile,
   `fastlane/`), so I'll flag the store-review and CI things as we go."_ or
   _"Looks like a POC — I'll skip the App Store / CI warnings and keep this
   short."_

   **Shipped** turns on the store-facing and pipeline items: the iOS privacy
   manifest (Step 9), and the `expo prebuild`, OTA and EAS-env lines in Step 5.4.
   **POC** turns them all off — there is no review, no CI and no OTA channel to
   get wrong, and reciting them is pure noise.
