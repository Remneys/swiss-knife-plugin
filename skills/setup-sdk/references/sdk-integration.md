## Step 5 — Integrate the SDK

### Contents

- 1. Install
- 2. Create `src/swiss-knife.ts`
- 3. Invoke it at the app root, before render
- 4. Project-shape hazards

> **Permissions (bare MCP prompt vs plugin).** Under the Swiss-Knife **plugin**, the
> file-edit and install tools (`npm install`, `npx expo install`, `Write` / `Edit`) are
> pre-allowed by this skill's `allowed-tools`, so the steps below run without prompts.
> If you reached this wizard via the **bare MCP prompt** (`/mcp__swiss-knife__setup`, no
> plugin installed), there are **no pre-granted permissions** — Claude Code may ask you
> once to approve the `npm` / `expo install` and the file writes. Choosing **"Yes, and
> don't ask again"** lets the rest of the wizard run smoothly.

**Wire exactly what they ticked in Step 2 — no more, no less.** Every capability
they selected gets real code here; every one they didn't gets nothing. That
includes the bootstrap call itself: **Subscriptions ticked → `SwissKnife.configure`;
Subscriptions unticked → `SwissKnife.configureCore`.** There is no always-emitted
call any more. If you find yourself about to add something nobody asked for,
don't.

> **The RN bridge exposes exactly two bootstrap calls: `configure` and
> `configureCore`.** `configurePurchases(...)` is real, but only on the native
> iOS/Android SDKs — it is **not** bridged to JavaScript. So on React Native
> there is no "start with core, add purchases later at runtime" path: adding the
> store means changing the single `configureCore` call to `configure`. Don't
> emit `SwissKnife.configurePurchases(...)` — it does not exist and will not
> typecheck.

1. **Install** (Expo): `npx expo install @app-swiss-knife/react-native-sdk`.
   For bare React Native: `npm install @app-swiss-knife/react-native-sdk`.

   **If — and only if — they ticked Crash reporting**, also install the optional
   peer `@react-native-async-storage/async-storage`
   (`npx expo install @react-native-async-storage/async-storage`). Crash
   reporting resolves it lazily to persist events and screenshots captured while
   the app is offline or already crashing; without it the SDK still works, but
   that queue is memory-only and a crash during launch can be lost before it is
   ever sent. Nothing else in the SDK needs it — it is **not** an entitlement
   cache.

   The package is on the public npm registry and installs like any other
   dependency — its native sources ship inside the tarball, so there is no
   CocoaPods trunk or Maven fetch to wait on. Expect it to just work; say so in
   one line and move on.

   > If the install **does** fail, that is a real error — surface it verbatim
   > rather than narrating past it. The usual causes are environmental: a private
   > or misconfigured npm registry, no network, or a lockfile conflict. Resolve it
   > before continuing, because Steps 5.2–5.3 import from this package and a
   > typecheck will fail without it.

   **Platform minimums — iOS 15.1, Android API 24.** The `npm` install itself
   never fails on these; the mismatch surfaces later, at `pod install` (i.e.
   during `npx expo prebuild -p ios`, which runs `pod install` for you). The pod
   declares `s.platforms = { :ios => "15.1" }`, which matches the React
   Native / Expo default `ios.deploymentTarget`, so a **stock** app resolves
   fine and there is nothing to do. But if the host app has **lowered** its
   deployment target below 15.1, CocoaPods refuses to resolve and prints:

   ```
   [!] CocoaPods could not find compatible versions for pod "SwissKnifePurchases":
     In Podfile:
       SwissKnifePurchases (from `../node_modules/@app-swiss-knife/react-native-sdk`)

   Specs satisfying the `SwissKnifePurchases (from `../node_modules/@app-swiss-knife/react-native-sdk`)` dependency were found, but they required a higher minimum deployment target.
   ```

   **That message never names the required version** — do not send the user
   hunting for it. If you see "required a higher minimum deployment target"
   anywhere near `SwissKnifePurchases`, it means exactly one thing: the app's
   iOS deployment target is below **15.1**. The fix is to raise it via
   `expo-build-properties`:

   ```bash
   npx expo install expo-build-properties
   ```

   ```json
   // app.json → expo.plugins
   ["expo-build-properties", { "ios": { "deploymentTarget": "15.1" } }]
   ```

   Then re-run `npx expo prebuild -p ios --clean`. For a **bare** React Native
   app there is no `expo-build-properties` — edit `ios/Podfile` directly
   (`platform :ios, '15.1'`) and re-run `pod install`.

   > **Say this out loud before you do it: raising `deploymentTarget` raises the
   > minimum iOS version of the user's _entire app_, not just this SDK.** Devices
   > below that version can no longer install it. iOS 15.1 is a low bar in
   > practice, but it is the user's call to make, not yours — tell them what it
   > costs and get agreement rather than silently editing `app.json`.

   Android has no equivalent trap: the module's `minSdkVersion` **defers** to the
   host app's `rootProject.ext.minSdkVersion` and only falls back to 24, which is
   already the React Native / Expo default. It cannot force a host app upward.

2. **Create `src/swiss-knife.ts`.** The template below is the API as it actually
   exists (it mirrors the SDK README's quickstart, with every capability wired).

   **How to use the template:** it is written with **every** capability wired.
   Copy it, then **delete the blocks whose capability the user did not tick**.
   Each optional block is fenced by `// ── <Capability> ──` /
   `// ── end <Capability> ──` and has a matching import on the first line to
   trim. Delete the fence comments themselves too — they are scaffolding for
   you, not for the user's codebase. Everything outside a fence (`API_KEY`,
   `SERVER_URL`, the bootstrap call) stays in **every** case.

   The **Subscriptions** fence is different from the others: it is not
   add-or-omit, it is a **swap**. Ticked, you keep `SwissKnife.configure(...)`
   and the `getOfferings()` smoke test. Unticked, you delete both and keep the
   `SwissKnife.configureCore(...)` line instead — exactly one of the two
   bootstrap calls survives into the file.

   > **Pass `serverURL` every single time — this is the trap that ruins a
   > self-hosted setup.** `CrashReporting` and `Performance` build their upload
   > URL as `const baseURL = _config.serverURL || "https://api.swiss-knife.dev"`
   > (`crash-reporting.ts` `postSessionPayload`, `performance.ts`
   > `flushTransactions`). Omit `serverURL` on a self-hosted deployment and the
   > app cheerfully POSTs its crash reports and traces to Swiss-Knife's **hosted**
   > host instead of the user's own — no error, no warning, no data in their
   > dashboard, and it looks like the SDK is simply broken. `Analytics.init` has
   > no such hardcoded fallback, but it only wires the native observability
   > backend when `serverURL` is truthy (Android), so it needs the value too.
   > That is why the template resolves the origin **once** into `SERVER_URL`,
   > with a local-dev fallback so it can never be `undefined`, and hands the same
   > value to every entry point.

   ```ts
   // src/swiss-knife.ts
   // Configure Swiss-Knife once, at app start, before any other SDK call.
   // User identity lives at your app's AUTH boundary, not here — Step 6 appends
   // the Identity.logIn / Identity.logOut helpers to this same file.
   import {
     SwissKnife,
     Purchases, // ── drop if Subscriptions was not selected
     Analytics, // ── drop if Analytics was not selected
     CrashReporting, // ── drop if Crash reporting was not selected
     Performance, // ── drop if Performance tracing was not selected
     Attribution, // ── drop if Install attribution was not selected
   } from "@app-swiss-knife/react-native-sdk";

   const API_KEY = process.env.EXPO_PUBLIC_SWISSKNIFE_API_KEY as string;
   // The deployment ORIGIN (no /api/mcp, no trailing slash). Resolved once and
   // handed to every entry point below — see the note above on why omitting it
   // silently ships your data to the wrong host.
   const SERVER_URL =
     process.env.EXPO_PUBLIC_SWISSKNIFE_API_URL ?? "http://localhost:3000";

   export function initSwissKnife() {
     // ── Subscriptions ──────────────────────────────────────────────────────
     // Core + the purchase engine: a StoreKit.Transaction.updates observer on
     // iOS, a BillingClient on Android, and the first-launch install
     // registration. Only correct if Swiss-Knife OWNS purchases in this app.
     SwissKnife.configure({
       apiKey: API_KEY,
       // proxyURL is the deployment ORIGIN — points the native SDK at your
       // Swiss-Knife backend instead of its built-in default host.
       proxyURL: SERVER_URL,
     });
     // ── end Subscriptions ──────────────────────────────────────────────────

     // ── No subscriptions (use INSTEAD of the block above) ──────────────────
     // Core only: ingest, Identity, Attribution, subscriber attributes. Starts
     // no StoreKit observer and no BillingClient, so it is safe to run next to
     // an existing IAP library. Registers no install as a side effect — call
     // Attribution.registerInstall() yourself if you want one.
     SwissKnife.configureCore({ apiKey: API_KEY, proxyURL: SERVER_URL });
     // ── end No subscriptions ───────────────────────────────────────────────

     // ── Analytics ──────────────────────────────────────────────────────────
     void Analytics.init({ apiKey: API_KEY, serverURL: SERVER_URL });
     // ── end Analytics ──────────────────────────────────────────────────────

     // ── Crash reporting ────────────────────────────────────────────────────
     CrashReporting.init({ apiKey: API_KEY, serverURL: SERVER_URL });
     // ── end Crash reporting ────────────────────────────────────────────────

     // ── Performance tracing ────────────────────────────────────────────────
     Performance.init({ apiKey: API_KEY, serverURL: SERVER_URL });
     // ── end Performance tracing ────────────────────────────────────────────

     // ── Subscriptions ──────────────────────────────────────────────────────
     // Smoke test: proves the key + origin are wired correctly. Requires the
     // purchase engine — it throws UninitializedPurchasesError after
     // configureCore, so it belongs to this fence.
     Purchases.getOfferings()
       .then((offerings) =>
         console.log(
           "[swiss-knife] configured. current offering:",
           offerings.current?.identifier ??
             "(none — create one with the offering_create tool)",
           "| packages:",
           offerings.current?.availablePackages.length ?? 0
         )
       )
       .catch((err) => console.warn("[swiss-knife] getOfferings failed:", err));
     // ── end Subscriptions ──────────────────────────────────────────────────

     // ── No subscriptions (use INSTEAD of the block above) ──────────────────
     // Core-only smoke test. isConfigured() reports the PURCHASE engine and is
     // correctly false here; isCoreConfigured() is the one that matters.
     SwissKnife.isCoreConfigured()
       .then((ok) => console.log("[swiss-knife] core configured:", ok))
       .catch((err) => console.warn("[swiss-knife] core check failed:", err));
     // ── end No subscriptions ───────────────────────────────────────────────
   }

   // ── Install attribution ──────────────────────────────────────────────────
   /**
    * Shows the iOS App Tracking Transparency prompt, then registers the install.
    * Call this from a DELIBERATE moment in your UX (e.g. after onboarding) —
    * never at cold start; Apple expects the prompt to be contextual, and
    * Info.plist must carry NSUserTrackingUsageDescription. Idempotent: a second
    * call returns the cached resolution without re-prompting or re-POSTing.
    * On Android there is no prompt; it collects the advertising id + install
    * referrer instead.
    */
   export async function promptForAttribution() {
     try {
       return await Attribution.requestAndRegister();
     } catch (err) {
       console.warn("[swiss-knife] attribution failed:", err);
       return null;
     }
   }
   // ── end Install attribution ──────────────────────────────────────────────
   ```

   Two options worth mentioning to the user but **not** worth setting for them:
   `CrashReporting.init` and `Performance.init` both accept `environment`
   (defaults to `"production"`) and `release`, which are what separate dev noise
   from real production incidents once they wire them to their build config;
   and `Performance.init({ enableAutoHTTP: true })` times every `fetch` by
   monkey-patching the global — genuinely useful, but it is off by default and
   turning it on is the app owner's call, not yours.

   **If you wired the attribution helper, say these things out loud** — they are
   the difference between working attribution and a silently wrong dashboard.
   `promptForAttribution()` is exported but **never called** by the wizard, so
   nothing prompts until they add a call site. Then, depending on which bootstrap
   you emitted:
   - **`configure` (Subscriptions ticked)** — `sendInstallOnFirstLaunch` defaults
     to `true`, so the **organic** install is already registered without any
     prompt. If they also run AppsFlyer / Adjust / Branch / Singular, the same
     install is now being counted by two systems.
   - **`configureCore` (Subscriptions unticked)** — **no install is registered at
     all.** Core has no first-launch side effects by design. If they want the
     organic install without the ATT prompt, they call
     `Attribution.registerInstall()` (no argument — no IDFA, no prompt); the
     `promptForAttribution()` helper above is the IDFA path and does prompt.
     Both are idempotent natively, so a second call returns the cached
     resolution rather than double-registering.

3. **Invoke it at the app root, before render.** `SwissKnife.configure` is a static
   class (no `<Provider>`), so call `initSwissKnife()` once at the top of the root
   component. In an Expo app whose entry renders `App.tsx`, add a root `useEffect`:

   ```tsx
   // App.tsx (or app/_layout.tsx for expo-router — put it in the ROOT component)
   import { useEffect } from "react";
   import { initSwissKnife } from "./src/swiss-knife";

   export default function App() {
     useEffect(() => {
       initSwissKnife();
     }, []);
     // …existing render…
   }
   ```

   For **expo-router**, add the same `useEffect` to the root `app/_layout.tsx`.
   Read the actual entry file first and Edit it in place — match the project's
   existing imports and structure; don't overwrite it.

   **Only if they ticked Crash reporting**, also wrap the rendered tree in
   `<SKErrorBoundary>`. This is not decoration: it is a React error boundary, and
   the global JS handler that `CrashReporting.init` installs **cannot see errors
   thrown during render or in lifecycle methods** — React swallows those. Without
   the boundary you lose exactly the class of crash that white-screens the app.
   `componentDidCatch` forwards them to `CrashReporting.captureException` tagged
   `react.error_boundary`. Wrap as high as possible, inside the component that
   calls `initSwissKnife()`:

   ```tsx
   import { SKErrorBoundary } from "@app-swiss-knife/react-native-sdk";

   return (
     <SKErrorBoundary fallback={<YourExistingErrorScreen />}>
       {/* …existing tree… */}
     </SKErrorBoundary>
   );
   ```

   `fallback` is optional, but **omitting it renders `null`** after a crash —
   i.e. a blank screen. Use the project's existing error/empty state if it has
   one; if it doesn't, a two-line "Something went wrong" view is enough. Don't
   invent an elaborate one.

   **Only if they ticked Performance tracing** and the app mounts React
   Navigation's `<NavigationContainer>` itself, wire the screen tracker so each
   route becomes a transaction:

   ```tsx
   import { Performance } from "@app-swiss-knife/react-native-sdk";

   // Module scope (or a useRef) — create it ONCE. The tracker holds the current
   // route internally, so a per-render instance never closes a transaction.
   const onNavigationStateChange = Performance.createNavigationTracker();

   <NavigationContainer onStateChange={onNavigationStateChange}>;
   ```

   **expo-router owns its own `NavigationContainer`**, so there is no
   `onStateChange` to hand this to. Don't fake it — tell the user plainly that
   automatic screen transactions aren't wired on expo-router, and that
   `Performance.startTransaction(name, op)` / `transaction.finish()` still work
   for the spans they care about. Same answer for any app with no navigator.

4. **Project-shape hazards — one line each, and only for the file you actually
   found.** Do not recite this list. Check the left column against the project,
   say the ones that hit, skip the rest in silence. The rows marked **[shipped]**
   apply only when Step 1.6 scored the project as shipped — on a POC they are
   noise.

   | You found                                                                                                                                          | Say (or do)                                                                                                                                                                                                                                                                       |
   | -------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | the host's `AndroidManifest.xml` sets `android:dataExtractionRules` on `<application>`                                                             | **A build failure, not a warning.** The SDK's manifest sets `android:dataExtractionRules="@xml/swissknife_data_extraction_rules"`; the merger refuses to choose between them. Fix it in their manifest — exact edit below.                                                        |
   | **[shipped]** a committed, hand-edited `ios/` or `android/` (bare workflow)                                                                        | **Do not run `npx expo prebuild`.** It regenerates those directories and can discard native edits — a `Podfile` tweak, an entitlement, an `AppDelegate` change. Run `cd ios && pod install` instead, and leave prebuild to the user.                                              |
   | **[shipped]** `expo-updates` in dependencies                                                                                                       | `eas update` **cannot deliver this**. A new native module needs a new binary; publishing an OTA after this wizard changes nothing on any device. Their testers need a fresh build.                                                                                                |
   | **[shipped]** `eas.json`                                                                                                                           | The `.env` you just wrote is **local only**. Put `EXPO_PUBLIC_SWISSKNIFE_API_KEY` in each profile's `env` block (or `eas env:create` / EAS secrets), or every CI build ships an `undefined` key and fails at runtime with no build error.                                         |
   | `react-native-config`, `react-native-dotenv`, or an `app.config.ts` that reads `process.env` into `expo.extra`                                     | Write the key **into their convention** (`Config.SWISSKNIFE_API_KEY`, `@env`, `Constants.expoConfig.extra`) and read it that way in `src/swiss-knife.ts` — do not add a lone `EXPO_PUBLIC_*` beside an existing env system. Two conventions is how a key goes missing in release. |
   | more than one bundle id — `productFlavors`, an `app.config.ts` that branches on env, `.env.staging`                                                | Staging and production should be **separate Swiss-Knife apps**, not one app with mixed data. Spotted in Step 1 → fold "which one?" into Step 2's bundle-id question (no extra call). Spotted now → say it and let them re-run the wizard for the second one.                      |
   | a consent platform (`@usercentrics/*`, `didomi`, `@sourcepoint/*`, UMP via `react-native-google-mobile-ads`, `react-native-tracking-transparency`) | `Analytics.init` emits `[SwissKnife] Session Start` immediately — **before** any consent gate runs. If that matters in their jurisdiction, init with `optOut: true` and call `Analytics.setOptOut(false)` once consent is granted.                                                |

   The manifest-merger fix, in the host's
   `android/app/src/main/AndroidManifest.xml` (add the `tools` namespace if it
   isn't there):

   ```xml
   <manifest xmlns:android="http://schemas.android.com/apk/res/android"
             xmlns:tools="http://schemas.android.com/tools">
     <application
         android:dataExtractionRules="@xml/their_existing_rules"
         tools:replace="android:dataExtractionRules" >
   ```

   `tools:replace` keeps **their** rules and drops ours — the SDK's file only
   excludes its own device-id entry from cloud backup, so the loss is that a
   restored backup can carry the old anonymous device id. Say that in a clause;
   don't turn it into a paragraph.
