# iOS credentials

Read when a credential is missing, invalid, or the repo is private.

## Contents

- App Store Connect API key
- Git token for a private repo
- Distribution certificate (.p12)
- Replacing credentials

## App Store Connect API key

Needed for managed signing and upload. Read the issuer id, key id and .p8 from a user-provided local file in Codex/Claude Code, or the private credential form in ChatGPT. Do not ask for private-key bytes in chat. The key must belong to the team that owns the app. Creating the key is a human step in App Store Connect; if the user has none, explain that prerequisite and pause only work that needs the key.

```
swiss-knife:build_credentials_save
  app_project_id, platform: "ios", kind: "asc_api_key",
  payload: { issuerId, keyId, p8 }
```

The answer carries `validation` (`valid`, `limited`, `invalid`). Always read it. `swiss-knife:build_credentials_validate` re-checks later. Saving the first one needs nothing more. Replace a stored key only when the user requested replacement; native write connections omit approval. Restricted legacy connections use the handshake below.

## Git token for a private repo

First inspect build_repo_status and retain an existing GitHub App installation that can read the repository. Private repositories work when included in that installation. Use a fine-grained token with read-only Contents for that repository only as a fallback when app access cannot be configured (GitLab: read_repository). Never request a second credential when the existing connection works.

```
swiss-knife:build_credentials_save
  app_project_id, platform: "ios", kind: "git_token", payload: { token }
```

Then `swiss-knife:build_git_token_check` proves it can read the repo. The token belongs to the app.

## Distribution certificate (.p12)

Only when the team is at Apple's certificate cap or the user signs manually. The .p12 must be a distribution certificate for this team that the user already holds.

```
swiss-knife:build_credentials_save
  app_project_id, platform: "ios", kind: "ios_dist_cert",
  payload: { p12Base64, password }
```

Then re-run `swiss-knife:ship_prerequisites`. The team's other apps reuse it.

## Replacing credentials

Read existing metadata first and retain a working credential unless the user requested replacement. An authenticated native connection with the required write scope and role can save the requested replacement directly, omitting `approval`. Read its validation result afterwards. Private bytes come from a user-provided local file or the private upload form, never chat.

Only if the server actually returns an approval refusal on a restricted legacy connection, call `swiss-knife:approval_request` for the same operation and relay its human step. Never invent or reuse a code. A missing role or scope needs the appropriate access, not an approval code.
