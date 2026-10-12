## Reconciling an existing setup (idempotency)

The wizard is **re-runnable**. Running it twice must converge on the same single
app, never produce a second one — that is what Step 1's discovery is for.

Two independent kinds of "already exists", handled in different places:

- **Backend** (a workspace / app already in the account) → handled by Step 1.5 +
  Step 2: discover, rank, adopt. Never create a duplicate.
- **Local wiring** (files already in the repo) → the Step 1.4 ledger, reconciled
  here.

If the ledger was non-empty, do **not** clobber. Instead:

- **Report before you touch anything.** State, item by item, what exists and what
  you intend to do with it: `.mcp.json` server entry, SDK dependency,
  `src/swiss-knife.ts` (or the native equivalents), root-component wiring,
  whether `Identity.logIn` is called anywhere, and which `.env` keys are real vs
  placeholders (`YOUR_SDK_KEY`, `YOUR_MCP_TOKEN`, or the older `…_replace_me`).
  Never echo a full secret — prefixes only.
- **Reconcile the obvious deltas without asking.** Placeholder values get filled
  in; a missing `.gitignore` line gets appended; a missing root-component call
  gets added. These are unambiguously repairs.
- **Identity is the common re-run gap.** A project wired by an earlier version of
  this wizard has `configure` but no `Identity.logIn` — run Step 6 against it
  rather than declaring the setup complete. If `Identity.logIn` is already called,
  leave it alone; just name the call site in the summary.
- **`logIn` wired only at the sign-in handler is a re-run gap too.** Earlier
  versions of this wizard only knew branches A and C, so a freemium app was wired
  "after successful sign-in" and everything its logged-out users do is stranded.
  If you find `Identity.logIn` called **only** from a sign-in success path, and
  Step 6's question 1 comes back as branch B, that is a repair: add the app-start
  call, and flag the sign-out call site if it uses `logOut()` where
  `logIn(newAnonId)` belongs. Say what you changed and why.
- **`configure` in an app that also ships another IAP library is a repair.** If
  the project has `react-native-purchases` / `react-native-iap` / `adapty` /
  `qonversion` / `glassfy` in `package.json` **and** an existing
  `SwissKnife.configure(...)` (or `Purchases.configure(...)`, the same function
  under its RC-compatible name), two transaction observers are racing in
  production today. Surface it — quote the Step 2 hazard — and offer switching to
  `configureCore`. Do not switch it silently: if Swiss-Knife genuinely owns
  purchases there, `configure` is right and the other library is the problem.
  That is the user's call, not yours.
- **Capabilities are the other re-run gap.** A project wired by an earlier version
  has a bootstrap call and nothing else — spelled `Purchases.configure` if it
  predates SDK 0.3.0, `SwissKnife.configure` if not — no `Analytics.init`, no
  `CrashReporting.init`, no `<SKErrorBoundary>` — even though the user asked for
  them. Add the ones they ticked this time that aren't there yet. Never delete a
  capability just because they left the box unticked on a re-run: unticking is
  "don't add", not "rip out". If they want one removed, they will say so.
- **"`Analytics.init` is there but nothing calls `Analytics.track`" is the newest
  re-run gap — and the one most likely to be reported as a bug.** Every project
  wired by a wizard older than Step 7 is in exactly this state: the pipe is
  correct, the dashboard shows one `[SwissKnife] Session Start` per launch, and it
  reads as a dead integration. Grep for `Analytics.track(` / `Analytics.trackScreen(`
  outside `src/swiss-knife.ts`; if there are no call sites, run Step 7 against the
  project rather than reporting the setup complete. Same for a mirror that exists
  but sits **below** a dev guard while the user is testing on a dev build — that
  is a decision to re-surface, not a defect to silently flip.
- **"`Identity.logIn` is wired but the person card is empty" is the Step 8 re-run
  gap.** Grep for `setAttributes(` / `setEmail(` outside `src/swiss-knife.ts`; if
  there are none, the dashboard shows a column of bare ids and it will be reported
  as missing data. Run Step 8 rather than calling the setup complete. While you
  are there: a `CrashReporting.setUser({ id })` that only passes the id is now
  redundant — the subject is pulled from `Identity` — so mention it, but only
  remove it if they say so.
- **A missing `serverURL` on an existing init is a repair, not a preference.**
  If you find `CrashReporting.init` or `Performance.init` in the project without
  a `serverURL`, that app is shipping its data to `api.swiss-knife.dev`. Fix it
  and say what you fixed.
- **Ask (via `AskUserQuestion`) before anything destructive or ambiguous** —
  above all when the local `sk_live_…` prefix points at a **different** app than
  the one being adopted. That means this project was previously wired to another
  app: surface both, and let the user choose which one wins. Never silently
  repoint a project at a different app.
- Other choices worth offering: **Re-mint the project token** (call
  `app_token_mint` again, update only `SWISS_KNIFE_MCP_TOKEN` in `.env`),
  **Recover the SDK key** (`app_reveal_key`, update
  `EXPO_PUBLIC_SWISSKNIFE_API_KEY`), **(Re)seed the starter offering** (only into
  an empty catalogue — check `offering_list` / `product_list` first), or
  **Repair a specific file**.
- Apply only the chosen change; leave the rest untouched. Never overwrite a
  populated `.env` or an edited `App.tsx` wholesale — read, then surgically Edit.
- **Finish with a reconciliation summary**: what you found, what you changed, what
  you deliberately left alone.
