## Step 7 — Wire the analytics boundary (what actually calls `Analytics.track`?)

### Contents

- Branch M — mirror at the wrapper (one import, one line)
- Branch V — screen views only (the zero-coupling middle path)
- Branch T / no analytics yet — say the quiet part out loud
- Branch S — scattered calls, no wrapper

**Step 5 built the pipe. Nothing is in it yet.** This is the step that fixes the
most common post-wizard reaction — _"I installed everything, it says `core
configured: true`, and the dashboard shows one event."_

> `Analytics.init(...)` emits exactly **one** event, `[SwissKnife] Session Start`,
> because `trackingSessionEvents` defaults to `true`. The only other event the SDK
> can produce on its own is `[SwissKnife] Screen Viewed`, and that one only exists
> when the app calls `Analytics.trackScreen(...)`. There is **no** automatic event
> collection — no autocapture, no tap sniffing, no network interception. So an app
> that never calls `Analytics.track(...)` shows exactly one event per launch,
> forever, and it looks indistinguishable from a broken integration. It isn't
> broken. It's empty.

**Skip this step** — in one honest line, not silently — when Analytics wasn't
ticked in Step 2 (_"Analytics isn't wired, so there's nothing to send"_), or when
the user is on a **native iOS / Android** stack (the `Analytics` module is a
React-Native surface; the native recipes cover `configure`, offerings and identity
only). Otherwise this step runs for every stack Step 5 touched.

**1. Restate what Step 1.4 found, in one line, with numbers.** If you skipped
those greps, run them now — do not ask the questions below blind. There are
exactly three shapes:

- **A central wrapper** — one module every event goes through, e.g. _"88 `track()`
  calls across 35 files, all through the `useTrack` hook, which already fans out
  to Firebase via `routeTrackEventToFirebase`."_ This is the seam: adding
  Swiss-Knife is **one line inside a pattern that already exists**.
- **Scattered calls** — the incumbent SDK is imported directly in many files.
  There is no seam, and there is no one-line fix. Say so.
- **No analytics at all** — nothing to mirror. This is a **first-class answer**,
  not a failure, and it is handled below.

**2. Ask (`AskUserQuestion`) — up to 3 questions in ONE call.**

**Question 1 — the branch selector. Its options depend on what you found.**

_If there is a wrapper (or scattered calls):_ **"Mirror your existing events into
Swiss-Knife?"** — and put the finding in the question text, e.g. _"Everything goes
through `useTrack` (88 calls, 35 files), which already fans out to Firebase.
Adding Swiss-Knife is one more line in that same pattern. It also **doubles your
event volume** — see the cost note below."_

| Option                                                                                   | Branch |
| ---------------------------------------------------------------------------------------- | ------ |
| **"Yes — add one line to `<wrapper>` so every existing event also goes to Swiss-Knife"** | **M**  |
| **"Screen views only — wire `Analytics.trackScreen`, touch nothing else"**               | **V**  |
| **"No — I'll add `Analytics.track(...)` calls where I want them"**                       | **T**  |
| **"Skip — I'll wire this later"**                                                        | —      |

On the **scattered** variant, relabel the first option honestly — _"Yes, but
there's no single wrapper; tell me which calls matter"_ — and let "Other" carry
their list. Never present a one-line fix that doesn't exist.

_If the app has no analytics at all:_ **"This app doesn't send any events yet.
Where should the first ones come from?"**

| Option                                                                                      | Branch |
| ------------------------------------------------------------------------------------------- | ------ |
| **"Screen views — wire `Analytics.trackScreen` so real events show up on the next launch"** | **V**  |
| **"Show me the pattern — I'll add `Analytics.track(...)` calls myself"**                    | **T**  |
| **"Both"**                                                                                  | V + T  |
| **"Skip — I'll wire this later"**                                                           | —      |

**Question 2 — dev builds. Ask this whenever the wrapper has a dev guard. It is
the trap this step exists for.** Put the actual code in the question text so the
consequence is visible, not inferred:

> **"Your `track()` wrapper starts with `if (isDev) return;` — tracking is
> disabled entirely in development, so nothing will be sent from a dev build. A
> dev build is exactly what you'll run right after this wizard. What should
> Swiss-Knife do there?"**

Offer exactly these four:

| Option                                                                                             | What it does                                                                                                                                                                                                        |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **"Send from dev builds, tagged `environment`"** _(recommended if they want to see it work today)_ | The Swiss-Knife line goes **above** your guard, and every event carries `environment: "development"` so you can filter dev noise out in the dashboard. Dev events count toward your volume.                         |
| **"Match the incumbent — dev sends nothing"**                                                      | The line goes **below** the guard and inherits it. Then your dev build shows **only `[SwissKnife] Session Start`** — `Analytics.init` emits that regardless of any wrapper. Verify on a release / TestFlight build. |
| **"Behind its own flag"**                                                                          | As above, but gated on `EXPO_PUBLIC_SWISSKNIFE_DEV_EVENTS === "1"`, off by default — one developer can opt in without changing anything for the team.                                                               |
| **"Silence Swiss-Knife entirely in dev"**                                                          | `Analytics.init({ …, optOut: __DEV__ })`. Nothing at all from a dev build, not even Session Start.                                                                                                                  |

**Whatever they pick, their existing guard comes out of this step
byte-identical.** You are inserting a line above it or below it — never moving,
weakening, deleting, or "temporarily commenting out" the guard that governs their
incumbent analytics. Changing what Firebase/Mixpanel do in dev is their decision
and a separate change; if you think it's warranted, say so and let them make it.

**If there is no dev guard** (no incumbent, or the wrapper sends in dev too),
don't spend the question — state the consequence in one line instead:
_"Swiss-Knife will send from dev builds, so development and production traffic
land in the same app project; say the word and I'll tag every event with
`environment`."_

**Question 3 — "Also emit a `[SwissKnife] Screen Viewed` event per screen?"** Ask
only when branch V isn't already the answer and the project actually has a
navigator. Sell it for what it is: **visible events on the next launch with zero
coupling to your existing analytics** — useful precisely when someone wants to
confirm the pipe works before committing to mirroring. Options: **"Yes"** /
**"No"**.

**3. Wire the branch they picked.**

### Branch M — mirror at the wrapper (one import, one line)

```ts
// e.g. src/hooks/useTrack.ts — read it first and match its style.
import { Analytics } from "@app-swiss-knife/react-native-sdk";

const track = useCallback(
  (event: string, properties?: Record<string, unknown>) => {
    Analytics.track(event, properties); // ← the only line added
    if (isDev) return Promise.resolve(); // ← UNCHANGED: still governs Firebase + Mixpanel
    routeTrackEventToFirebase(event, properties);
    return enqueueMixpanelOperation(/* …unchanged… */);
  },
  []
);
```

That placement is the **"send from dev builds"** answer. For **"match the
incumbent"**, the same line moves below the guard, next to
`routeTrackEventToFirebase` — one line, either way.

Why one line is genuinely enough, and safe to drop into a hot path:

- `Analytics.track()` returns `void` and is fire-and-forget: it builds and ships
  the event on a detached async task. It cannot change the wrapper's return value,
  its ordering, or its error behaviour, and it never rejects into the call site.
  **Do not add `await`** — it would buy nothing and change the wrapper's timing.
- Event names and property bags pass through **verbatim**. Swiss-Knife has no
  schema to register and no naming rules to satisfy, so their existing taxonomy
  arrives unchanged and the two dashboards stay comparable.
- This is the entire change. **Do not touch the incumbent's calls, config, or
  guards** beyond adding this line — you are adding a fan-out, not migrating them.

**Tagging the environment** (the "send from dev" and "own flag" answers). Register
it as a super-property, once, in `initSwissKnife()` **before** `Analytics.init` so
even `[SwissKnife] Session Start` carries it:

```ts
// src/swiss-knife.ts — inside initSwissKnife(), immediately before Analytics.init
Analytics.registerSuperProperties({
  environment: __DEV__ ? "development" : "production",
});
void Analytics.init({ apiKey: API_KEY, serverURL: SERVER_URL });
```

Super properties are merged into every event's properties (an explicit property of
the same name still wins), so this is one line for permanent, filterable
separation. It is **not** retroactive — only events sent after it runs carry it.

**Own-flag variant**, when they picked that: guard only the Swiss-Knife line, and
add the var to `.env.example` (unset by default) so nobody has to guess it exists.

```ts
const SEND_IN_DEV = process.env.EXPO_PUBLIC_SWISSKNIFE_DEV_EVENTS === "1";
// …in the wrapper, guarding ONLY the Swiss-Knife call:
if (!__DEV__ || SEND_IN_DEV) Analytics.track(event, properties);
```

**Full-silence variant**: `Analytics.init({ apiKey, serverURL, optOut: __DEV__ })`.
`optOut` is checked inside `track()` itself, so it suppresses Session Start too —
this is the only answer that produces a genuinely silent dev build.
`Analytics.setOptOut(false)` flips it at runtime when they do want to look.

> **Name the cost — before they answer, not after.** Mirroring **doubles event
> volume**: every event the app already sends now goes to two backends. That is
> more device network and battery, more ingest, and it consumes Swiss-Knife plan
> limits at exactly the rate their incumbent consumes theirs. Doubling is the
> _point_ — two dashboards that agree is how you trust a migration — but it has to
> be a decision, not a side effect. If they'd rather start narrow, the good middle
> is screen views plus the three or four events they'd actually act on
> (activation, paywall shown, purchase), then widen once the data earns trust.

### Branch V — screen views only (the zero-coupling middle path)

Real events on the next launch, with no dependency on the incumbent and no change
to any existing call site. `Analytics.trackScreen(name)` emits
`[SwissKnife] Screen Viewed` with a `[SwissKnife] Screen Name` property.

**expo-router** — and note this is the one place expo-router is _easier_ than
React Navigation: Step 5.3's `Performance.createNavigationTracker()` has no
`onStateChange` to hang off there, but screen analytics do, via `usePathname()`:

```tsx
// app/_layout.tsx — the same root component that calls initSwissKnife()
import { useEffect } from "react";
import { usePathname } from "expo-router";
import { Analytics } from "@app-swiss-knife/react-native-sdk";

export default function RootLayout() {
  const pathname = usePathname();
  useEffect(() => {
    Analytics.trackScreen(pathname);
  }, [pathname]);
  // …existing layout…
}
```

`usePathname()` returns the resolved path, so `/post/42` and `/post/43` are two
distinct screen names. If that cardinality is unwelcome, use
`useSegments().join("/")` instead — it yields the route _pattern_ (`post/[id]`).

**React Navigation** (the app mounts `<NavigationContainer>` itself):

```tsx
import {
  NavigationContainer,
  useNavigationContainerRef,
} from "@react-navigation/native";
import { Analytics } from "@app-swiss-knife/react-native-sdk";

// …inside the component that renders the container:
const navigationRef = useNavigationContainerRef();
const routeNameRef = useRef<string | undefined>(undefined);

return (
  <NavigationContainer
    ref={navigationRef}
    onReady={() => {
      routeNameRef.current = navigationRef.getCurrentRoute()?.name;
    }}
    onStateChange={() => {
      const previous = routeNameRef.current;
      const current = navigationRef.getCurrentRoute()?.name;
      if (current && current !== previous) Analytics.trackScreen(current);
      routeNameRef.current = current;
    }}
  >
    {/* …existing tree… */}
  </NavigationContainer>
);
```

If Step 5.3 already wired `Performance.createNavigationTracker()` to
`onStateChange`, there is only one `onStateChange` — call both from it rather than
replacing one with the other:

```tsx
onStateChange={(state) => { onNavigationStateChange(state); /* …trackScreen… */ }}
```

No navigator at all? Say so plainly and fall back to branch T — a
`Analytics.trackScreen("Home")` on mount of the two or three real screens is fine.

### Branch T / no analytics yet — say the quiet part out loud

**`Analytics.track(...)` is a call they make. The SDK will not invent events.**
State it in one sentence, then show a real example — not `foo` / `bar`:

```ts
import { Analytics } from "@app-swiss-knife/react-native-sdk";

// after onboarding finishes
Analytics.track("Onboarding Completed", { steps: 4, skipped: false });

// when a paywall is presented, and when it converts
Analytics.track("Paywall Viewed", {
  placement: "settings",
  offering: "default",
});

// a screen view (emits "[SwissKnife] Screen Viewed")
Analytics.trackScreen("Settings");
```

Free-form name, free-form property bag, fire-and-forget, nothing to register
first. Three or four events they would actually act on beats fifty they won't.
Then point at **files you actually saw in this project** — the onboarding screen,
the paywall component, the purchase handler — as the natural first call sites. A
concrete suggestion, not a lecture. **Do not add these calls unless they picked
branch T's "show me the pattern" and asked you to place them.**

### Branch S — scattered calls, no wrapper

Do **not** rewrite N call sites, and do **not** introduce a project-wide wrapper on
their behalf — that is a refactor they own. Offer the two honest options: name the
three-to-five events that matter and add `Analytics.track(...)` beside those calls
(surgical, reviewable, easy to revert), or wire branch V and add events later. If
they ask what a wrapper would look like, show five lines and let them decide.

**4. Say what arrives, and when.** Three lines, because each one prevents a
false alarm:

- **Events are batched, not instant.** They queue natively and flush on the native
  schedule (`flushIntervalMs` defaults to 30 s, `flushQueueSize` to 30), so a first
  event can take up to ~30 seconds to appear. `Analytics.flush()` forces it. Don't
  let them declare the wiring broken after ten seconds.
- **Events carry identity automatically.** Every event is stamped with the device
  id, the session id, and — once Step 6's `Identity.logIn` has run — the subject's
  `user_id`, so the dashboard joins events to a person. Analytics and identity stay
  separate calls; they resolve to the same subject.
- **Nothing is backfilled.** Events that happened before the mirror line existed
  are not imported from the incumbent. The two dashboards only agree going forward.
