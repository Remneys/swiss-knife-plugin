---
name: swiss-knife
description: Entry point for the Swiss-Knife MCP server, which runs mobile builds, store uploads, TestFlight and Play releases, store listings, subscriptions, push, analytics and crash reports. Covers connecting the server (OAuth or token), starting with the help tool, choosing the right skill for a goal, handling real human prerequisites, and delegated write permissions. Use when the user mentions Swiss-Knife, asks what it can do, needs to connect it, or gives a mobile release or monetization goal that no more specific skill covers.
---

# Swiss-Knife

**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

Swiss-Knife is an MCP server. Everything it does is a tool call; this skill only
gets you connected and pointed at the right skill.

**In ChatGPT** your tool list is short: the read tools, the in-chat cards, and
`build_start` and `build_retry`. You can look at anything the account can see and
propose a build. A person can also use the in-chat app's buttons to configure
apps and repositories, edit listings, manage support and send push notifications
with one in-chat preview and confirmation, without a copied dashboard code. Open it with `dashboard_open`;
the app's buttons keep their own permissions and are not model tools. Builds > CI
accepts App Store Connect .p8 keys, Play service-account JSON, iOS P12 certificates
and Android keystores by private direct upload. Never ask for private key bytes
or passwords in chat or tool arguments. The workspace panel can reveal an SDK
key or mint a scoped CI token privately; Builds > CI can explicitly register a
machine a person operates, after confirming the registration in its card. Builds > Sandbox
provides simulated-store controls for sandbox apps. The app also accepts push,
Git and RevenueCat credentials, private build environment values and outgoing
webhook setup. Its workspace panel offers private receiver addresses, one-time
webhook signing keys and SDK-key rotation/revoke, retaining the administrator role and one in-chat confirmation. Workspace notifications can also connect or replace Slack through a
private input, test the hook before saving, and disconnect it after review.
Support > Inboxes can privately connect Telegram, WhatsApp and SMS, update
channel credentials with an explicit replacement confirmation, and copy newly generated
inbound secrets privately. Never copy these values into chat or tool arguments.
Actions the app does not yet offer use a separately connected Codex or Claude
Code session. When a step needs a tool you were not given, relay
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
`build_env_set` and `build_worker_register` as needed. Worker registration uses the connected admin permissions; no copied code is needed in Codex or Claude Code. Reuse the existing app and signing
keys when migrating from local fastlane. Continue with `ship-ios` and/or
`ship-android` for prerequisites, preflight, build and upload verification.

Codex may omit individual tools to fit its tool-list size limit. When a tool is
not listed, call `help {tool: "exact_name"}`, parse its `input_schema` JSON, then
use `tool_read {tool, arguments}` for reads or `tool_call {tool, arguments}` for
writes. These gateways preserve the target's real permissions and validation.
They do not accept app-only UI tools and are unavailable to ChatGPT connections.
A missing individual tool is not a reason to stop or ask the user to reinstall.

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

Workspace housekeeping with no skill of its own (tools, read `swiss-knife:help` with `area` for arguments): rotate an app's SDK key with `swiss-knife:app_key_rotate` (the old key stays valid as the previous key until you revoke it, so ship the new key in a build first; `swiss-knife:app_reveal_key` returns the new one), see and cut off connected assistants with `swiss-knife:connection_list` and `swiss-knife:connection_revoke`, change an app's attribution window, subscription transfer behavior or setup step with `swiss-knife:app_update`, and mint a scoped token for CI with `swiss-knife:app_token_mint` (`app_project_ids` up to 50 apps of one workspace; the role is clamped to yours and `roleClamped` says so). Members and invitations are in `support-inbox`. Key rotation, revoking a connection and minting a token change access to production data: perform them only within the user's explicit request, retaining existing authorization when the target and scope are already clear.

If two goals are mixed ("set up and ship", "ship and submit"), do them in that
order, one skill at a time.

## 4. Real human prerequisites

Some steps require a person or an external account action: accepting agreements, pairing a phone, granting store access or supplying facts for a legal declaration. A blocked call can return `error.details.human_action`, or a `human_action` beside the result: `{ who, what, url, why, verify_with }`.

Relay the actual required step and why it is needed. Pause only work that depends on it and continue independent authorized work. Do not repeat an unchanged refusal or bypass permissions. After completion, use `verify_with.tool` and continue when it passes. Reuse factual answers the user already supplied; ask only for missing facts.

Read the whole result: `approval_required: false` is informational, not a reason to stop or demand a dashboard code. A role or scope refusal requires the stated access; it cannot be solved by inventing an approval.

## 5. Authorization

Codex and Claude Code use the write access granted at connection time. The server checks current workspace membership, token role and allowed apps on every call. Carry out the user's request with the appropriate tools and verify the result. Omit `approval` on direct calls. Do not ask for another yes when the user already authorized the operation.

Use `approval_request` only to inspect a restricted connection's actual approval refusal. When `approval_required` is false, call the action directly. If it is true, relay the returned human step; never invent a code, session or permission. A supplied code is still verified even on a delegated connection.

## Rules

- Do everything a tool can do through the tools. Send the user somewhere, or ask
  them to do something by hand, only when a `human_action` says so.
- Retry a failed call only when `error.retryable` is true. `invalid_request`: fix
  the arguments. `precondition_failed`: change the state first.
