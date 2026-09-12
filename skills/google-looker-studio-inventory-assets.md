---
name: google-looker-studio-inventory-assets
description: >-
  Inventory the Looker Studio (Data Studio) reports and data sources an authenticated Google
  Workspace user can see, using the only operation the published contract exposes. Use this to
  audit report sprawl, find assets owned by a departing employee, or locate a report by title
  before linking to it.
api: Google Looker Studio Assets:search API
spec: openapi/google-looker-studio-assets-search-api-openapi.yml
operations:
- searchAssets
generated: '2026-09-12'
method: generated
source: >-
  Grounded in openapi/google-looker-studio-assets-search-api-openapi.yml (operationId searchAssets
  is the only operationId in the spec) and the provider reference
  https://developers.google.com/looker-studio/integrate/api/reference/assets/search. The provider
  publishes no skills/ or AGENTS.md — checked 2026-09-12.
---

# Inventory Looker Studio assets

## Before you start — this will fail for most accounts

The Data Studio API is available **only** to users in a Google Workspace or Cloud Identity
organization, and only after a Workspace admin has authorized your OAuth client through
domain-wide delegation. A consumer `@gmail.com` account cannot call it at any scope. If
authorization returns `Error 400: invalid_scope`, the admin step has not been done — that is not
something you can fix from the client side. Stop and escalate to the Workspace admin with your
OAuth client ID and the scope list below.

Do **not** add Data Studio scopes to your own OAuth client. Google's instructions are explicit
that the admin attaches them during delegation.

## Scopes

- `https://www.googleapis.com/auth/datastudio.readonly` — sufficient for everything in this skill
- `https://www.googleapis.com/auth/datastudio` — only if you also intend to write permissions

Prefer the read-only scope. This skill never writes.

## Steps

1. **Obtain an access token.** Authorization code flow against
   `https://accounts.google.com/o/oauth2/v2/auth`, token exchange at
   `https://oauth2.googleapis.com/token`. Send it as `Authorization: Bearer <token>`.

2. **Call `searchAssets` once per asset type.** `assetTypes` is required and takes exactly one
   value, so a full inventory is two passes, not one.

   ```
   GET https://datastudio.googleapis.com/v1/assets:search?assetTypes=REPORT
   GET https://datastudio.googleapis.com/v1/assets:search?assetTypes=DATA_SOURCE
   ```

3. **Narrow with the search string if you are looking for something specific.** Filters go
   *inside* the `title` parameter, not as separate parameters, and they combine:

   ```
   ?assetTypes=REPORT&title=owner:user@example.com
   ?assetTypes=REPORT&title=Sales creator:me
   ?assetTypes=REPORT&title=parentWorkspace:2a080c66-50cb-4399-92a8-74c534da2de9
   ```

   Supported prefixes: `creator:`, `owner:`, `projectNumber:`, `parentWorkspace:`, `from:`, `to:`.
   The literal `me` resolves to the authenticated user.

4. **Page to exhaustion.** `pageSize` defaults to 1000. Follow `nextPageToken` into `pageToken`
   until the response omits it or returns it empty. An empty `nextPageToken` means done — do not
   treat it as another page.

5. **Sweep the trash separately if the audit needs it.** `includeTrashed=true` returns **only**
   trashed assets; it does not add them to the live set. A complete inventory is four calls:
   two asset types times trashed/not-trashed.

6. **Resolve each hit to a URL** with `https://datastudio.google.com/reporting/{asset.name}` —
   `name` is the opaque ID, `title` is the human label.

## Reading the response

Each asset carries `name`, `title`, `description` (REPORT only), `assetType`, `owner`, `creator`,
`trashed`, `createTime`, `updateTime`, `updateByMeTime`, `lastViewByMeTime`.

Two things to hold onto:

- The spec in this repo calls the type field `type`; the live API returns `assetType`. Read
  `assetType`.
- `owner` and `creator` are **bare** email addresses. The same principal inside a Permissions
  object is prefixed `user:`. Do not pass one where the other is expected.
- Report internals — filters, sections, dimensions — are **not** retrievable. If the task needs
  them, this API cannot answer it.

## Errors

Errors arrive in the `google.rpc.Status` envelope, not RFC 9457 problem+json:
`{"error": {"code", "message", "status", "details": [...]}}`.

| Status | `status` | What to do |
|---|---|---|
| 401 | `UNAUTHENTICATED` (`CREDENTIALS_MISSING`) | No/expired token. Refresh and retry once. |
| 403 | `PERMISSION_DENIED` | The Data Studio API is not enabled on the project, or the caller has no established identity. |
| 400 | `invalid_scope` at authorization time | Domain-wide delegation missing or misconfigured. Escalate. |

There are no `RateLimit-*` or `Retry-After` headers and no published quota, so back off
exponentially with jitter on any 429 or 5xx rather than reading a budget off the wire. There is no
request-id header either — you have nothing to quote in a support ticket beyond the timestamp and
the exact URL.

## Do not

- Do not retry a write after an ambiguous timeout expecting replay protection. This API has no
  idempotency mechanism at all. (This skill is read-only; the warning is for whatever calls it next.)
