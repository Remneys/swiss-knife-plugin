---
name: setup-sdk
description: Provisions Swiss-Knife and wires its mobile SDK into an Expo or React Native app, end to end - adopts the user's existing workspace and app (creates one only when nothing exists), mints the project token, writes .mcp.json / .envrc / .env, installs the SDK, wires identity, analytics and the user's data, and runs a getOfferings() smoke test. Safe to re-run. Use when the user wants to set up, onboard, add or connect Swiss-Knife (subscriptions, analytics, crash reporting, install attribution) to a mobile project. Native iOS and Android continue in setup-native. Not for shipping builds (use ship-ios / ship-android).
---

# Swiss-Knife setup wizard

Provision the backend and wire the SDK into the user's project so that, at the
end, `Purchases.getOfferings()` returns a real offering. Be crisp: narrate each
step in one line, do the work with the tools, never dump raw JSON at the user.

The `swiss-knife` MCP server is the **control plane**: an account-level
connection you use to provision. Every tool named below is one of its tools, for
example `swiss-knife:workspace_whoami`. The project you write gets its own
**data-plane** `.mcp.json` (a per-app token).

**Reading the references.** Each step below names a file under `references/`.
Read it when you reach that step, not before. When this playbook arrives as the
`setup` MCP prompt, the same files follow this text, each under a
`<!-- references/<name>.md -->` marker; read them there.

**Two secrets, two purposes - never mix them up:**

| Secret            | Env var                          | Used by                   | Value shape |
| ----------------- | -------------------------------- | ------------------------- | ----------- |
| SDK API key       | `EXPO_PUBLIC_SWISSKNIFE_API_KEY` | the mobile app (SDK auth) | `sk_live_…` |
| Project MCP token | `SWISS_KNIFE_MCP_TOKEN`          | Claude Code (`.mcp.json`) | `sk_mcp_…`  |

The SDK key is client-embeddable: it ships inside the app bundle and is not a
server secret despite the "live" prefix. The MCP token is a real secret; keep it
private and never print either in full.

The **origin** is the deployment both point at: the SDK's `proxyURL` and the MCP
`url` (origin + `/api/mcp`). Self-hosters set `SWISS_KNIFE_MCP_URL` (plugin) and
`EXPO_PUBLIC_SWISSKNIFE_API_URL` (app, bare origin).

## The cardinal rule

**DISCOVER, THEN ADOPT, AND ONLY THEN CREATE.** Most users already have a
workspace and an app (the dashboard sign-up creates both). Calling
`swiss-knife:workspace_create` or `swiss-knife:app_create` unconditionally forks
their SDK data across two apps, invisibly. Creating is the fallback, never the
default.

## Checklist

Copy this and tick it off as you go:

```
- [ ] 1 Preflight: connect, detect the stack, read the project, list the account
- [ ] 2 Ask the user: identity (workspace/app) and which capabilities to turn on
- [ ] 3 Provision or adopt: workspace, app, SDK key, project token, optional seeds
- [ ] 4 Write .gitignore FIRST, then .mcp.json, .envrc, .env
- [ ] 5 Integrate the SDK (Expo / React Native), wire only what was ticked
- [ ] 6 Wire identity at the app's identity boundary
- [ ] 7 Wire the analytics boundary (React Native only)
- [ ] 8 Attach the user's own fields to the person card
- [ ] 9 Verify and hand off honestly
```

## Step 1 - Preflight (writes nothing)

Read [references/preflight.md](references/preflight.md). In short:

1. `swiss-knife:workspace_whoami`. On an auth error tell the user to approve the
   browser sign-in (or `/mcp` -> swiss-knife -> Authenticate), then retry.
2. Detect the stack from files, first match wins: Expo, bare React Native, native
   iOS, native Android, other. Native and other stacks use **setup-native** for
   Step 5 only; Steps 1-4, 6, 8 and 9 still apply to them.
3. Read bundle id and app name, take stock of what is already wired (a local
   ledger), find incumbent IAP, crash and analytics libraries and the app's
   `track` wrapper.
4. Enumerate the account (`swiss-knife:workspace_list`, `swiss-knife:app_list`)
   and rank apps against the project: key prefix, bundle id, name, unfinished
   onboarding.
5. Score the project's stage silently (POC or shipped) to decide which warnings
   to say.

## Step 2 - Ask the user

Read [references/questions.md](references/questions.md). Use `AskUserQuestion`,
at most 4 questions per call, in two calls:

- **Identity**: workspace, app, and (only when creating) name, platform, bundle
  id. Detected values are recommended options, never silent choices. A brand-new
  user with zero workspaces gets no adoption questions.
- **Capabilities**: Subscriptions, Analytics, Crash reporting, Performance tracing,
  Install attribution, and a starter offering. Subscriptions decides `configure` (core + store) versus
  `configureCore` (no StoreKit observer). Warn about an existing IAP library or
  crash reporter before ticking.

Stop and ask rather than invent a workspace, app or bundle name.

## Step 3 - Provision or adopt

Read [references/provision.md](references/provision.md). Call a `*_create` tool
only for something the user chose to create.

1. Workspace: adopt the chosen id, or `swiss-knife:workspace_create`.
2. App: adopt and recover the key with `swiss-knife:app_reveal_key`, or
   `swiss-knife:app_create` and capture `sdkKey` (shown once). A forbidden error
   means the user is a member, not an admin: say so, never create a duplicate.
3. `swiss-knife:app_token_mint` for the project token (`sk_mcp_…`, shown once,
   expires; report `expiresAt`).
4. Only if asked: seed a starter offering with `swiss-knife:monetization_setup`,
   after checking `swiss-knife:offering_list` is empty.
5. Only if crash reporting is ticked: one silent
   `swiss-knife:error_alert_create` rule, notifying the account holder's own
   email from `workspace_whoami`, never a guess.

## Step 4 - Write the config files

Read [references/config-files.md](references/config-files.md). Order matters:
`.gitignore` first, then secrets. Use the exact templates. If the Step 1 ledger
was non-empty, reconcile instead of rewriting
([references/reconcile.md](references/reconcile.md)).

## Step 5 - Integrate the SDK

Expo and React Native: read
[references/sdk-integration.md](references/sdk-integration.md). Install with
`npx expo install @app-swiss-knife/react-native-sdk`, create `src/swiss-knife.ts`
from the template, invoke it at the app root before render, and wire exactly the
capabilities ticked in Step 2. Native iOS, native Android or an unknown stack:
load the **setup-native** skill instead and continue at Step 6.

## Step 6 - Wire identity

Read [references/identity.md](references/identity.md). `Identity.logIn` means
"this device now belongs to this subject id", not "the human authenticated", so
wire it at app start for anonymous server users, not only at sign-in. Pick branch
A (real accounts), B (anonymous server users) or C (no server user) from the
user's answers. Never invent an auth system.

## Step 7 - Wire the analytics boundary

React Native only. Read
[references/analytics-boundary.md](references/analytics-boundary.md). Mirror the
app's central `track` wrapper into Swiss-Knife with one line, and tell the user
about any dev-build guard that would make a correct integration look dead.

## Step 8 - Attach the user's data

Read [references/user-data.md](references/user-data.md). Attach only fields you
saw by name; never a spread of the user object, never credentials, no PII by
default.

## Step 9 - Verify and hand off

Read [references/verify-handoff.md](references/verify-handoff.md). Typecheck,
state plainly that you cannot run the app here, tell the user the dev-client
launch should log `[swiss-knife] configured…`, say the Session Start line (until
something calls `Analytics.track`, the dashboard shows one event per launch),
list the capabilities wired, and separate adopted from created. Do not claim
the integration works from the files alone: say what you could not verify.

## Hard rules

- Never print a full `sk_live_` or `sk_mcp_` value; show prefixes only.
- Never `git add` secrets; `.env` is gitignored before it is written.
- A permission error on a mint or reveal is a stop-and-tell, not a workaround.
- Re-running is normal: observe, report what exists, apply only the deltas.

## Native and other stacks

Native iOS, native Android and unrecognised stacks use the sibling skill
**setup-native** in place of Step 5. If it is not available (you arrived through
the bare MCP prompt), say so and point the user at
`skills/setup-native/SKILL.md` in the public plugin repository
(`github.com/Remneys/swiss-knife-plugin`, or `npx skills add Remneys/swiss-knife-plugin`)
rather than improvising a native recipe from memory.
