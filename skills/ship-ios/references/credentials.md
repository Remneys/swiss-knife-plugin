# iOS credentials

Read when a credential is missing, invalid, or the repo is private.

## Contents

- App Store Connect API key
- Git token for a private repo
- Distribution certificate (.p12)
- Replacing credentials

## App Store Connect API key

Needed for managed signing and upload. First inspect stored metadata and
validate an existing key; members reuse the app's configured connection.
For a missing or unusable key, prefer the private Apple Account form in
[apple-owner.md](apple-owner.md). The owner signs in and confirms the intended
team and key action there. Creating a key is handled by that connection;
manual portal export is not a universal prerequisite.

An owner who already has a .p8 can instead use the private credential upload
form. Private-key bytes, passwords and Apple verification codes never belong
in chat or model tool arguments. The key must belong to the app's Apple team.
Do not replace a stored key unless the user requested replacement.

Verify with `build_credentials_list` and `build_credentials_validate`.
Read `validation`: `valid`, `limited`, or `invalid`; a saved row alone does
not establish working Apple access.

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
