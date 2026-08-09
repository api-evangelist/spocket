---
name: Provision hosting for your own customers over the Spocket Platform API
description: >-
  Server-to-server provisioning with the Platform REST API — exchange client credentials
  for a token, create an app under a sub-account with an idempotency key, and surface the
  plain-language diagnosis when a customer's app breaks.
api: https://www.spocket.dev/api/v1
surface: https://www.spocket.dev/documentation/platform-api
operations:
  - 'POST /api/v1/token'
  - 'POST /api/v1/apps'
  - 'GET /api/v1/apps'
  - 'GET /api/v1/apps/:id'
  - 'POST /api/v1/apps/:id/power'
  - 'GET /api/v1/apps/:id/logs'
  - 'DELETE /api/v1/apps/:id'
  - 'GET /api/v1/sub-accounts'
  - 'POST /api/v1/sub-accounts'
  - 'GET /api/v1/account'
generated: '2026-08-09'
method: generated
source: https://www.spocket.dev/documentation/platform-api
---

# Provision hosting for your own customers over the Spocket Platform API

This is the **server-side** surface, separate from the MCP server. MCP is a person
connecting their editor and clicking Allow; this is a backend with no human in front of
it. Spocket publishes **no OpenAPI** for it — every endpoint below comes from the
provider's published endpoint table.

## Preconditions

- A **paid Fleet plan**. The key page unlocks only on Fleet.
- A key created at <https://www.spocket.dev/platform>, with scopes chosen at creation:
  `apps:read`, `apps:write`, `apps:delete`, `domains:write`. Give a provisioning
  integration `apps:write` and leave `apps:delete` off — a cancelling customer should be
  suspended, not destroyed.
- Two values in the backend environment: `SPOCKET_CLIENT_ID` (`spk_live_…`) and
  `SPOCKET_CLIENT_SECRET` (`sk_…`). The secret is shown once and stored hashed.

## Steps

1. **Get a token.** `POST /api/v1/token` with
   `{"grant_type":"client_credentials","client_id":…,"client_secret":…}`. Tokens last an
   hour; mint another when needed. Send it as `authorization: Bearer <token>`. Never send
   the secret on ordinary requests — a token that leaks into a log expires on its own; a
   leaked secret means rotating across the whole fleet.
2. **Optionally create the customer.** `POST /api/v1/sub-accounts` ahead of provisioning,
   or just pass `sub_account_ref` on the app and the grouping happens implicitly.
3. **Provision.** `POST /api/v1/apps` with `{name, kind, sub_account_ref, files[]}`, where
   each file is `{path, content}` and `content` is **base64**. One call creates the app and
   deploys its first version.
   - **Always send an `Idempotency-Key` header** with your own request id. If the call
     times out and you retry, you get the app that was already created rather than a
     second one you would also be billed for. This is the single most important
     convention on this API.
4. **Poll or read on demand.** `GET /api/v1/apps` lists, optionally filtered by
   `sub_account_ref`. `GET /api/v1/apps/:id` returns status **and a plain-language
   diagnosis** when the platform can tell what went wrong — a missing package named, a
   rejected token, memory exhausted. Show that string to your customer; it usually ends
   the ticket before it reaches you.
5. **Control.** `POST /api/v1/apps/:id/power` takes `start`, `stop` or `restart`.
   `GET /api/v1/apps/:id/logs` returns recent output.
6. **Wind down.** `DELETE /api/v1/apps/:id` stops the app and stops billing immediately,
   but destroys nothing for **seven days** — a loop that deleted the wrong thing can be
   undone.
7. **Watch capacity.** `GET /api/v1/account` returns capacity, usage and app health.
   Fleet covers twenty apps; each app beyond that is $2.45/month at the same rate whether
   you run twenty or a thousand. Provisioning and deleting adjust the count immediately
   and prorate the difference.

## Rotation

Rotating a key keeps the old secret working for **24 hours**, so the new one can be rolled
out without taking customers offline. Plan rotations inside that window.

## Known gaps in the published contract

Say these plainly rather than guessing:

- No OpenAPI, Postman collection or SDK is published. There is nothing to generate a
  client from.
- No error reference. Status codes and error bodies for these endpoints are not
  documented; errors observed on the platform use a bare `{"error":"<slug>"}` envelope,
  not RFC 9457 `application/problem+json`.
- No pagination parameters are documented on `GET /api/v1/apps`.
- No rate limits, quotas or 429 semantics are published.
- `domains:write` is a published scope with **no** endpoint in the published table —
  custom domains are reachable only through the MCP tools today.
