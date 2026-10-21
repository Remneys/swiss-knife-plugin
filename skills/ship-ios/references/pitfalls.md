# iOS pitfalls

Read when a prerequisite, preflight, build or upload step fails.

## Contents

- Failure table
- ASC_CERT_LIMIT in detail
- Build numbers
- Reading human_action

## Failure table

Format: error code or situation -> what to do.

| Situation                                                                                                                                                         | What to do                                                                                                                                                                                                                                                                                                   |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ASC_CERT_LIMIT` (also `signing_certificate` failed in `swiss-knife:ship_prerequisites`, or `managed_signing_cert_cap` blocking in `swiss-knife:build_preflight`) | Stop. Never auto-revoke. See below.                                                                                                                                                                                                                                                                          |
| `human_action` in any answer                                                                                                                                      | Relay the required step, pause dependent work, then use `verify_with` after completion.                                                                                                                                                                                                                      |
| Refusal that needs an approval code only on a restricted legacy connection                                                                                        | `swiss-knife:approval_request`, relay, wait for the human's code, repeat the same call with `approval`.                                                                                                                                                                                                      |
| App record does not exist in App Store Connect                                                                                                                    | The `human_action` says to create it (New App, exact bundle id). If the bundle id is unregistered, `swiss-knife:signing_action` verb `register_bundle_id` needs an approval code only on a restricted legacy connection. Then verify with `swiss-knife:store_sync_now` and `swiss-knife:ship_prerequisites`. |
| Agreement missing or expired (`REQUIRED_AGREEMENTS_MISSING_OR_EXPIRED`)                                                                                           | Relay the `human_action`; only the Account Holder can sign.                                                                                                                                                                                                                                                  |
| `asc_api_key` validation `invalid`                                                                                                                                | Ask for a correct key and save again (native write connections omit approval when replacement was requested). Do not build.                                                                                                                                                                                  |
| No `swiss-build.config.*` at the ref, or no `ios` section                                                                                                         | Add it to the repo as a plain object. Never import the package or tell anyone to `npm install` it.                                                                                                                                                                                                           |
| Private repo, clone or ref check fails                                                                                                                            | Save a `git_token`; see references/credentials.md.                                                                                                                                                                                                                                                           |
| No build machine online                                                                                                                                           | Warning only: the build queues and waits (the queue times out after 6 hours). Tell the user.                                                                                                                                                                                                                 |
| `asc_build_upload_commit_invalid` (fatal)                                                                                                                         | Not retryable; report it with the `failureReason`.                                                                                                                                                                                                                                                           |
| `build_exists_in_build_channel`                                                                                                                                   | Success: the build is already attached. Run the verify step.                                                                                                                                                                                                                                                 |

## ASC_CERT_LIMIT in detail

Apple limits distribution certificates per team. When none that this workspace holds the key for can be reused and no slot is free, the build refuses with `ASC_CERT_LIMIT` and lists the team's certificates. Do not select an arbitrary certificate to revoke: other apps or CI may still depend on its key. Inspect current usage before carrying out any explicit revocation request.

1. Call `swiss-knife:signing_team_certs` and show the user the list: which certificates this workspace holds a key for and which do not.
2. Offer exactly two options and let them choose:
   - Upload a .p12 of a distribution certificate they already hold for this team (`swiss-knife:build_credentials_save`, kind `ios_dist_cert`; recipe in references/credentials.md).
   - Revoke a certificate nobody uses, themselves, in the Apple Developer portal (Certificates, Identifiers & Profiles).
3. Do not revoke an arbitrary certificate to make room. If the owner explicitly selects an unused certificate for revocation, inspect its usage and use `signing_action` with `revoke_certificate`; honor the server's in-use guard. Otherwise pause that signing step while the owner chooses an existing certificate or a specific revocation.
4. After they confirm, re-run `swiss-knife:ship_prerequisites`, then `swiss-knife:build_preflight`, then `swiss-knife:build_retry` or `swiss-knife:build_start`.

### A certificate another workspace of the same owner holds

Read `signing_team_cert_candidates` to find reusable certificates in another workspace the user owns on this Apple team. Offer a matching candidate instead of assuming a new certificate or portal visit is needed. Two workspaces of ONE owner may share one certificate; a link references the other workspace's key (it is never copied) and avoids consuming another certificate slot.

1. `swiss-knife:signing_team_cert_candidates` with the `app_project_id` of the app that cannot sign. It lists `source_workspace_id`, `cert_serial` and `expires_at` of certificates the user's other workspaces hold on this app's Apple team. Empty: say so and go back to the two choices above.
2. `swiss-knife:signing_team_cert_link` with `app_project_id`, `source_workspace_id` and `cert_serial`; `dry_run` defaults to true: it runs the live proof and writes nothing. Show the user `approval.summary`.
3. The caller must own BOTH workspaces, and sharing this certificate must match the user's intent. If authorized, repeat with `dry_run: false` using native write access and omit `approval`. Do not ask again when the user already selected this link. Only an actual restricted-connection approval refusal uses `approval_request`; a role failure needs the proper owner.
4. Check `swiss-knife:signing_team_cert_links_list` (`status: "active"`), then continue at step 4 above. `swiss-knife:signing_team_cert_unlink` with the `link_id` undoes it: no code, say so when the user asks to stop sharing.

## Build numbers

The control plane assigns the build number; you never set it. A duplicate (Apple already has that number for the version) means: `swiss-knife:build_retry` for a failed build (it gets a fresh number), or a new `swiss-knife:build_start`. For a binary built outside Swiss-Knife, `swiss-knife:artifact_register` refuses a build number already on record: raise it in the project and rebuild.

## Reading human_action

Shape: `{ who, what, url, why, verify_with: { tool, args?, note } }`. Show who, what, url and why unchanged. Do not drop the url. After the user confirms, call `verify_with.tool` once; if the same prerequisite remains, report it without repeating the blocked call. Continue independent authorized work.
