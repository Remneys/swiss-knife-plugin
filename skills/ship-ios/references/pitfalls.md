**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

# iOS pitfalls

Read when a prerequisite, preflight, build or upload step fails.

## Contents

- Failure table
- ASC_CERT_LIMIT in detail
- Build numbers
- Reading human_action

## Failure table

Format: error code or situation -> what to do.

| Situation                                                                                                                                                         | What to do                                                                                                                                                                                                                                                            |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ASC_CERT_LIMIT` (also `signing_certificate` failed in `swiss-knife:ship_prerequisites`, or `managed_signing_cert_cap` blocking in `swiss-knife:build_preflight`) | Stop. Never auto-revoke. See below.                                                                                                                                                                                                                                   |
| `human_action` in any answer                                                                                                                                      | Relay verbatim, STOP, wait, then call its `verify_with` tool.                                                                                                                                                                                                         |
| Refusal that needs an approval code                                                                                                                               | `swiss-knife:approval_request`, relay, wait for the human's code, repeat the same call with `approval`.                                                                                                                                                               |
| App record does not exist in App Store Connect                                                                                                                    | The `human_action` says to create it (New App, exact bundle id). If the bundle id is unregistered, `swiss-knife:signing_action` verb `register_bundle_id` needs an approval code. Then verify with `swiss-knife:store_sync_now` and `swiss-knife:ship_prerequisites`. |
| Agreement missing or expired (`REQUIRED_AGREEMENTS_MISSING_OR_EXPIRED`)                                                                                           | Relay the `human_action`; only the Account Holder can sign.                                                                                                                                                                                                           |
| `asc_api_key` validation `invalid`                                                                                                                                | Ask for a correct key and save again (approval code to replace). Do not build.                                                                                                                                                                                        |
| No `swiss-build.config.*` at the ref, or no `ios` section                                                                                                         | Add it to the repo as a plain object. Never import the package or tell anyone to `npm install` it.                                                                                                                                                                    |
| Private repo, clone or ref check fails                                                                                                                            | Save a `git_token`; see references/credentials.md.                                                                                                                                                                                                                    |
| No build machine online                                                                                                                                           | Warning only: the build queues and waits (the queue times out after 6 hours). Tell the user.                                                                                                                                                                          |
| `asc_build_upload_commit_invalid` (fatal)                                                                                                                         | Not retryable; report it with the `failureReason`.                                                                                                                                                                                                                    |
| `build_exists_in_build_channel`                                                                                                                                   | Success: the build is already attached. Run the verify step.                                                                                                                                                                                                          |

## ASC_CERT_LIMIT in detail

Apple allows 3 distribution certificates per team. When none that this workspace holds the key for can be reused and no slot is free, the build refuses with `ASC_CERT_LIMIT` and lists the team's certificates. swiss-knife never revokes one on its own, and neither do you: revoking stops every build signed with that certificate, including other apps and other CI.

1. Call `swiss-knife:signing_team_certs` and show the user the list: which certificates this workspace holds a key for and which do not.
2. Offer exactly two options and let them choose:
   - Upload a .p12 of a distribution certificate they already hold for this team (`swiss-knife:build_credentials_save`, kind `ios_dist_cert`; recipe in references/credentials.md).
   - Revoke a certificate nobody uses, themselves, in the Apple Developer portal (Certificates, Identifiers & Profiles).
3. STOP and wait. Do not call `swiss-knife:signing_action` with `revoke_certificate`, even if the user hints at it; revoking is the human's decision in the portal.
4. After they confirm, re-run `swiss-knife:ship_prerequisites`, then `swiss-knife:build_preflight`, then `swiss-knife:build_retry` or `swiss-knife:build_start`.

### A certificate another workspace of the same owner holds

Only when the user says they own another workspace on this Apple team that already signs with a distribution certificate: offer it as a third choice, and never start with it. Two workspaces of ONE owner may share one certificate; a link references the other workspace's key (it is never copied) and saves a slot of Apple's 3.

1. `swiss-knife:signing_team_cert_candidates` with the `app_project_id` of the app that cannot sign. It lists `source_workspace_id`, `cert_serial` and `expires_at` of certificates the user's other workspaces hold on this app's Apple team. Empty: say so and go back to the two choices above.
2. `swiss-knife:signing_team_cert_link` with `app_project_id`, `source_workspace_id` and `cert_serial`; `dry_run` defaults to true: it runs the live proof and writes nothing. Show the user `approval.summary`.
3. Only the person who owns BOTH workspaces can approve. Call `swiss-knife:approval_request` with tool `signing_team_cert_link` and the same arguments, relay what it says, wait for the code, then repeat the link call with `dry_run: false` and `approval`. Never link without a code the user gave you; `forbidden` with a `human_action` means relay it and stop.
4. Check `swiss-knife:signing_team_cert_links_list` (`status: "active"`), then continue at step 4 above. `swiss-knife:signing_team_cert_unlink` with the `link_id` undoes it: no code, say so when the user asks to stop sharing.

## Build numbers

The control plane assigns the build number; you never set it. A duplicate (Apple already has that number for the version) means: `swiss-knife:build_retry` for a failed build (it gets a fresh number), or a new `swiss-knife:build_start`. For a binary built outside Swiss-Knife, `swiss-knife:artifact_register` refuses a build number already on record: raise it in the project and rebuild.

## Reading human_action

Shape: `{ who, what, url, why, verify_with: { tool, args?, note } }`. Show who, what, url and why unchanged. Do not drop the url. After the user confirms, call `verify_with.tool` once; if it still shows the block, relay it again and stop.
