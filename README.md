# Swiss-Knife plugin

Ship and monetize mobile apps from chat. This plugin connects ChatGPT, Codex
and Claude Code to [Swiss-Knife](https://swiss.remneys.com): iOS and Android
builds, TestFlight and Google Play releases, store listings, subscription
products, push, and crash and revenue analytics, through the Swiss-Knife MCP
server (`https://swiss.remneys.com/api/mcp`).

It bundles two things:

- **11 agent skills** that encode the order of operations and the stores' real
  pitfalls, so the model follows a tested path instead of guessing.
- **The MCP server connection**, with account sign-in (OAuth). You need a
  Swiss-Knife account; the plugin only sees the workspaces and apps that account
  can open.

| Skill            | Use it to                                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------------------------- |
| `swiss-knife`    | Connect the MCP server, start with `help`, pick a skill, relay human steps and approvals.                 |
| `setup-sdk`      | Provision a workspace and app and wire the mobile SDK (Expo / React Native).                              |
| `setup-native`   | The native iOS / Android half of the setup wizard (the SDK sources vendored from the public npm package). |
| `ship-ios`       | Build an iOS app and put it in TestFlight, checked against App Store Connect.                             |
| `ship-android`   | Build an Android app and put it on the Play internal track, checked against Google Play.                  |
| `store-listings` | Edit listing text, translations, screenshots and age rating on both stores, checked against the store.    |
| `submit-release` | Submit for review, phased rollout and promotion, with human approval codes.                               |
| `monetization`   | Products, entitlements, offerings and the store catalogue push.                                           |
| `push`           | Push credentials (APNs, FCM), sending with a dry run, and device hygiene.                                 |
| `observability`  | Events, funnels, subscription and crash stats, error issues and alerts.                                   |
| `sandbox`        | Rehearse a release against a simulated App Store Connect and Google Play: review, time and faults.        |

Actions that change a store or spend money (submitting to App Review,
promoting to production, replacing a signing credential) stop and wait for an
approval code that you issue yourself. The code is tied to that one action and
expires after 15 minutes.

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

Download `swiss-knife-plugin-<version>.zip` from the
[latest release](https://github.com/Remneys/swiss-knife-plugin/releases/latest)
and upload it with **Upload plugin archive** in ChatGPT's plugin settings. The
archive has `plugin.json` at its root and is well under the 100 MB limit.
Sign in with your Swiss-Knife account when asked. ChatGPT's plugin screens
change often: if the label differs, look for the option to add a plugin from a
ZIP archive.

In ChatGPT the plugin reads and explains (build and release status, what blocks
a release, listings, products, revenue and crash reports) and shows in-chat
cards. Starting builds and editing listings or products is done with the same
account from Codex or Claude Code, or in the dashboard.

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
| `skills/`                                                       | The 11 skills, `skills/<name>/SKILL.md` plus `references/`                 |
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
