## Step 6 — Wire user identity (which subject does this device belong to?)

### Contents

- 1. Find the identity boundary
- 2. Ask the four questions (and 2b: is the id guessable?)
- 3. Wire the branch: A real accounts, B anonymous server users, C no server user
- 4. What `logIn` does, ordering and the merge
- Native equivalents

Everything Step 5 wired is **device-scoped**: purchases, entitlements, events and
attribution all hang off an anonymous device id (`$SKAnonymousID:<deviceId>`,
Keychain on iOS / EncryptedSharedPreferences on Android). Until you call
`Identity.logIn(...)`, Swiss-Knife has no idea that the device belongs to one of
_your_ subjects, so a subscriber who reinstalls or picks up a second device looks
like a brand-new stranger, and the dashboard cannot join their activity to a
person. This step closes that gap.

> **Read this before you wire anything: `logIn` does NOT mean "the human
> authenticated."** The name is RevenueCat parity; the contract is narrower and
> more useful. It means _"from now on, this device's activity belongs to this
> subject id."_ Nothing about it asserts that anyone proved who they are.
>
> So it is entirely legitimate — and for most freemium apps it is **correct** —
> to call it with the id of an **anonymous** subject, long before any sign-in
> exists. If your server already has a row for this person (progress, purchases,
> a deviceId-keyed id) while they are logged out, that row's id is what belongs
> here, from the very first launch.
>
> Wiring `logIn` only "after a successful sign-in" is the single most common
> mistake in this step. It strands everything the person did before signing in,
> and for an app where most users never sign in at all, it strands almost
> everything.

This step applies to **every** stack, including the native iOS / Android recipes
below.

**1. Find the identity boundary — read the project, don't guess.** Use Grep /
Glob / Read, exactly like the stack detection in Step 1.2. You are looking for
**two** things, not one: where a session is _authenticated_, and where a subject
id is _resolved at app start_ (which in a freemium app is the more important of
the two).

- **An auth context or provider** — `AuthProvider`, `AuthContext`, `SessionProvider`,
  or a hook like `useAuth` / `useSession` / `useUser`.
- **A third-party auth client in `package.json`** — `@supabase/supabase-js`,
  `firebase` / `@react-native-firebase/auth`, `@clerk/clerk-expo`,
  `react-native-auth0`, `@aws-amplify/auth`, `better-auth`. Each has a canonical
  hook: Supabase `supabase.auth.onAuthStateChange`, Firebase `onAuthStateChanged`,
  Clerk `useUser()` / `useAuth()`, Auth0 `useAuth0()`.
- **A sign-in screen** — files matching `*(login|sign-?in|auth)*` under `app/`,
  `src/`, or `screens/`.
- **A session store** — a zustand / redux / jotai slice named `user`, `auth`, or
  `session`, or a `SecureStore` / `AsyncStorage` key holding a token.
- **An app-start bootstrap that fetches "me"** — this is the freemium tell. A
  `/me`, `/session`, `/users/register-device`, `/bootstrap` or `/init` call fired
  on launch that returns a user id **without anyone having signed in**; a
  `deviceId` / `installId` generated locally and POSTed to the server; a
  `useCurrentUser()` that resolves for logged-out users too. If you find one, the
  app almost certainly has **anonymous server users** — branch B below.
- **Native**: Swift `AuthManager` / `Auth.auth().addStateDidChangeListener`;
  Kotlin `AuthRepository` / `FirebaseAuth.getInstance()`.

Report what you found in one line, naming files — e.g. _"Found
`src/context/AuthContext.tsx` exporting `useAuth()` with `signIn` / `signOut`, and
`src/api/bootstrap.ts` calling `POST /me` on launch with a stored `deviceId` —
that second one suggests anonymous users already exist server-side."_ If you found
nothing, say that plainly too.

**2. Ask (`AskUserQuestion`) — up to 4 questions in ONE call.** The tool allows
at most **4 per call** and Step 2 spends its own budget in separate calls, so
these four fill this one exactly. Ask them together; do not split them, and do not
add a fifth — question 4 is Step 8's only decision, riding along here so Step 8
costs no call of its own.

**Question 1 — "What does your backend know about the current user?"** This is
the branch selector. Offer exactly these, and read the labels carefully because
the middle one is the one everybody gets wrong:

| Option                                                                                                                                      | Branch |
| ------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| **"Real accounts only — a user exists on my server only after they sign in"**                                                               | **A**  |
| **"Every user is a row on my server from first launch — anonymous ones included"** _(recommend if the app-start bootstrap above was found)_ | **B**  |
| **"Nothing server-side — there is no user record at all, signed in or not"**                                                                | **C**  |
| **"Skip — I'll wire it myself later"**                                                                                                      | —      |

Branch **B** is the most common shape in freemium apps and the one the old
version of this wizard could not express. If the app has a row with progress and
purchases tied to a `deviceId`, that is B — **not** C. C means there is genuinely
no server-side subject, only a device-local fiction.

**Question 2 — "When someone signs in, does the user id your app holds change?"**
This is a **backend** question, and the answer decides whether there is anything
to wire at the auth boundary at all. Do not try to infer it from the client code;
ask. Offer:

- **"Yes — sign-in can return a different id than the one we had"** (the usual
  answer for B: an existing account is matched by email, so the anonymous id is
  replaced)
- **"No — the same id survives sign-in, we just attach an email to it"**
- **"Sometimes — it depends how sign-in resolves"** ← for B this is very often
  the true answer; treat it as "Yes"
- **"There is no sign-in"**

**Question 3 — "Where should I wire it?"** Offer, in this order:

1. **Each detected call site**, best first and pre-selected, labelled with the
   reason — e.g. _"`src/api/bootstrap.ts` — resolves the user id on launch"_ or
   _"`src/context/AuthContext.tsx` — `signIn()` / `signOut()` pair"_.
2. **"Somewhere else"** via "Other", so they can name a file you missed.
3. **"Skip — I'll wire it myself later"**.

**Question 4 — "Besides the id, what should I attach to this person?"** This is
Step 8's decision, asked here. Build the options from the fields you actually read
off the user object in Step 1.4 — **name them**, don't say "profile fields" — and
if the app already calls an incumbent people API, offer what that call sends as
the recommended option ("mirror what you already send to Mixpanel: `plan`,
`signup_date`"). Offer, in this order:

1. **"Just the id"** — nothing beyond `Identity.logIn`. Always a valid answer.
2. **"`<the non-PII fields you found>`"** — plan / tier / cohort / locale /
   signup date. Recommended when they exist.
3. **"Also email and name"** — labelled as what it is: _"PII. Attributes are
   shown verbatim on the person card in the dashboard and travel with crash
   reports."_ Never pre-select this one.
4. **"Skip — I'll attach data myself later"**.

**Two things you may never put in these options.** Never offer a credential field
(`accessToken`, `refreshToken`, `idToken`, `password`, `secret`, `session`) even
if the user object carries one, and never offer an "everything on the user object"
option — see Step 8's two hard rules, which hold no matter what is picked here.

Skip the whole call only when everything is unambiguous (exactly one boundary,
and the user has already told you the shape and the id behaviour in
conversation) — then just state what you're wiring and do it.

**2b. Check the id itself is not guessable — and say so if it is.**

> **The app user id is a lookup key, not a secret — but it must not be
> _guessable_.** Subscription and entitlement state is readable by app user id
> using the SDK key, and the SDK key ships inside the app bundle, so everyone
> who downloads the app has it. That is the same trade RevenueCat makes, and
> like them we mitigate it with the shape of the id rather than with a token
> exchange.
>
> When you see what the app's user ids look like in Step 1.4, check them:
>
> - **Good** — a random, unguessable identifier: a UUID v4, a ULID, a Firebase
>   uid, any opaque server-issued token.
> - **Bad** — a **sequential integer** (`1`, `2`, `1043`). An autoincrement
>   primary key lets anyone walk the whole customer base one id at a time.
> - **Bad** — an **email address**. Not secret, frequently known to exactly the
>   people who should not be reading someone's subscription record, and it puts
>   PII in your URLs and logs.
>
> If their ids are one of the bad shapes, **say so once, plainly, and move on**
> — do not block the integration and do not offer to migrate their user table.
> The right fix is usually a separate opaque `analytics_id` column on their user
> row that they pass to `Identity.logIn` instead of the primary key. Suggest it,
> note it in the Step 9 summary, and continue. The dashboard also surfaces this
> as a warning on the SDK Users page once real ids start arriving.

**3. Wire the branch they picked.** Append the helpers to `src/swiss-knife.ts`
(verbatim; the same helpers serve all three branches — what differs is where you
call them and which one you call on sign-out):

```ts
// src/swiss-knife.ts (append) — widen the existing import rather than adding a
// second one from the same module:
//   import { SwissKnife, Identity } from "@app-swiss-knife/react-native-sdk";

/**
 * Bind this device to a subject id. Call it whenever you KNOW the id — at app
 * start, after sign-in, after a token refresh. The id may belong to an
 * anonymous subject; that is a supported and often correct use.
 *
 * Safe and cheap to call repeatedly: a repeat call with the id already in
 * effect is a client-side no-op — no bridge hop, no network request — so you
 * never need to guard it with "did this change?" logic.
 */
export async function identifySwissKnifeUser(userId: string) {
  try {
    await Identity.logIn(userId);
  } catch (err) {
    // Non-fatal — the SDK keeps working under the previous subject. Never block
    // your login UI on this. It IS worth retrying on the next app start: a
    // rejection means the device was identified locally but NOT merged
    // server-side.
    console.warn("[swiss-knife] Identity.logIn failed:", err);
  }
}

/**
 * Drop the subject entirely and revert to the anonymous device id.
 *
 * Only correct when there is genuinely no subject any more. If signing out
 * returns the user to an ANONYMOUS SERVER USER, do NOT call this — call
 * identifySwissKnifeUser(newAnonymousUserId) instead. See Step 6 branch B.
 */
export async function forgetSwissKnifeUser() {
  try {
    await Identity.logOut();
  } catch (err) {
    console.warn("[swiss-knife] Identity.logOut failed:", err);
  }
}
```

### Branch A — real accounts only (a user exists only after sign-in)

Edit the auth boundary — read it first and match its style:

```ts
// e.g. src/context/AuthContext.tsx
import { identifySwissKnifeUser, forgetSwissKnifeUser } from "../swiss-knife";

// …after your own sign-in resolves:
await identifySwissKnifeUser(user.id);

// …and on sign-out, before you clear local session state:
await forgetSwissKnifeUser();
```

If the app restores a session on launch (the common case), also call
`identifySwissKnifeUser(user.id)` from wherever that restore lands — an
`onAuthStateChange` / `onAuthStateChanged` subscriber is the natural home, and it
covers sign-in, restore and token refresh in one place.

Here `logOut()` **is** the right call: after sign-out there is no subject, and
the device correctly reverts to its anonymous id.

### Branch B — anonymous server users (the freemium shape)

The app has real users on the server before anyone signs in: a row carrying
progress and purchases, keyed to a deviceId. That row's id is a subject, so give
it to Swiss-Knife immediately.

**The recommendation, and it is not the same as branch A's:**

> **Call `Identity.logIn(serverUserId)` on every app start — including for an
> anonymous user — and re-call it whenever the id changes.**

Not "after a successful sign-in". At app start, as soon as your bootstrap knows
the id. Wire it where the id is _resolved_, not where a password is checked:

```ts
// e.g. src/api/bootstrap.ts — wherever the app resolves "who is this" on launch
import { identifySwissKnifeUser } from "../swiss-knife";

const user = await fetchCurrentUser(); // anonymous or signed-in, doesn't matter
await identifySwissKnifeUser(user.id);
```

Then re-call it from the auth boundary **only if the id can change** (Question 2).
Their sign-in resolves three ways, and only two of them need anything:

| Sign-in resolves to…                                    | Does `userId` change? | What to do                                     |
| ------------------------------------------------------- | --------------------- | ---------------------------------------------- |
| An existing account matched by email                    | **Yes**               | `logIn(newId)` — and see the server note below |
| No email match, but this device's anonymous user exists | **No**                | Nothing. The repeat call no-ops for free       |
| Neither — a fresh account is created                    | **Yes**               | `logIn(newId)` — and see the server note below |

Because the middle case is a no-op, the safe wiring is simply to call
`identifySwissKnifeUser(currentUser.id)` again after sign-in resolves,
unconditionally. If the id didn't change, the call costs nothing.

> **⚠️ The client call is not sufficient when the id changes. Say this out loud —
> it is the one thing that will silently lose their data.**
>
> `Identity.logIn(...)` always posts the device's anonymous id as the merge
> source — `anonymous_id` is `$SKAnonymousID:<deviceId>` — and never the previous
> **server** id. The backend only accepts `anonymous_id` as a merge source while
> that device row is still unidentified. Once your first `logIn("anon_A")` has
> bound the device to `anon_A`, a later `logIn("acct_B")` finds no anonymous
> source and **does not merge** — everything the person did as `anon_A` stays on
> `anon_A`.
>
> So when the id changes, **your backend** must also call the identify endpoint
> with the previous subject — and it must be your backend, because this call
> requires an agent credential that must never ship inside the app: an `sk_mcp_`
> token, minted by an MCP session with the `app_token_mint` tool (scope
> `write`, role `admin`, allow-listed to this one app; it is shown once, so hand
> it straight to whoever puts it in the backend's secrets). There is no other kind of key to mint: the old
> `sk_secret_` management keys are retired and answer 401 `credential_retired`.
>
> ```http
> POST /api/sdk/users/identify
> Authorization: Bearer <your sk_mcp_… token>
> Idempotency-Key: <stable key for this transition>
>
> { "previous_user_id": "anon_A", "user_id": "acct_B" }
> ```
>
> A token allow-listed to exactly one app needs nothing more. A token that spans
> several apps names the app with an `X-App-Project-Id` header (or an
> `app_project_id` field in the body); without it the call is a 400.
>
> **`previous_user_id` is server-to-server only.** Sending it with the SDK key
> (`X-API-Key`) is a **403 `previous_user_id_requires_secret_key`** (the code's
> name is kept for existing clients; the credential it asks for is the
> `sk_mcp_` token). The reason is simple: `previous_user_id` means "move
> everything from subject X to subject Y", the SDK key ships inside your app
> bundle, and anyone who downloads the app has it — so accepting that pair from
> a client would let a stranger drain any user id they can guess. Mint an
> `sk_mcp_` token as above and keep it on your server. The `anonymous_id` form
> stays client-callable and is what `Identity.logIn` uses; only the
> subject-to-subject form is restricted.
>
> That is the call that re-points events, sessions, subscriptions, store
> transactions, paywall/funnel events, experiment assignments, attribution and
> consumable balances from the old subject onto the new one, then deletes the old
> row. Send exactly one of `previous_user_id` / `anonymous_id` — both is a 400
> `ambiguous_source`, neither is a 400 `missing_source`. Passing a
> `previous_user_id` equal to `user_id` is an accepted no-op (200), so it is safe
> to fire even when you are not sure the id actually changed. A genuine failure
> is a **500 `merge_failed`**
> with `retryable: true` — it is not silently swallowed, so treat it as a real
> error and retry. Replays of the same idempotency key return the original
> response verbatim.
>
> Do not offer to write this for them — it lives in their backend, which you are
> not touching. State it as a required follow-up, name the endpoint and the field,
> and put it in the Step 9 summary.

**Sign-out in branch B is where the old docs were simply wrong.** If signing out
drops the person back to an anonymous server user — which is what a freemium app
does — then there is still a subject, so `logOut()` is the wrong call. It would
discard a perfectly good id and revert to the raw device id:

```ts
// …on sign-out, if your server hands back a NEW anonymous user:
const anon = await createAnonymousUser();
await identifySwissKnifeUser(anon.id); // NOT forgetSwissKnifeUser()
```

Use `forgetSwissKnifeUser()` / `logOut()` only when signing out genuinely leaves
**no subject at all**. Note that this transition changes the id too, so it needs
the same server-side `previous_user_id` call as any other id change.

### Branch C — no server-side user record at all

**This is not a failure, and you must not invent an auth system to satisfy it.**
Do not add a login screen, do not install an auth library, do not generate a fake
user id (a random UUID per install is just a worse device id). Say what stays
device-scoped, and move on to Step 7 (the analytics boundary):

> Everything keeps working — purchases, entitlements, offerings, analytics and
> attribution are all keyed to the anonymous device id. What you give up until
> server-side users exist: activity can't follow a person across devices or
> reinstalls, restore-purchases leans entirely on the store account rather than
> your user id, and the dashboard's person view shows one row per device. When
> you do add them, a single `Identity.logIn(userId)` at that point back-fills
> this device's anonymous history onto the new subject — nothing recorded now is
> lost.

Be careful not to land here by mistake. If the app has any server-side row for a
logged-out user, that is **branch B**, and treating it as C throws away the exact
merge this step exists to set up.

**Which id to pass.** Your backend's **stable primary key** for the subject —
Supabase `user.id`, Firebase `uid`, Clerk `user.id`, your own users table id.
**In branch B, the id of the anonymous user row is exactly right** — it is a real
primary key on a real row, which is all this needs. Not an email, phone number,
or username: those rotate, and the value becomes the app-user's
`external_user_id`, the join key the dashboard uses to resolve a person. It is
not a secret, but it IS shown in the dashboard, so don't pass a session token or
an access JWT. And don't pass the raw device id — that is what the SDK already
uses when there is no subject, so it adds nothing.

**Ordering.** `Identity.logIn` requires the **core** to be configured — native
throws `notConfigured` otherwise. Either bootstrap satisfies this: `configure`,
`configureCore`, or `Analytics.init` all bring core up. Step 5.3 calls
`initSwissKnife()` at the root, which happens before any auth callback or
bootstrap fetch can fire, so the ordering is already correct; just don't move the
identify call above the bootstrap call.

**4. Say what `logIn` actually does, so nobody expects magic.** It POSTs
`(anonymous_id, user_id)` to `/api/sdk/users/identify`; the backend **merges** the
anonymous record into the identified one — its events, sessions and subscriptions
are re-pointed at the identified subject, and the anonymous person is folded into
the identified person. So anonymous history is preserved, and an entitlement
bought before sign-in still resolves afterwards. It then invalidates the
entitlements cache and refreshes `CustomerInfo` under the new subject (so user
A's entitlements can't leak into user B's session), and resolves once both have
landed.

**The merge is device-anonymous → subject, and it fires once.** The client only
ever sends the device's `anonymous_id`, so the very first `logIn` on a device
does a real merge and every later `logIn` with a _different_ id does not — the
device row is no longer anonymous, so there is nothing for that path to pick up.
Subject-to-subject re-pointing is the `previous_user_id` path, and only the app's
own backend can drive it (branch B above). If you take one thing from this step:
**the client call establishes the subject; the server call moves the history.**

Repeat calls are free — a `logIn` with the id already in effect returns from a
JS-side cache with no bridge hop and no network request. It is deliberately safe
to call on every app start with no guard. A `logOut()`, `reset()`, `wipe()`, a
different id, or a previous `logIn` that rejected all force a real call again.

What it deliberately does **not** do:

- **It does not emit an analytics `$identify` event.** Identity is decoupled from
  analytics on purpose. Events pick up the new `user_id` automatically from the
  native identity snapshot, but user _properties_ are yours to send:
  `Analytics.identify(new SKIdentify().set("plan", "pro"))`, separately and only
  if you want them.
- **`SwissKnife.configure({ appUserID })` is not an identify.** It only seeds the
  local identity snapshot at process start — it never POSTs to
  `/api/sdk/users/identify`, so no anonymous→identified merge happens. `configure`
  stays the entry point; `Identity.logIn` is the auth boundary. Don't substitute
  one for the other.
- **`reset()` and `wipe()` are not sign-out.** `Identity.reset()` rotates the
  in-memory identity but keeps the persisted device id ("switch test user" UX);
  `Identity.wipe()` also clears the persisted device id (GDPR delete / test-build
  cleanup, and it makes the install look brand new). Sign-out is `logOut()` —
  but only in branch A or when no subject remains. In branch B, sign-out is
  `logIn(newAnonymousUserId)`.

> **Migrating from RevenueCat?** `Purchases.logIn(appUserID)` and
> `Purchases.logOut()` still exist as RC-compatible aliases and route through the
> same native Identity path, so a straight port keeps working. They are
> **`@deprecated`** — rename them to `Identity.logIn` / `Identity.logOut` as you
> go, and write `Identity.*` in anything new.

**Native equivalents** (same semantics, same three branches):

```swift
// whenever you know the subject id — app start AND after sign-in — SwissKnifeiOS
try await SwissKnife.Identity.logIn("user_12345")
// branch A sign-out (no subject remains)
try await SwissKnife.Identity.logOut()
// branch B sign-out (back to an anonymous server user)
try await SwissKnife.Identity.logIn(newAnonymousUserId)
```

```kotlin
// whenever you know the subject id — com.appswissknife.identity.Identity
Identity.logIn(
  userId = "user_12345",
  onSuccess = { /* subject bound */ },
  onError = { error -> /* non-fatal: log it, don't block the UI */ },
)
// branch A sign-out
Identity.logOut(onSuccess = { }, onError = { error -> })
// branch B sign-out: Identity.logIn(newAnonymousUserId, …) instead
```
