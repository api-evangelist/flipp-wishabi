---
name: pull-circular-offers
description: Pull filtered, paginated offer items out of a retailer's Flipp publications — either across every publication or within one — and expand a single offer to its full detail record with coupons, reviews and rich media.
api: Flipp FlyerKit API v4.0
base_url: https://api.flipp.com/flyerkit/v4.0
generated: '2026-08-12'
method: generated
source: openapi/flipp-wishabi-flyerkit-openapi.yml
operations:
  - "GET /publications/{merchant_identifier}/products"
  - "GET /publication/{publication_id}/products"
  - "GET /product/{product_id}"
---

# Pull offers out of a retailer's circulars

This is the flow behind an email campaign, a "this week's deals" module, or a search index over a
retailer's weekly ad. The FlyerKit v4.0 Swagger document declares no `operationId`, so operations
are cited by method and path.

## Choose your scope first

Two operations return offers, and they differ only in scope:

| Scope | Operation | Returns |
| --- | --- | --- |
| Across **all** of a merchant's publications | `GET /publications/{merchant_identifier}/products` | `merchant_product[]` |
| Within **one** publication | `GET /publication/{publication_id}/products` | `product[]` |

In v4.0 the two payload shapes are field-for-field identical. The cross-publication operation was
**added in v4.0** — it did not exist in v3.0.

## Step 1 — list offers across a merchant

`GET /publications/{merchant_identifier}/products`

Required: `merchant_identifier` (path), `access_token`, `locale`, **`store_code`**.
Note `store_code` is required here even though it is optional on the publication list operation.

```
GET https://api.flipp.com/flyerkit/v4.0/publications/{merchant_identifier}/products
      ?access_token={token}&locale=en-US&store_code={store_code}&size=50&offset=0
```

Filtering (all optional): `keywords`, `tags`, `category`, `display_type`, `postal_code`,
`see_future`, `show_storefronts`. `keywords` and `tags` were **added in v4.0**.

## Step 2 — or list offers inside one publication

`GET /publication/{publication_id}/products`

Required: `publication_id` (path), `access_token`.
This is the only offer operation that also accepts `sort_by` and `sort_order`. It additionally
accepts `store_code`, `postal_code`, `keywords`, `tags`, `category` and `display_type`.

## Pagination — read this before you loop

Both offer operations accept `page`, `size` **and** `offset`. The spec documents no precedence
between them, no default `size` and no maximum `size`.

The response is a **bare JSON array with no envelope** — there is no `total`, no `next`, no
cursor and no `Link` header. You cannot tell whether more results exist except by requesting the
next window and seeing whether it comes back empty. So:

1. Pick **one** paging idiom — `size` + `offset` — and do not mix it with `page`.
2. Loop until a page returns fewer than `size` items, or returns `[]`.
3. Cap your loop. Nothing in the contract bounds the result set.

The other six collection operations in this API are **not paginated at all** and return unbounded
arrays.

## Step 3 — expand one offer to its detail record

`GET /product/{product_id}`

Required: `product_id` (path), `access_token`. Optional: `postal_code`, `store_code`.

Returns a single `detailed_product`. This is a second round trip per item — there is no `expand`,
`fields` or `include` parameter anywhere in FlyerKit, so depth is only available by re-fetching.
Only call it for items you are actually going to render in full.

The detail record adds `image_url`, `cutout_image_url`, `disclaimer_text` and
`hosted_coupon_image` over the collection shape, and is where coupons, reviews, rich media, specs
and features attach. `product_coupon` carries `loyalty_program_coupon_id` and `loyalty_program_id`,
which is what lets an offer be clipped to a retailer's loyalty card.

## Rendering prices correctly

Do not compose your own price string. The offer models carry a three-part presentation vocabulary
that the retailer's legal and merchandising teams control:

- `pre_price_text` + `price_text` + `post_price_text` — concatenate in that order.
- `current_price` / `original_price`, and `current_price_range` / `original_price_range` for items
  priced as a range.
- `dollars_off`, `percent_off`, `sale_story`.
- `disclaimer_text` on the detail record — render it if it is present.
- `valid_from` / `valid_to` — a price outside its validity window must not be shown as current.

`categories` is an **array** in v4.0. It was a single string named `category` in v3.0 and was
renamed and retyped in the same change — if you are porting v3.0 code, this will break silently.
Note that the `sub_item` model still carries the old singular `category` string.

`custom_id_field_1`, `_2` and `_3` are retailer-defined passthrough identifiers — use them to join
FlyerKit offers to your own catalog.

## Errors

The only declared error status is **422**, envelope `{"message": string, "code": number}`:

- `422 {"message":"Missing access_token parameter"}` — token absent.
- `422 {"message":"Invalid publication_id or access token"}` — bad id or bad credential; the
  message does not distinguish them.
- `422` also covers a missing required parameter — most commonly the `store_code` that
  `GET /publications/{merchant_identifier}/products` requires and its sibling operation does not.

No `401`, `403`, `404`, `429` or `5xx` is declared anywhere in the contract. Full catalog:
`errors/flipp-wishabi-problem-types.yml`.

## Retries and freshness

All three operations are `GET` and safe to retry. No rate limits are published and no
`RateLimit-*` or `Retry-After` header is returned (probed 2026-08-12), so choose your own
conservative backoff.

There is no event, webhook or streaming surface — FlyerKit is poll-only. A new flyer run going
live will not be pushed to you. Poll on the retailer's known circular cadence and use
`available_from` / `available_to` on the parent publication to decide when a new run has started.
