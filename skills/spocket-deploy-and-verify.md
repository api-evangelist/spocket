---
name: Deploy an app to Spocket and verify it is healthy
description: >-
  Ship a folder to Spocket over MCP, set the secrets it needs, and confirm the container
  is actually running before telling the user it worked.
api: mcp/spocket-mcp.yml
surface: https://www.spocket.dev/api/mcp
operations: [spocket_deploy, spocket_set_env, spocket_status, spocket_logs, spocket_urls]
generated: '2026-08-09'
method: generated
source: https://www.spocket.dev/documentation/quickstart
---

# Deploy an app to Spocket and verify it is healthy

Grounded in the twenty MCP tools Spocket publishes at
<https://www.spocket.dev/documentation/mcp-tools>. Every tool named below appears in that
reference verbatim.

## Before you start

- The server is `https://www.spocket.dev/api/mcp` over Streamable HTTP. Always use the
  `www` host — the bare domain redirects and some clients do not follow redirects on
  discovery.
- There is **no API key**. Authentication is OAuth 2.1 with PKCE and RFC 7591 dynamic
  client registration; the first tool call opens a browser for human consent. If a call
  returns `401 {"error":"unauthorized"}`, the connection has not been approved or the
  token was revoked — ask the user to approve it, do not retry in a loop.
- App names are lowercase letters, numbers and dashes (`mod-app`, not `Mod App`).

## Steps

1. **Decide what you are shipping.** Send source files only. Leave out `node_modules`,
   `.git`, `.env` and regenerable build output. If the deploy is unexpectedly large, say
   what it included rather than shipping it silently.
2. **Pick the `kind`.** One of `sites`, `discord`, `slack`, `telegram`, `twitch`,
   `worker`, `scraper`, `cron`, or `bot` for anything not listed. It selects an icon and
   suggests a token name; it gates nothing.
3. **`spocket_deploy`** the folder under a name. Deploying to a name that already exists
   ships a **new version** of that app rather than creating a second one. Runtime and
   entry point are inferred — a `package.json` means Node 22 and `index.js`, a
   `requirements.txt` means Python 3.11 and `main.py`. State them explicitly if the
   project is laid out differently. For a static site pass `runtime: 'static'`; an
   `index.html` at the top level or in `dist/`, `build/`, `out/`, `public/` or `_site/`
   is found automatically.
4. **`spocket_set_env`** for every secret the app needs. Values are encrypted and
   **cannot be read back** — only replaced. Setting one restarts the app. Do not ask the
   user to paste a production token into chat; tell them to set it in the panel's
   Environment tab if it matters.
   - If the call returns `not_injected`, the app is on a machine built before injection
     was supported. **Deploy again** to rebuild it. A restart will not pick the values
     up, and `spocket_migrate` is the wrong remedy — migration is for node failure.
5. **`spocket_status`** the app. Status reads the live container, not a stored guess.
   `running` is the only success state; `starting` means keep waiting; `stopped` or
   `crashed` means go to step 6.
6. **`spocket_logs`** on anything that is not `running`. The cause is almost always in the
   first few lines. Lines prefixed `[spocket]` are the platform's; the rest is the app's.
7. **`spocket_urls`** if the app serves HTTP. It returns the free
   `<name>-<suffix>.spocket.dev` hostname and the path-based
   `https://www.spocket.dev/hooks/<slug>` webhook URL. **Never construct the hostname** —
   the suffix is random. The app must bind `0.0.0.0` on `process.env.PORT`; a server bound
   to `localhost` inside the container is unreachable.

## Rules

- Never call `spocket_delete` on an assumption. It is the one destructive tool and it
  confirms by name first; deletion is a human decision.
- You cannot read an environment value back, see how many machines exist or how full they
  are, or choose which machine an app lands on. Do not plan around any of those.
- Nothing over 1 GB of memory, nothing needing a GPU, and no persistent local storage —
  disk is per-container and does not survive a rebuild.
