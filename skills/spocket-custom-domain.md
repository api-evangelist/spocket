---
name: Put a Spocket app on a custom domain
description: >-
  Attach a user's own domain to an app, relay the exact DNS records, verify, and
  troubleshoot the four documented verification failures.
api: mcp/spocket-mcp.yml
surface: https://www.spocket.dev/api/mcp
operations: [spocket_urls, spocket_add_domain, spocket_verify_domain, spocket_remove_domain, spocket_status]
generated: '2026-08-09'
method: generated
source: https://www.spocket.dev/documentation/domains-and-webhooks
---

# Put a Spocket app on a custom domain

## Preconditions

The app must serve HTTP — it has to bind `process.env.PORT` on `0.0.0.0`. A gateway-only
bot (Discord gateway, Slack Socket Mode, Telegram polling) has nothing to serve on a
domain. Check with `spocket_urls`: if there is no free `*.spocket.dev` address, the app is
not listening.

Custom-domain allowance is per plan: Solo 1, Builder 5, Fleet unlimited.

## Steps

1. **`spocket_add_domain`** with the app and the hostname. It returns the **exact** DNS
   records to create — one routing record and one TXT record for proof of ownership.
2. **Relay the records verbatim.** You cannot create them; the user has to add them at
   whoever hosts their DNS (registrar, Cloudflare, wherever the domain points today). Do
   not paraphrase the values.
3. **`spocket_verify_domain`.** A failure immediately after adding records is normal —
   DNS takes a few minutes to propagate. Wait and try again rather than telling the user
   something is wrong.
4. **Confirm.** Once verified, paths reach the app unchanged: `/about` on their domain
   hits `/about` on the app, and visitors never see a `spocket.dev` URL. Certificates are
   issued and renewed automatically.

Nothing is served on the domain until it verifies. Pointing a CNAME alone is not enough —
verification is a TXT record only someone with zone access can create.

## When verification fails

| What you see | What it usually means |
|---|---|
| No TXT record found | Not propagated yet, or created under the wrong name — it belongs at `_spocket`, not the root. |
| Found a TXT record, wrong value | Two records exist, or the old one was edited rather than replaced. Delete any stale `_spocket` record. |
| Verified, but the site does not load | The routing record is missing or still points elsewhere. Verification and routing are separate records; both have to be right. |
| This domain is not connected | The domain resolves to Spocket but is not verified, or it was removed. Re-add and verify. |

DNS is usually live within minutes, but a provider can cache an old answer for its full
TTL. If a record is definitely correct and still failing, the wait is on their side — say
that rather than re-adding the domain.

## Removing

`spocket_remove_domain` detaches the domain and releases its certificate. The free
`<name>-<suffix>.spocket.dev` address and the `https://www.spocket.dev/hooks/<slug>`
webhook URL keep working either way — both survive restarts, redeploys and machine moves,
which is why they are safe to paste into Stripe or GitHub.
