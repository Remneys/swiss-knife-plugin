---
name: swiss-knife
description: Entry point for the Swiss-Knife MCP server, which runs mobile builds, store uploads, TestFlight and Play releases, store listings, subscriptions, push, analytics and crash reports. Covers connecting the server (OAuth or token), starting with the help tool, choosing the right skill for a goal, relaying human_action blocks, and the approval-code flow. Use when the user mentions Swiss-Knife, asks what it can do, needs to connect it, or gives a mobile release or monetization goal that no more specific skill covers.
---

# Swiss-Knife

Swiss-Knife is an MCP server. Everything it does is a tool call; this skill only
gets you connected and pointed at the right skill.

**In ChatGPT** your tool list is short: the read tools, the in-chat cards, and
`build_start` and `build_retry`. You can look at anything the account can see and
propose a build. A person can also use the in-chat app's buttons to configure
apps and repositories, edit listings, manage support and send push notifications
with the required confirmation and approval. Open it with `dashboard_open`;
the app's buttons keep their own permissions and are not model tools. Builds > CI
accepts App Store Connect .p8 keys, Play service-account JSON, iOS P12 certificates
and Android keystores by private direct upload. Never ask for private key bytes
or passwords in chat or tool arguments. The workspace panel can reveal an SDK
key or mint a scoped CI token privately; Builds > CI can explicitly register a
machine a person operates, with the existing human approval. Builds > Sandbox
provides simulated-store controls for sandbox apps. Other credential kinds and actions the
app does not offer use a separate connected Codex or Claude Code session. When a step needs a tool you were not given, relay
any `human_action` and never guess a tool name or call a hidden write.

## 1. Connect

Check first: call `swiss-knife:workspace_whoami`. If it answers, you are connected;
check its scopes, then continue below. A successful read proves the connection,
not that this host exposes the setup tools.

Otherwise connect the server (the user runs this once):

```
claude mcp add --transport http swiss-knife https://swiss.remneys.com/api/mcp
```

Then `/mcp` -> swiss-knife -> Authenticate (a browser sign-in; OAuth, account-wide).
For CI or a machine with no browser, a workspace admin creates an `sk_mcp_…` token
and the server is added with
`--header "Authorization: Bearer $SWISS_KNIFE_MCP_TOKEN"`; the token is capped to
one workspace, a role and read/write scopes. Self-hosted: use your own origin plus
`/api/mcp`. The Claude Code plugin does all of this through its `.mcp.json`
(`SWISS_KNIFE_MCP_URL` overrides the origin). Codex:
`codex mcp add swiss-knife --url https://swiss.remneys.com/api/mcp`, then
`codex mcp login swiss-knife`; the Codex plugin does this for you.

`swiss-knife:workspace_whoami` returns your identity, role and scopes. A
`read`-scoped token cannot change anything; say so rather than trying.

**In Codex and Claude Code**, configuring builds is part of this plugin: discover
the exact schemas with `help {tool}` and use `app_create`,
`build_credentials_save`, `build_repo_connect`, `build_profile_upsert`,
`build_env_set` and `build_worker_register` as needed. Worker registration needs
a human approval code; never invent one. Reuse the existing app and signing
keys when migrating from local fastlane. Continue with `ship-ios` and/or
`ship-android` for prerequisites, preflight, build and upload verification.

If those tools are missing and `help` reports the ChatGPT surface, this is a
ChatGPT connector even if the surrounding desktop app is Codex. Report that
connection mismatch instead of claiming the local plugin is ready for setup.
Do not change the server's client restrictions or bypass them to continue.

## 2. Start with `help`

Call `swiss-knife:help` with no arguments: it lists the tool areas and the goals
with the recommended order of calls. With `goal`, `area` or `tool` it narrows to
one. Discover the current tool names rather than guessing.

## 3. Pick the skill for the goal

| The user wants to                                          | Use                                              |
| ---------------------------------------------------------- | ------------------------------------------------ |
| Set up the mobile SDK (Expo / React Native)                | `setup-sdk`                                      |
| Set up the SDK in native iOS or Android                    | `setup-sdk`, then `setup-native`                 |
| Ship an iOS build to TestFlight                            | `ship-ios`                                       |
| Ship an Android build to Play internal testing             | `ship-android`                                   |
| Edit store texts, translations, screenshots, age rating    | `store-listings`                                 |
| Submit for App Review, phased rollout, promote to users    | `submit-release`                                 |
| Products, subscriptions, entitlements, offerings, prices   | `monetization`                                   |
| Push notifications (APNs / FCM keys, sending)              | `push`                                           |
| Events, funnels, subscription stats, crashes, error alerts | `observability`                                  |
| Saved reports, cohorts, boards, anomalies, error merging   | `observability`                                  |
| Support inbox, replies, teammates, people                  | `support-inbox`                                  |
| Campaign links, install attribution, deferred deep links   | `attribution`                                    |
| Onboarding screens and quizzes shown to new users          | `onboarding-funnels`                             |
| Export-compliance hold on a TestFlight build               | `ship-ios`                                       |
| Is revenue data arriving, one-time purchases               | `monetization`                                   |
| Rehearse a release against a simulated store (sandbox)     | `sandbox`                                        |
| Build on this machine without a worker                     | `swiss-knife:help` with goal `ship_local_build`  |
| Save store or signing credentials                          | `swiss-knife:help` with goal `setup_credentials` |
| Anything else                                              | `swiss-knife:help` with `area`                   |

Workspace housekeeping with no skill of its own (tools, read `swiss-knife:help` with `area` for arguments): rotate an app's SDK key with `swiss-knife:app_key_rotate` (the old key stays valid as the previous key until you revoke it, so ship the new key in a build first; `swiss-knife:app_reveal_key` returns the new one), see and cut off connected assistants with `swiss-knife:connection_list` and `swiss-knife:connection_revoke`, change an app's attribution window, subscription transfer behavior or setup step with `swiss-knife:app_update`, and mint a scoped token for CI with `swiss-knife:app_token_mint` (`app_project_ids` up to 50 apps of one workspace; the role is clamped to yours and `roleClamped` says so). Members and invitations are in `support-inbox`. Key rotation, revoking a connection and minting a token change who can reach production data: confirm with the user first.

If two goals are mixed ("set up and ship", "ship and submit"), do them in that
order, one skill at a time.

## 4. Human steps: relay and stop

Some steps only a person can do (create the app record in a store, install the
GitHub App, create a store API key, answer a legal declaration). A blocked call
returns `error.details.human_action`, or a `human_action` field beside a result:
`{ who, what, url, why, verify_with }`.

1. Relay `who`, `what`, `url` and `why` to the user verbatim.
2. Stop. Do not retry, poll, or look for a workaround.
3. When the user says it is done, call `verify_with.tool` and continue only if it
   passes.

A `human_action` means only a person can clear it, whatever `error.code` says.

## 5. Approval codes

Calls that reach App Review, real users, testers, a store catalogue, Apple signing
objects, or that delete a stored credential need an `approval` code issued by a
signed-in human. You never issue or invent one.

1. Call `swiss-knife:approval_request` with the gated tool's name and the arguments
   you intend to use. It is read-only and returns what the human would approve, the
   exact page, and the call to make with the code.
2. Relay that to the user and stop.
3. When they give you the code, send the SAME call again with it as `approval`.

A code is bound to the exact action; if the arguments change, ask again.

## Rules

- Do everything a tool can do through the tools. Send the user somewhere, or ask
  them to do something by hand, only when a `human_action` says so.
- Retry a failed call only when `error.retryable` is true. `invalid_request`: fix
  the arguments. `precondition_failed`: change the state first.
