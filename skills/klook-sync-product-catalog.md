---
name: Sync a Klook supplier product catalog and pricing
description: >-
  Pull a supplier's full OCTO product hierarchy and its availability/pricing on a
  schedule, using capability negotiation to get the right level of detail.
api: openapi/klook-octo-openapi-original.json
operations:
  - GET /supplier
  - GET /products
  - GET /products/{id}
  - POST /availability/calendar
  - POST /availability
generated: '2026-07-19'
method: generated
source: >-
  Grounded in openapi/klook-octo-openapi-original.json and
  https://klook.gitbook.io/openapi. These five operations carry no operationId
  in the published spec, so they are referenced by method and path; the
  overlay at overlays/klook-octo-overlay.yaml proposes operationIds for them.
---

# Sync a Klook supplier product catalog and pricing

This is the read side of the Klook (OCTO) integration: harvesting what a
supplier sells, at what price, with what availability. Klook itself runs this
sync against connected suppliers on a configurable cadence — the documentation
gives "every 4 hours for the next 365 days" as an example.

## Headers

```http
Authorization: Bearer {your_API_key}
Octo-Capabilities: octo/pricing, octo/content
```

Capability negotiation is the whole game here:

| Capability | Adds | Status |
|---|---|---|
| `octo/pricing` | Prices on products, availabilities and bookings | Available, mandatory |
| `octo/content` | Rich content and images on product, option and unit | Available |
| `octo/pickups` | Hotel pickup options | Coming soon |
| `octo/webhooks` | Programmatic webhook creation | Coming soon |

Without `octo/pricing` you get a catalog with no prices. Always read the
`Octo-Capabilities` **response** header to confirm what was actually applied.

## Steps

### 1. Identify the supplier

`GET /supplier` returns the supplier and contact details for your API key. One
key maps to one supplier relationship — Klook recommends a unique key per
reseller-supplier pair, so this is how you confirm which relationship you are
operating in.

### 2. Pull the catalog

`GET /products` returns every product available to you. There is **no
pagination**, no `limit`, no cursor — you get the whole list in one response.
Size your timeouts accordingly.

Walk the four-level hierarchy:

```
Supplier
└── Product      (id, internalName, locale, timeZone, allowFreesale,
    │             availabilityType, deliveryFormats, deliveryMethods,
    │             redemptionMethod)
    └── Option   (id, default, internalName, restrictions{minUnits, maxUnits})
        └── Unit (id: "adult"|"child"|..., type, restrictions{minAge, maxAge,
                  idRequired, minQuantity, maxQuantity, paxCount},
                  pricingFrom[])
```

`GET /products/{id}` fetches one product when you only need a delta.

Note that Product/Option ids are **UUIDs** but Unit ids are **human-readable
slugs** (`adult`, `child`). Do not assume a uniform id format.

### 3. Pull availability

Two operations, two jobs:

- `POST /availability/calendar` — one object per **day**, optimized for large
  date ranges. This is the right shape for a bulk catalog sync. **But** it is
  struck through as deprecated in Klook's documentation, so treat it as at-risk
  and plan a migration.
- `POST /availability` — one object per **start time**. Slower, but the only
  operation that returns a bookable `availabilityId`.

For a catalog sync you want the calendar shape; for anything that leads to a
booking you must use Availability Check. See
`skills/klook-reserve-and-confirm-booking.md`.

### 4. Store prices correctly

Pricing objects carry:

```json
{ "original": 7999, "retail": 7999, "net": null, "currency": "USD", "currencyPrecision": 2 }
```

Amounts are integers in **minor units**. Divide by `10^currencyPrecision` — do
not hardcode 2, and do not store as float. `net` is frequently `null`.
`pricingFrom` on a unit is a *from* price, not the bookable price — the
authoritative price for a specific slot comes back on the availability.

### 5. Handle the timezone trap

Each Product carries its own `locale` and `timeZone` (e.g.
`America/Los_Angeles`). Availability ids are ISO 8601 with offset
(`2020-01-01T10:30+08:00`). Never normalize these to your own server timezone
before storing — a booking made against a shifted timestamp will be rejected.

## Rules that apply throughout

**Error handling.** `200 OK` or `400 Bad Request`; branch on the body's `error`
field. Note that `GET /supplier`, `GET /products` and `GET /products/{id}`
declare **no** error responses at all in the published spec — that is a spec
gap, not a guarantee. Handle `INVALID_PRODUCT_ID`, `FORBIDDEN` and
`INTERNAL_SERVER_ERROR` defensively anyway.

**No rate limits are published.** No quota headers, no 429 responses. Sync
frequency is agreed per connection with Klook rather than enforced by a
published limit — be conservative and back off on `INTERNAL_SERVER_ERROR`.

**Change detection.** There is no `updated_since` filter and no ETag support.
Until the Notifications capability ships (see
`asyncapi/klook-notifications-webhooks.yml`, currently "Coming Soon"), full
polling is the only mechanism. When it does ship, `PRODUCT_UPDATE` and
`AVAILABILITY_UPDATE` events will let you re-fetch only what changed.

## Related

- `conventions/klook-conventions.yml` — capability negotiation, headers, sync model
- `data-model/klook-data-model.yml` — full entity graph and enumerations
- `asyncapi/klook-notifications-webhooks.yml` — the announced event surface
- `lifecycle/klook-lifecycle.yml` — deprecation posture on Availability Calendar
