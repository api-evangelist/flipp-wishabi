---
name: locate-store-and-inventory
description: Resolve a shopper to the nearest Flipp-enabled store for a retailer, use that store to scope circular content, and read live store-level inventory and add-to-cart affordances for an offer.
api: Flipp FlyerKit API v4.0
base_url: https://api.flipp.com/flyerkit/v4.0
generated: '2026-08-12'
method: generated
source: openapi/flipp-wishabi-flyerkit-openapi.yml
operations:
  - "GET /fsa/{merchant_name_identifier}"
  - "GET /stores/{merchant_identifier}"
  - "GET /publications/{merchant_identifier}"
  - "GET /product/{product_id}/sub_items"
---

# Locate a store, then read its inventory

FlyerKit content is **location-scoped**: the same publication id is not globally meaningful
without its location context. This skill establishes that context first, then reads inventory
against it. Operations are cited by method and path because the v4.0 Swagger document declares no
`operationId`.

## Step 1 (optional) — resolve the shopper's region from IP

`GET /fsa/{merchant_name_identifier}`

Required: `merchant_name_identifier` (path), `access_token`. Optional: `ip_override`.

Returns an `fsa` object with a single field, `geo_locate_fsa` — the normalized Forward Sortation
Area for the caller. Use this only when you have no postal/ZIP code from the shopper. `ip_override`
lets you resolve for an address other than the caller's, which is what you need when your server
is making the call on a shopper's behalf.

## Step 2 — find the closest stores

`GET /stores/{merchant_identifier}`

Required: `merchant_identifier` (path), `access_token`, **`postal_code`**.
Accepts a Canadian postal code or a United States ZIP code.

```
GET https://api.flipp.com/flyerkit/v4.0/stores/{merchant_identifier}
      ?access_token={token}&postal_code={postal_code}
```

Returns the closest `store[]` — the spec's summary is "Returns closest stores to a postal/zip
code". Each `store` carries `merchant_store_code` (the retailer's own code), `latitude`,
`longitude`, `address`, `city`, `province`, `phone_number`, and seven day-pair opening times as
flat fields (`sun_open`/`sun_close` … `sat_open`/`sat_close`).

**`merchant_store_code` is the handle you carry forward.** It is the value the `store_code` query
parameter expects everywhere else in the API.

This operation is not paginated and returns a bare array. The count returned is not configurable.

## Step 3 — scope circular content to that store

`GET /publications/{merchant_identifier}?access_token={token}&locale={locale}&store_code={merchant_store_code}`

Passing `store_code` rather than `postal_code` pins content to the exact store the shopper
selected, which matters because pricing and assortment vary by location. Required alongside it:
`locale`, one of `en-CA`, `fr-CA`, `en-US`.

See `skills/flipp-wishabi-browse-retailer-circular.md` for the rest of that flow.

## Step 4 — read live store-level inventory for an offer

`GET /product/{product_id}/sub_items`

Required: `product_id` (path), `access_token`, **`store_code`**. All three are mandatory — there
is no store-agnostic form of this operation, because the data is per-location.

Returns `inventory_sub_item[]`. Two things to know before you parse it:

1. **This model is camelCase**, unlike every other model in the API. Its fields are `sku`,
   `storeCode`, `name`, `imageUrl`, `description`, `price`, `originalPrice`, `promotionText`,
   `url`, `buttons`. The spec's own operation summary says the data comes "from MI9 Retail API" —
   it is an upstream vendor shape passed through, so do not assume the snake_case conventions used
   elsewhere.
2. `buttons` carries `cart_button` objects: `sku`, `storeCode`, `enabled`, `label`,
   `quantityControl`, `quantityControlOption`. Honour `enabled` — a disabled button means the item
   is not purchasable at that store. `quantityControlOption` supplies `unit`, `interval`,
   `minimum`, `maximum` and `default` for a quantity stepper; use those bounds rather than
   inventing your own.

Related but distinct: the `sub_item` model (returned nested, not by this operation) is the
catalogue variant record and carries `average_rating`, `web_url` and `web_commission_url`.

## Errors

**422** is the only declared error status, envelope `{"message": string, "code": number}`:

- `422 {"message":"Missing access_token parameter"}` — token absent from the query string.
- `422` — a missing required parameter. On this flow that is almost always the required
  `postal_code` on `GET /stores/{merchant_identifier}` or the required `store_code` on
  `GET /product/{product_id}/sub_items`.
- `422 {"message":"Invalid publication_id or access token"}` — bad id or bad credential,
  undistinguished.

A postal/ZIP code with no nearby store returns **HTTP 200 with `[]`**, not a 404. Handle the empty
array as "no store in range", and do not treat it as an error. Full catalog:
`errors/flipp-wishabi-problem-types.yml`.

## Privacy and safety

- The `access_token` travels in the query string and the API sets `access-control-allow-origin: *`.
  A browser-side call therefore exposes your partner credential to the shopper and to every
  referrer log in the path. Make these calls **server-side**.
- `ip_override` accepts an arbitrary IP. Sending a shopper's IP to a third party is a privacy
  decision — pass a postal code instead whenever you already have one.
- Store hours and geo coordinates are personal-adjacent when joined to a shopper. Do not persist
  the resolved store against an identified user without a lawful basis.

## Retries

Every operation here is a `GET` and safe to retry. Flipp publishes no rate limits and returns no
`RateLimit-*` or `Retry-After` headers (probed 2026-08-12), so back off on your own schedule.
Inventory is the most volatile data in this API — cache it briefly if at all, and never longer
than the offer's `valid_to`.
