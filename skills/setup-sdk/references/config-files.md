## Step 4 — Write the data-plane config files

Write these into the project root. **Templates are exact — insert verbatim**,
substituting the captured values. Prefer Write for new files; if a file exists,
read it and merge rather than clobber.

**If the Step 1.4 ledger was non-empty, reconcile instead of rewriting.** Before
touching anything, tell the user in a few lines what you found and what you will
change — e.g. _"`.mcp.json` already has a swiss-knife server (updating the token),
`src/swiss-knife.ts` exists and looks correct (leaving it), `.env` has placeholder
values (filling them in)."_ Then apply only the deltas. Never clobber a populated
`.env` or an edited `App.tsx` wholesale — read, then surgically Edit. See
_Reconciling an existing setup_ for the full rules.

**Order matters — ensure `.gitignore` FIRST, then write the secrets.** Step (c)
below writes real `sk_live_…` / `sk_mcp_…` values into `.env`; if `.env` isn't
already ignored, a stray `git add -A` in that window would commit them. So do (a)
before writing any secret file.

**(a) `.gitignore`** — ensure these lines exist (append if missing; don't
duplicate). Keep `.mcp.json`, `.env.example`, and `.envrc` **committed** — only
`.env` carries secrets:

```gitignore
.env
.env.local
```

**(b) `.mcp.json`** — paste the `mcpJson` you got from `app_token_mint`. It
references an env var (not the literal token), so it's safe to commit:

```jsonc
{
  "mcpServers": {
    "swiss-knife": {
      "type": "http",
      "url": "<ORIGIN>/api/mcp",
      "headers": { "Authorization": "Bearer ${SWISS_KNIFE_MCP_TOKEN}" },
    },
  },
}
```

**(c) `.env`** (gitignored via (a) — holds the real values):

```bash
# Swiss-Knife — DO NOT COMMIT (gitignored)
# Mobile SDK auth. Expo inlines EXPO_PUBLIC_* into the app bundle at build time.
# NOTE: EXPO_PUBLIC_SWISSKNIFE_API_KEY (sk_live_…) is a CLIENT-embeddable key —
# it is DESIGNED to ship inside the app bundle and is NOT a server secret despite
# the "live" prefix. (It's still kept out of git via .env as good hygiene.)
EXPO_PUBLIC_SWISSKNIFE_API_KEY=<sdkKey from app_create>
# Deployment ORIGIN. Load-bearing, not cosmetic: EVERY SDK entry point in Step 5
# is handed this value, and crash reporting / performance tracing fall back to
# Swiss-Knife's HOSTED host if it goes missing. See the note in Step 5.2.
EXPO_PUBLIC_SWISSKNIFE_API_URL=http://localhost:3000
# Claude Code MCP token for this project's .mcp.json — a real secret, keep private:
SWISS_KNIFE_MCP_TOKEN=<token from app_token_mint>
```

**(d) `.env.example`** (committed — placeholders only, never real secrets):

```bash
EXPO_PUBLIC_SWISSKNIFE_API_KEY=YOUR_SDK_KEY
EXPO_PUBLIC_SWISSKNIFE_API_URL=http://localhost:3000
SWISS_KNIFE_MCP_TOKEN=YOUR_MCP_TOKEN
```

**(e) `.envrc`** (committed) — Claude Code does **not** auto-load `.env`, so it
can't see `SWISS_KNIFE_MCP_TOKEN` for the `${…}` in `.mcp.json` on its own.
`direnv` loads `.env` into the shell so both Claude Code and Expo see everything:

```bash
dotenv
```

`direnv allow` is a trust action, so **ask the user to run `direnv allow`**
themselves. **No direnv?** The safe fallback is to `export SWISS_KNIFE_MCP_TOKEN=…`
in the shell before launching Claude Code (or `source .env`) — this keeps
`.mcp.json` committed and secret-free. The `.mcp.json` token only resolves once
that env var is present in Claude Code's shell.

> **Don't inline the literal token** into `.mcp.json`'s `Authorization` header.
> `.mcp.json` is conventionally a **tracked** file, and adding it to `.gitignore`
> does NOT untrack an already-tracked file — so a literal `sk_mcp_…` there can
> still be committed. Not recommended. If you truly cannot use an env var: first
> `git rm --cached .mcp.json`, confirm `git check-ignore .mcp.json` reports it as
> ignored, and only then inline the token.
