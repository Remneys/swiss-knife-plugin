# Swiss-Knife plugin

Ship and monetize mobile apps from chat. This plugin connects ChatGPT, Codex
and Claude Code to [Swiss-Knife](https://swiss.remneys.com): iOS and Android
builds, TestFlight and Google Play releases, store listings, subscription
products, push, and crash and revenue analytics, through the Swiss-Knife MCP
server (`https://swiss.remneys.com/api/mcp`).

It bundles two things:

- **14 agent skills** that encode the order of operations and the stores' real
  pitfalls, so the model follows a tested path instead of guessing.
- **The MCP server connection**, with account sign-in (OAuth). You need a
  Swiss-Knife account; the plugin only sees the workspaces and apps that account
  can open.

| Skill                | Use it to                                                                                                 |
| -------------------- | --------------------------------------------------------------------------------------------------------- |
| `swiss-knife`        | Connect the MCP server, start with `help`, pick a skill, relay human steps and approvals.                 |
| `setup-sdk`          | Provision a workspace and app and wire the mobile SDK (Expo / React Native).                              |
| `setup-native`       | The native iOS / Android half of the setup wizard (the SDK sources vendored from the public npm package). |
| `ship-ios`           | Build an iOS app and put it in TestFlight, checked against App Store Connect.                             |
| `ship-android`       | Build an Android app and put it on the Play internal track, checked against Google Play.                  |
| `store-listings`     | Edit listing text, translations, screenshots and age rating on both stores, checked against the store.    |
| `submit-release`     | Submit for review, phased rollout and promotion, with human approval codes.                               |
| `monetization`       | Products, entitlements, offerings and the store catalogue push.                                           |
| `push`               | Push credentials (APNs, FCM), sending with a dry run, and device hygiene.                                 |
| `observability`      | Events, funnels, subscription and crash stats, error issues and alerts.                                   |
| `attribution`        | Campaign links, install attribution, deferred deep links and campaign subscription results.               |
| `onboarding-funnels` | Create, publish and measure server-driven onboarding screens and quizzes.                                 |
| `support-inbox`      | Support conversations, replies, internal notes, teammates and triage.                                     |
| `sandbox`            | Rehearse a release against a simulated App Store Connect and Google Play: review, time and faults.        |

Sensitive actions keep their server-side permission and approval checks. Where
an approval code is required, a workspace administrator issues it for that exact
action; it expires after 15 minutes. In ChatGPT, available write controls also
require the app's review and confirmation flow, and administrator approval and
workspace opt-in where applicable.

## Install

### Codex (app and CLI)

Needs Codex 0.162 or newer. Older versions (checked on 0.141) cannot read a plugin
that sits at the root of a marketplace repository and list nothing from this one;
update Codex, or use the MCP-only route below, which works from 0.141.

```bash
codex plugin marketplace add Remneys/swiss-knife-plugin
codex plugin add swiss-knife@swiss-knife
codex mcp login swiss-knife
```

In the Codex desktop app: **Plugins, Add, Add a marketplace**, source
`Remneys/swiss-knife-plugin`, then install **Swiss-Knife** and sign in when
asked. Update later with `codex plugin marketplace upgrade swiss-knife`.

### Claude Code

```text
/plugin marketplace add Remneys/swiss-knife-plugin
/plugin install swiss-knife@app-swiss-knife
```

Then try `/swiss-knife:setup-sdk`, or ask "ship my app to TestFlight". To point
the plugin at a self-hosted deployment, set `SWISS_KNIFE_MCP_URL` (your origin
plus `/api/mcp`) before launching Claude Code.

### ChatGPT

The [latest release](https://github.com/Remneys/swiss-knife-plugin/releases/latest)
provides `swiss-knife-plugin-<version>.zip`, with `plugin.json` at its root, for
plugin testing and submission. Use the installation flow available to your
account; see OpenAI's [connect and test guide](https://developers.openai.com/plugins/deploy/connect-chatgpt).
Sign in with your Swiss-Knife account when prompted.

The in-chat app reads build and release status, prerequisites, store data,
analytics, revenue, errors and performance traces. Its controls also support
app/repository configuration, build starts and retries, store listings and
supported release operations, products, and private credential and file uploads.
For example, Builds > CI accepts signing/store credentials and IPA/AAB files;
Revenue > Products accepts RevenueCat credentials; Errors > Intake accepts debug
files. Secret values go directly from the app to Swiss-Knife, outside chat and
model-visible tool arguments. Never paste them into a conversation.

The assistant can read and propose actions; the app's buttons execute supported
writes after their required reviews. This is not full parity with every native
MCP tool. For an action the app does not offer, use a separately connected Codex
or Claude Code session with the same Swiss-Knife account. Those clients expose
the native write tools, with the same server-side permissions and approval codes.

### Distribution status

The public repository and GitHub release archives are available for manual
installation. They do not establish that Swiss-Knife has been approved or
published in OpenAI's plugin directory. Directory publication is a separate
[review and publish process](https://developers.openai.com/plugins/deploy/submission);
this repository does not confirm its status.

### Any other agent (skills only)

```bash
npx skills add Remneys/swiss-knife-plugin                  # pick the skills when prompted
npx skills add Remneys/swiss-knife-plugin --skill ship-ios # or one skill
```

This installs the skill directories into your agent's own skills folder
(Cursor, Gemini CLI, Codex, Claude Code and others). The skills need the MCP
server to act, so add that too, below.

### MCP server only

No skills, just the tools. The server is a streamable-HTTP MCP endpoint at
`https://swiss.remneys.com/api/mcp` with browser sign-in (OAuth), so any MCP
client can use it:

```bash
codex mcp add swiss-knife --url https://swiss.remneys.com/api/mcp
codex mcp login swiss-knife

claude mcp add --transport http swiss-knife https://swiss.remneys.com/api/mcp
```

## What is in this repository

| Path                                                            | What                                                                       |
| --------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `plugin.json`, `.codex-plugin/plugin.json`                      | Agent Plugins manifest (ChatGPT, Codex). The two files are byte-identical. |
| `mcp.json`                                                      | The MCP server, `https://swiss.remneys.com/api/mcp`, streamable HTTP       |
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` | Claude Code plugin manifest and marketplace                                |
| `.agents/plugins/marketplace.json`                              | Codex marketplace                                                          |
| `skills/`                                                       | The 14 skills, `skills/<name>/SKILL.md` plus `references/`                 |
| `assets/`                                                       | Logo and composer icons                                                    |
| `.github/workflows/release.yml`                                 | Builds the release archives when a version tag is pushed                   |
| `SECURITY.md`, `.github/PULL_REQUEST_TEMPLATE.md`               | How to report a problem; why a pull request here changes nothing           |

Releases carry two archives, both with the files at the archive root:
`swiss-knife-plugin-<version>.zip` (the ChatGPT upload) and
`swiss-knife-plugin-<version>.tgz` (this whole tree).

This repository is generated from the Swiss-Knife monorepo on every change and
holds nothing else: no server code and no credentials. Edits made here directly
are overwritten by the next sync. Report problems at
[swiss.remneys.com/support](https://swiss.remneys.com/support) or open an issue
here.

## License

[MIT](./LICENSE).
