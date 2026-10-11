---
name: swiss-knife
description: Entry point for the Swiss-Knife MCP server, which runs mobile builds, store uploads, TestFlight and Play releases, store listings, subscriptions, push, analytics and crash reports. Covers connecting the server (OAuth or token), starting with the help tool, choosing the right skill for a goal, relaying human_action blocks, and the approval-code flow. Use when the user mentions Swiss-Knife, asks what it can do, needs to connect it, or gives a mobile release or monetization goal that no more specific skill covers.
---

# Swiss-Knife

Swiss-Knife is an MCP server. Everything it does is a tool call; this skill only
gets you connected and pointed at the right skill.

## 1. Connect

Check first: call `swiss-knife:workspace_whoami`. If it answers, you are connected;
skip to step 2.

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

## 2. Start with `help`

Call `swiss-knife:help` with no arguments: it lists the tool areas and the goals
with the recommended order of calls. With `goal`, `area` or `tool` it narrows to
one. The server has about 200 tools; never guess a name.

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
| Rehearse a release against a simulated store (sandbox)     | `sandbox`                                        |
| Build on this machine without a worker                     | `swiss-knife:help` with goal `ship_local_build`  |
| Save store or signing credentials                          | `swiss-knife:help` with goal `setup_credentials` |
| Anything else                                              | `swiss-knife:help` with `area`                   |

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

- Do everything a tool can do through the tools; do not send the user to a settings
  page or tell them to click anything unless a `human_action` says so.
- Retry a failed call only when `error.retryable` is true. `invalid_request`: fix
  the arguments. `precondition_failed`: change the state first.
