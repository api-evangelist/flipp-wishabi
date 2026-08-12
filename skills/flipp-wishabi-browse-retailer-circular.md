---
name: browse-retailer-circular
description: Find a retailer's currently available circular (publication) for a shopper's store or postal/ZIP code on the Flipp FlyerKit API, then walk its pages, highlight regions and categories.
api: Flipp FlyerKit API v4.0
base_url: https://api.flipp.com/flyerkit/v4.0
generated: '2026-08-12'
method: generated
source: openapi/flipp-wishabi-flyerkit-openapi.yml
operations:
  - "GET /publications/{merchant_identifier}"
  - "GET /publication/{publication_id}/pages"
  - "GET /publication/{publication_id}/highlights"
  - "GET /publication/{publication_id}/categories"
---

# Browse a retailer's circular

The FlyerKit API declares no `operationId` on any operation, so operations are referenced here by
method and path exactly as they appear in the published v4.0 Swagger document.

## Before you start

- You need an `access_token`, issued out of band by a Flipp technical contact. It is passed as a
  **query parameter**, not a header. There is no self-service issuance. Because it travels in the
  URL, never put it in a client-side request you do not control — see
  `authentication/flipp-wishabi-authentication.yml`.
- You need a `merchant_identifier`. There is no merchants operation; the identifier comes from
  your Flipp technical contact.
- Every operation is a `GET`. Nothing in this skill mutates anything.

## Step 1 — find the publications available at a location

`GET /publications/{merchant_identifier}`

Required: `merchant_identifier` (path), `access_token`, `locale`.
Supply **either** `store_code` **or** `postal_code` to select the location. `locale` must be one
of `en-CA`, `fr-CA`, `en-US`.

```
GET https://api.flipp.com/flyerkit/v4.0/publications/{merchant_identifier}
      ?access_token={token}&locale=en-CA&postal_code={postal_code}
```

Returns an array of `publication`. Read these fields carefully — they are not interchangeable:

- `valid_from` / `valid_to` — the period the **pricing** is good for.
- `available_from` / `available_to` — the period the publication may be **displayed**.
  Filter on availability when deciding what to show; filter on validity when quoting a price.
- `total_pages`, `deep_link`, and the thumbnail renditions
  (`first_page_thumbnail_150h_url` / `_400h_url` / `_2000h_url`).

Optional: `see_future=true` includes publications not yet live, but that requires an access token
with elevated permission. `show_storefronts` controls whether publications not optimized for
vertical scroll are suppressed.

This operation is **not paginated** — it returns a bare array with no page controls.

## Step 2 — walk the pages

`GET /publication/{publication_id}/pages`

Required: `publication_id` (path), `access_token`.

Returns `publication_page[]` with `page`, three image renditions and a `left`/`top`/`width`/
`height` bounding box. Render the height rendition that matches your viewport; the `2000h` asset
is large.

## Step 3 — overlay the interactive regions

`GET /publication/{publication_id}/highlights` returns named hot-spot rectangles.
`GET /publication/{publication_id}/categories` returns merchandising category regions, each with a
`thumbnail_image_url`.

Both take only `publication_id` and `access_token`, and both return bare unpaginated arrays. Their
geometry is in the same coordinate space as the page bounding boxes from step 2, which is what
lets you draw them over the page image.

## Error handling

FlyerKit declares exactly one error status: **422**, with the envelope
`{"message": string, "code": number}`.

- `422 {"message":"Missing access_token parameter"}` — the token is absent from the query string.
- `422 {"message":"Invalid publication_id or access token"}` — this single message conflates a bad
  id with a bad credential. Verify the token against a known-good publication id first, then the
  id.
- A `422` is also what you get for a missing `locale` or a missing required parameter.

Note the spec declares `code` as a string, but a live probe on 2026-08-12 returned it as an
unquoted number. Parse it loosely. Full catalog: `errors/flipp-wishabi-problem-types.yml`.

**An empty result is HTTP 200 with `[]`, never a 404.** Flipp's own documentation states that when
localized content is unavailable for a locale, "an empty response is returned" — so a missing
translation and a genuinely empty store are indistinguishable. If you get `[]`, retry once with a
different supported locale before concluding there is nothing to show.

There is no `404` on resource operations. A mistyped path returns a 1052-byte HTML page from the
edge proxy, not the JSON error envelope — so check `content-type` before parsing.

## Retries and rate limits

Flipp publishes no rate limits and returns no `RateLimit-*`, `X-RateLimit-*` or `Retry-After`
headers (probed 2026-08-12), and declares no `429`. All operations are `GET` and therefore safe to
retry. Back off conservatively on your own schedule — you have no runtime signal to read.

Responses carry an `x-request-id` UUID. It is undocumented, but capture it: it is the only
correlation handle you can give a Flipp technical contact on a support thread.

The unauthenticated `GET /copyright` endpoint returns `Cache-Control: max-age=3600, public` and a
weak `ETag`; authorized publication endpoints return `Cache-Control: no-cache`.
