## Step 8 — Attach the user's data (what the person card actually shows)

**Step 6 bound the device to a subject id. That id is all the dashboard has.** The
person card is one opaque string and the People list is a column of them, which
reads as "the SDK collects nothing" when the truth is "nobody has sent anything
yet." This step closes that, and it is the step whose calls are otherwise found
one at a time, by symptom.

Three calls write user data — they land in **three different places**, only two of
which touch the person — plus a fourth that needs nothing from you at all:

| Call                                          | Where it lands                                            | Person card?                                                         |
| --------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------------------------- |
| `Identity.logIn(id)` — Step 6                 | the app-user's `external_user_id`                         | **Yes** — creates and links the workspace person                     |
| `Attributes.set({ … })` / `setEmail(…)`       | the app-user's `properties`; reserved `$`-keys go further | **Yes** — links the person, and this is what actually fills the card |
| `Analytics.identify(new SKIdentify().set(…))` | user properties on the analytics event stream             | **No** — nothing you send here reaches the person card               |
| `CrashReporting.setUser({ … })`               | the subject on crash reports                              | n/a — the id is **already pulled from `Identity`** at report time    |

So: **`Attributes.*` is the only path to the person card** — `set` writes it
directly, `identify` segments events, and `setUser` is an override you almost
never need. Do not tell anyone to call
`CrashReporting.setUser` to get the user id onto their crashes — crash reports
resolve the subject from `Identity` when the report is built (SDK ≥ 0.2.5), so
Step 6 already did it. `setUser` is for an explicit `email` / `username`, or for
reporting against a subject that is deliberately not the identified one.

**Skip this step in one honest line** when Step 6's question 4 came back "just the
id" or "skip", or when Step 6 landed on branch C — no server-side subject, nothing
to describe.

**1. Wire it where Step 6 wired identity**, immediately after
`identifySwissKnifeUser(user.id)`, so the subject and its data can never drift
apart. Append to `src/swiss-knife.ts`:

```ts
// src/swiss-knife.ts (append) — widen the existing import rather than adding a
// second one: import { SwissKnife, Identity, Attributes, Analytics, SKIdentify } from "…";

/**
 * Describe the current subject. Call right after identifySwissKnifeUser().
 *
 * Every field is listed BY NAME on purpose — see the two rules below. Add a
 * field here only when the app owner asked for that field.
 */
export async function describeSwissKnifeUser(user: {
  plan?: string | null;
  signupDate?: string | null;
}) {
  try {
    // The person card + segmentation. Values are strings; null clears one.
    await Attributes.set({
      plan: user.plan ?? null,
      signup_date: user.signupDate ?? null,
    });
  } catch (err) {
    // Non-fatal, and never worth blocking the UI for.
    console.warn("[swiss-knife] Attributes.set failed:", err);
  }

  // Analytics user properties — same values, different destination. Returns
  // void and is fire-and-forget, so there is nothing to await.
  Analytics.identify(new SKIdentify().set("plan", user.plan ?? "free"));
}
```

Only if question 4 allowed PII, and only for the fields they ticked, add the
typed helpers instead of hand-rolling the reserved keys —
`await Attributes.setEmail(user.email)` and
`await Attributes.setDisplayName(user.name)` (also `setPhoneNumber`,
`setPushToken`). They validate the format server-side and are what the dashboard
renders as the person's name and email.

`Attributes.set` and its helpers need **core** configured, which `configure`,
`configureCore` and even `Analytics.init` all satisfy — so this works in an
analytics-only app with no store, and needs no extra bootstrap.

**2. Two rules. They are not style preferences, and they hold whatever the user
answered.**

> **Never send the whole user object.** No `setAttributes({ ...user })`, no
> `Object.entries(user)` fed into `SKIdentify`, no "just send everything and
> we'll filter later" — not even when asked for it directly. A real app's user
> type keeps `accessToken` and `refreshToken` **next to** `email` and `plan`; a
> spread ships a live session credential into an analytics backend, where it is
> stored, rendered verbatim on the person card, and visible to everyone in the
> workspace. Enumerate the fields by name. If you can't name it, you can't send
> it — and if the user asks for a spread, say why you won't and offer the
> explicit list instead.

> **PII is the app owner's decision, not your default.** Email, name and phone
> are legitimate and useful — and they are shown verbatim in the dashboard,
> travel onto crash reports through `setUser`, and can carry consent obligations
> you cannot see from here. Same rule as the ATT prompt: attach them only when
> the user deliberately ticked them, never widen the set later "while you're in
> there", and if they never answered, send the non-PII set and say so in one
> line.

**3. Say what to expect.** Attributes go out as their own request rather than
riding Step 7's 30-second event batch (on Android through WorkManager, so it needs
connectivity and retries on its own), which usually makes the person card the
first surface to light up. `Analytics.identify` rides the batch like every other
event.
