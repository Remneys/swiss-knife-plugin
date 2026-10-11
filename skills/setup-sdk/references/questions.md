## Step 2 — Ask the user (use the `AskUserQuestion` tool)

### Contents

- Fast path for a brand-new user
- Call 1: identity (workspace, app, name, platform, bundle id)
- Call 2: what to turn on (Subscriptions, crash reporting, attribution, ...)
- Store-conflict and crash-reporter coexistence warnings
- Question budget

**Never silently use a detected or invented workspace / app / bundle name, and
never silently create one that may already exist.** Every identity field below MUST
appear in the questions for the user to confirm or change; if a value can't be
detected from the project, ASK for it — don't fabricate one. What you detected and
discovered in Step 1 are **defaults you surface as the recommended option**, never
choices you make on the user's behalf.

**The brand-new-user fast path (do not add friction here).** If `workspace_list`
returned **zero** workspaces, there is nothing to adopt: do **not** ask a workspace
question and do **not** ask an app question. Go straight to asking only what you
genuinely need to create — workspace name, app name, platform, bundle id — exactly
as before. Adoption questions exist only when there is something to adopt.

`AskUserQuestion` allows at most **4 questions per call**, so split identity from
setup into two calls. Pre-fill each question with your detected value as the
recommended option, but ALWAYS let the user pick a different value via **"Other"**.

**Call 1 — identity (up to 4 questions):**

- **Workspace** — offer each workspace from `workspace_list` by `name` (include
  your role, and mark the one containing the best-matching app as the recommended
  option), plus **"Create new workspace"** as the LAST option. If the list is
  empty, skip this question entirely and create one (above). Whenever the user
  picks "Create new", you MUST obtain a workspace **name** — offer a sensible
  default option (e.g. derived from the app / project name) and let them change it
  via "Other". Never hard-code a name like "My Workspace".
- **App** — ask this **only if the chosen workspace already contains apps.** Offer
  each existing app as `<name> — <bundle id or platform>`, ordered by the Step 1
  ranking with the **best match FIRST and pre-selected as the recommended
  option**, plus **"Create a new app"** as the LAST option. Label the reason
  inline so the choice is obvious, e.g. _"Acme (com.acme.app) — matches this
  project's bundle id"_ or _"Acme (setup unfinished)"_.
  - If a **key-prefix match** was found, say so plainly ("this project is already
    connected to this app") and make it the only sensible pick.
  - If the workspace has **no** apps, skip this question — creating is the only
    option.
- **App name**, **Platform**, **Bundle id** — ask these **only on the create
  path.** Adopting an existing app takes its name / platform / bundle id as-is;
  re-asking is noise, and the wizard does not modify an adopted app's identity.
  On the create path: default the app name to the detected `expo.name` /
  `app.json` name; platform is `ios`, `android`, or `both`; default the bundle id
  to the one detected in Step 1, confirmable via "Other". For **`both`**, this
  question covers the iOS id; collect the Android id in Call 2 so you never exceed
  4 questions in one call.

Because the adopt path drops three questions, Call 1 is usually a **single**
question (or none at all when there is exactly one workspace with one exactly
matching app — in that case just state what you're adopting and move on).

**Call 2 — what to turn on (1 question), plus the Android bundle id if platform
is `both`:** one multi-select, _"What should I wire up?"_. Offer exactly these
six options, with the first three and the last **pre-selected** — except that if
Step 1.4 found a competing IAP library, leave **Subscriptions unchecked**, and if
it found an incumbent crash reporter, leave **Crash reporting unchecked**. Both
exceptions carry their reason **in the question text**, so the user unticks
nothing and reads why the box is already the way it is (the two notes below).

| Option                      | Default      | What ticking it actually produces                                                                                                                                                                                                                                                                                                                       |
| --------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Subscriptions**           | ✅ checked   | `SwissKnife.configure(...)` (core **+** the store) + catalogue + the `getOfferings()` smoke test. Untick it and Step 5 emits `configureCore(...)` instead — no store.                                                                                                                                                                                   |
| **Analytics**               | ✅ checked   | `Analytics.init(...)` in `initSwissKnife()` — the **pipe**, not the events. Step 7 wires what actually calls `Analytics.track(...)`; without it the dashboard shows only `[SwissKnife] Session Start`.                                                                                                                                                  |
| **Crash reporting**         | ✅ checked   | `CrashReporting.init(...)` + an `<SKErrorBoundary>` at the root + an alert rule.                                                                                                                                                                                                                                                                        |
| **Performance tracing**     | ⬜ unchecked | `Performance.init(...)` — the **pipe**, not the spans. Pure JS, no native module. Screen transactions only if the app mounts a `NavigationContainer` (step 3 below): **expo-router has none**, so there the section stays empty until they call `Performance.startTransaction(...)` themselves or turn on `enableAutoHTTP`. Say which case they are in. |
| **Install attribution**     | ⬜ unchecked | An `Attribution.requestAndRegister()` helper — **shows the iOS ATT prompt.**                                                                                                                                                                                                                                                                            |
| **Seed a starter offering** | ✅ checked   | A monthly + annual "pro" so `getOfferings()` returns something real.                                                                                                                                                                                                                                                                                    |

**This answer is a contract, not a formality.** Step 5 emits code for every box
they ticked and nothing for the boxes they didn't. Carry the selection forward
verbatim — do not quietly widen it ("analytics is useful, I'll add it anyway") or
narrow it.

**Subscriptions is a real switch — it decides which bootstrap call Step 5
emits.** The SDK's bootstrap is split in two, and the two halves are separately
callable:

- **Ticked** → `SwissKnife.configure(...)`, which is exactly **core + the purchase
  engine**: a process-wide `StoreKit.Transaction.updates` observer on iOS, a
  `BillingClient` on Android, and the first-launch install registration
  (`sendInstallOnFirstLaunch`, default `true`).
- **Unticked** → `SwissKnife.configureCore(...)` instead. Core is event ingest,
  `Identity`, `Attribution` and subscriber attributes — everything analytics and
  identity need, with **no store ownership at all**: no observer, no
  `BillingClient`, and no install registered as a side effect.

An analytics/identity-only integration is a **fully supported shape**, and it
must produce `configureCore`. Emitting `configure` for someone who unticked
Subscriptions is a bug, not a safety net — you would be handing store ownership
to an app that never asked for it.

> Historical note, in case you have seen the old advice: `configure` used to be
> described as mandatory "because on iOS it is what wires the native backend."
> That was describing a bug as a feature — `Analytics.init(...)` was a silent
> no-op on **both** platforms because of a truthiness guard on `serverURL`. That
> guard is gone. `Analytics.init({ apiKey, serverURL })` now bootstraps core
> natively on iOS and Android alike, so an analytics-only app genuinely needs no
> `configure`. Do not resurrect the old claim.

> **RevenueCat compatibility, for free.** `Purchases.configure` / `Purchases.setAttributes`
> and friends still work — same function objects as `SwissKnife.*` / `Attributes.*`, not
> deprecated wrappers — so an app migrating from `react-native-purchases` doesn't have to
> rename a single call site. Requires SDK ≥ 0.3.0.

**⚠️ If the app already ships another IAP library, ticking Subscriptions is a
live-revenue hazard.** Before you emit `configure`, grep `package.json` for
`react-native-purchases` (RevenueCat), `react-native-iap`,
`expo-in-app-purchases`, `adapty`, `qonversion`, `glassfy` — and on native, for
an existing `Transaction.updates` loop or `BillingClient`. If any is present,
say this out loud and let the user decide before you wire anything:

> Turning on Swiss-Knife purchases starts a **second**
> `StoreKit.Transaction.updates` observer, and its handler calls
> `transaction.finish()`. Your existing IAP SDK is running its own observer doing
> exactly the same thing. Both wake for the same live renewal and race to finish
> it — whichever wins can retire the transaction before the other has recorded
> it, so a real renewal from a paying customer silently goes unrecorded by one of
> the two systems. Android is the same story with two `BillingClient`s and two
> `PurchasesUpdatedListener`s.

This does **not** reproduce in a simulator or on a fresh sandbox purchase. It
surfaces weeks later as renewals that one dashboard has and the other doesn't,
which is the worst possible way to find out. Two SDKs can safely share analytics
and identity; they cannot safely share the store transaction queue.

So the honest options are: **keep the incumbent** → untick Subscriptions and use
`configureCore`, which exists precisely for this and is safe to run alongside
anything; or **migrate to Swiss-Knife** → tick it, but remove the other SDK's
`configure`/observer in the same change. "Run both for a while and compare" is
not a third option — that is the racing configuration.

**If the app already ships a crash reporter, ticking Crash reporting gives them
two reporters — not a broken one.** State it calmly; this is a duplication
trade-off, not a hazard. On **iOS** (and only iOS — Android and the JS layer have
always chained) both SDKs install `NSSetUncaughtExceptionHandler` and signal
handlers, so the question is what happens to the incumbent's. Ours **captures
whatever was installed before it and chains to it**: the incumbent's handler still
runs on every crash, and signal handlers it can't call (`SIG_DFL` / `SIG_IGN`) are
filtered rather than jumped to. Nothing of theirs stops working.

So the real cost is duplication: the same crash lands in two dashboards, fires two
alert rules and consumes two quotas. Exactly what you want while migrating off
Sentry / Crashlytics; noise otherwise — which is why the box starts unticked when
one is present. Say the trade-off in one line and let them tick it. If they want
the incumbent to keep owning **native** crashes while Swiss-Knife handles the JS
side, that is a supported shape:
`CrashReporting.init({ …, enableNativeCrashHandler: false })` — and its siblings
`enableGlobalErrorHandler`, `enablePromiseRejectionTracking`,
`enableAppStateTracking`, `enableAutoBreadcrumbs` are all `true` by default and opt
out the same way.

> **Don't try to verify coexistence on a dev build.** `enabled: !__DEV__` is a
> near-universal line in these SDKs' setup snippets, so on a dev build the
> incumbent installs **no native handler at all** — there is nothing to chain to
> and a clean-looking test proves nothing. Check on a release / TestFlight build,
> or don't claim to have checked.

**One more thing this question does NOT control: adopting an app that already
has offerings.** Drop "Seed a starter offering" from the pre-selected set in that
case; Step 3.4 checks and refuses to duplicate a catalogue anyway.

**Why install attribution is unchecked, and must stay unchecked unless they
deliberately tick it.** `Attribution.requestAndRegister()` triggers the iOS **App
Tracking Transparency** prompt. That is not a neutral default:

- It needs `NSUserTrackingUsageDescription` in `Info.plist` and a privacy
  manifest that declares tracking. Get either wrong and App Store review rejects
  the build.
- It is a visible, one-shot, user-facing prompt. Turning it on for someone
  changes their app's first-run experience without them asking for it.
- If the app already ships **AppsFlyer / Adjust / Branch / Singular**, both SDKs
  register the same install and the same campaign gets **counted twice**. Grep
  `package.json` for those before wiring anything; if one is present, say so and
  let the user decide which system owns attribution.

State the flip side too, so nobody thinks this is all-or-nothing — but state it
**conditionally**, because the store split moved this:

- **If they ticked Subscriptions** (so you emit `configure`),
  `sendInstallOnFirstLaunch` defaults to **`true`** and registers an **organic**
  install on first launch — no IDFA, no prompt, no ATT. Leaving the attribution
  box unchecked still gives them organic install attribution; what they give up
  is the ad-network / IDFA-attributed slice.
- **If they did not** (so you emit `configureCore`), there is **no automatic
  install at all**. Core deliberately registers nothing as a side effect — the
  `sendInstallOnFirstLaunch` option is accepted on the options object but inert
  on this path. A core-only app that wants install attribution must call
  `Attribution.registerInstall()` explicitly. Say this plainly rather than
  letting them assume installs are arriving; Step 5's template handles it.

**Question budget.** `AskUserQuestion` caps at **4 questions per call**, and no
call below is allowed to exceed that:

| Call              | Questions | What they are                                                                     |
| ----------------- | --------- | --------------------------------------------------------------------------------- |
| Step 2 — call 1   | ≤ 4       | workspace, app name, platform, iOS bundle id (the adopt path drops three)         |
| Step 2 — call 2   | ≤ 2       | this multi-select + the Android bundle id when platform is `both`                 |
| Step 6 — one call | ≤ 4       | identity shape, does the id change on sign-in, which call site, which user fields |
| Step 7 — one call | ≤ 3       | mirror existing events, dev builds, screen views                                  |

**Four** calls, never more than 4 questions in any one of them — and Step 7's call
doesn't happen at all unless Analytics was ticked here. **Step 8 spends nothing of
its own**: its single decision (which user fields to attach) is question 4 of Step
6's call, and its "where" is the call site Step 6 already chose. Do not add
questions beyond these; if you find yourself wanting a fifth call, you are asking
something the project files could have told you.
