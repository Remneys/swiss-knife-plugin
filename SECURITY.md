# Security

This repository holds the distributable Swiss-Knife plugin and nothing else:
agent skills (Markdown), plugin manifests, the address of the Swiss-Knife MCP
server and icons. It has no server code and no credentials.

## Reporting a vulnerability

Please do not open a public issue for a security problem. Use GitHub's private
vulnerability reporting instead: open this repository's **Security** tab and
choose **Report a vulnerability**. If that is not available to you, say so in a
public issue without any detail and we will arrange another channel.

Things worth reporting here:

- a skill instruction that could make an agent leak a credential, run a
  destructive command without asking, or skip an approval step;
- a manifest or MCP configuration that points somewhere other than the
  Swiss-Knife server;
- a credential, token or private address that was published by mistake.

Problems with the Swiss-Knife service itself (the dashboard, the MCP server, an
SDK) are handled by the same channel.

## About this repository

It is generated from the Swiss-Knife monorepo on every change, and edits made
here directly, including a merged pull request, are overwritten by the next
sync. A fix has to be made at the source, so open an issue and we will make it
there.
