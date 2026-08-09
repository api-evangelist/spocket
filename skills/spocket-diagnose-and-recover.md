---
name: Diagnose a failing Spocket app and get it running again
description: >-
  Work a stopped or crash-looping app back to running — read the logs, identify the cause,
  fix and redeploy, or roll back to the version that worked.
api: mcp/spocket-mcp.yml
surface: https://www.spocket.dev/api/mcp
operations: [spocket_status, spocket_logs, spocket_search_logs, spocket_deploy, spocket_rollback, spocket_restart, spocket_migrate, spocket_nodes]
generated: '2026-08-09'
method: generated
source: https://www.spocket.dev/documentation/logs-and-status
---

# Diagnose a failing Spocket app and get it running again

## Read the state first

`spocket_status` returns the live container state, so an app that died a second ago says
so rather than showing stale good news:

| State | Means |
|---|---|
| `running` | The process is up. |
| `starting` | Fetching, installing, or booting. |
| `stopped` | Not running. Read the logs for why. |
| `crashed` | Failed five times in thirty minutes; automatic restarts have been stopped. |
| `suspended (billing)` | Paused for payment. Nothing has been deleted. |

`crashed` is deliberate: an app failing on startup fails identically every time, so
Spocket stops retrying and makes the problem visible instead of hiding it behind a
restart loop. Restarting it without changing anything will not help.

## Find the cause

- `spocket_logs` returns up to 500 lines of recent output. Logs are a rolling window sized
  by plan (8 MB trial / 32 MB Solo / 128 MB Builder / 512 MB Fleet), not an archive — if
  the failure is older than the window it is gone.
- `spocket_search_logs` when the log is long. Search returns matching lines with position
  and surrounding context; that is usually enough to see what led to the failure.

The four causes the provider names, in order of frequency:

1. A missing environment variable — the thing `.env` had that the deploy did not.
2. A package the code imports but `package.json` / `requirements.txt` never declared.
3. An invalid or reset token. Discord shows a token once; resetting invalidates the old one.
4. An entry point that does not match the file that is actually there.

If the app serves HTTP and returns nothing, check it binds `0.0.0.0` rather than
`localhost` — that is the single most common cause of a dead webhook URL.

## Recover

- **Fix and reship.** Change the code and `spocket_deploy` under the same name. That ships
  a new version; the previous version is kept. Environment values survive a redeploy.
- **Roll back.** `spocket_rollback` re-serves an earlier bundle and restarts on it.
  Rolling back does **not** delete the newer version — you can go forward again after
  fixing it. Prefer this when the app was working an hour ago.
- **Restart.** `spocket_restart` restarts, or starts it if it is already down. Only
  useful when the cause was transient — a `crashed` app needs a code or config change.
- **Move it.** `spocket_migrate` rebuilds the app on a different machine, removing the old
  container only once the new one exists, so a failed move leaves it where it was. Use
  this only when the machine is misbehaving — `spocket_nodes` shows which machines your
  apps are on. You cannot choose the destination. **Migration is never the fix for a
  configuration problem.**

## Report honestly

Say which of the states above the app is in, quote the log lines that show the cause, and
name the fix. Do not report success from a deploy alone — re-check `spocket_status` and
only call it done on `running`.
