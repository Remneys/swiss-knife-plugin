---
name: sandbox
description: Rehearses a release against a simulated App Store Connect and Google Play with the Swiss-Knife MCP server (create a sandbox app, let time pass, make App Review approve or reject, inject store failures, read the request log, reset). Use when the user says "try the release flow without touching the real store", "simulate App Review", "what happens if Apple rejects it", "test my submission flow", or "make the store fail". Not for shipping to the real stores (use ship-ios / ship-android / submit-release) and never for a real app.
---

# Rehearse a release in the sandbox

**Codex / Claude Code authorization:** a connected `write` scope delegates operations within the user's request and current workspace role. Call the requested tool directly, omitting `approval`; do not ask for a dashboard code or repeat an authorization already given in this conversation. Preview consequential changes and verify their result. Enabling a write switch is allowed through its tool when the user authorized that setup. A read token or insufficient role still cannot write. The approval-code instructions below apply only when this connection actually returns `approval_required` or an approval `human_action` (for example a restricted or older server). ChatGPT keeps its in-chat controls and permissions.

Success: the real tools (`swiss-knife:submission_*`, `swiss-knife:listing_push`, ...) ran against a sandbox app, `swiss-knife:sandbox_state` shows the world where you expect it (version in review, rejected, released), and `swiss-knife:sandbox_requests` shows only simulated traffic. Nothing here touches Apple or Google.

- In ChatGPT use `swiss-knife:dashboard_open` with `section: "builds"`, `tab: "sandbox"` and the existing sandbox app id. A person uses its buttons to advance time, decide a simulated review, add or clear faults and reset from a fresh plan. Manage workspace offers sandbox app creation only in an operator-enabled sandbox workspace. The model can inspect state and requests through `swiss-knife:detail_read` with the corresponding sandbox kind; it must not call hidden write tools. The direct tool steps below are for a connected Codex or Claude Code session with the full MCP surface.

## Rules that never bend

- Sandbox tools work on a sandbox app only. `sandbox_only` (with `details.reason`) means the app is a real one, its workspace is not tagged sandbox, or the server has the sandbox store off. Stop and tell the user; do not try another app of theirs, and never delete a credential to "make it a sandbox".
- Sandbox tools use the same connected permissions as real apps. Native write-scoped connections need no copied codes. The sandbox controls are not a way around a gate: `swiss-knife:sandbox_review_decide` decides only what was submitted.
- If a real prerequisite is returned in `human_action`, relay it and pause only dependent work. An informational response with `approval_required:false` does not block progress.
- Tagging a workspace as sandbox is a super admin's act; you cannot do it. If `swiss-knife:sandbox_app_create` answers `sandbox_only` with `workspace_not_sandbox`, say so and stop.

## Checklist

```
- [ ] 1. Sandbox app: sandbox_app_create (client_key) or an existing one from app_list
- [ ] 2. World read: sandbox_state (versions, builds, tracks, rejection_reasons)
- [ ] 3. The release flow run with the REAL tools (prepare, attach, submit with a code)
- [ ] 4. Time and the reviewer driven: sandbox_advance_time, sandbox_review_decide
- [ ] 5. Failures rehearsed: sandbox_inject_fault (bounded), real tools retried
- [ ] 6. Verified with sandbox_state and sandbox_requests; reset if the user wants a clean world
```

## Steps

1. **App.** `swiss-knife:sandbox_app_create` with a `client_key` (lowercase, digits, underscores). Repeating the call returns the same app. `platform` `ios`, `android` or `both`; `with_catalogue: true` seeds a subscription. Its bundle id is `app.swissknife.sandbox.<client_key>`. For an existing sandbox app, find it with `swiss-knife:app_list`.
2. **Look.** `swiss-knife:sandbox_state` (add `platform` when the app has both). A look changes nothing.
3. **Run the flow with the real tools.** `swiss-knife:submission_prepare`, `swiss-knife:submission_attach_build`, then `swiss-knife:submission_submit` with `dry_run: false` when authorized; native connections omit `approval`. The simulated store answers as the real one would.
4. **Let the store act.** `swiss-knife:sandbox_advance_time` with `by` like `31m` or `2d` (give a `client_key` so a retry advances once): builds finish processing, a waiting review enters review, Google's wait ends. Then `swiss-knife:sandbox_review_decide`: `approve`, or `reject` with a `reason` from `rejection_reasons`, and read the failure the real tools then show; references/rejections.md lists what to try.
5. **Break it on purpose.** `swiss-knife:sandbox_inject_fault` with `action: "add"`: a `route` (an App Store Connect operation id) or `path`, `times` (required, at most 20), and one effect: a catalogue `reason`, a bare `status`, or a transport `reset`. Faults expire (`ttl`, default one hour of the world's time). `action: "list"` shows `fired` and `remaining`; `action: "clear"` removes them.
6. **Verify.** `swiss-knife:sandbox_requests` (add `failed_only`) shows each request and the status the world answered, injected faults included. `swiss-knife:sandbox_state` shows the result.
7. **Start over.** `swiss-knife:sandbox_reset` with the default `dry_run` shows what would be dropped and a fingerprint; run it again with `dry_run: false` and that `plan_fingerprint`. It returns the world to its seed and clears faults and the log.

## When it breaks

- `sandbox_only` -> not a sandbox app (rule above).
- `conflict` on `sandbox_app_create` -> that `client_key` made an app of another platform; use a new key.
- `precondition_failed` on `sandbox_review_decide` -> nothing is waiting for review: submit first (step 3), or the review was already decided.
- `invalid_request` on `sandbox_advance_time` -> `by` is a duration like `90s`, `10m`, `1h30m`, `2d`, between 1 second and 30 days.
- `precondition_failed` on `sandbox_advance_time` -> the world is already a year ahead; reset it.
- `precondition_failed` "at most 10 faults" -> clear some with `action: "clear"`.
- `mirror.refreshed: false` in an answer -> the change was made; the refresh met a fault or an error. Run `swiss-knife:store_sync_now` once the cause is gone.
