# iOS credentials

Read when a credential is missing, invalid, or the repo is private.

## Contents

- App Store Connect API key
- Git token for a private repo
- Distribution certificate (.p12)
- Approval handshake

## App Store Connect API key

Needed for managed signing and upload. Ask the user for the issuer id, key id and the .p8 contents (Apple lets it be downloaded once). The key must belong to the team that owns the app. Creating the key is a human step in App Store Connect; if the user has none, tell them so and stop.

```
swiss-knife:build_credentials_save
  app_project_id, platform: "ios", kind: "asc_api_key",
  payload: { issuerId, keyId, p8 }
```

The answer carries `validation` (`valid`, `limited`, `invalid`). Always read it. `swiss-knife:build_credentials_validate` re-checks later. Saving the first one needs nothing more. Overwriting a stored one loses it, so it needs an approval code: see below.

## Git token for a private repo

The Swiss-Knife GitHub App is private, so a private repository is not readable through it. Use a fine-grained personal access token with read-only Contents on that one repository (GitLab: `read_repository`; classic GitHub: `repo`). Ask the user to create it.

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

## Approval handshake

Replacing a stored credential is refused without a code. Call `swiss-knife:approval_request` with `tool: "build_credentials_save"` and the arguments you mean to use. It says what the human approves, the page to open and the call to make. Relay that verbatim and stop. When the user gives you the code, repeat the same save with `approval` set to it. Do not invent, guess or reuse a code.
