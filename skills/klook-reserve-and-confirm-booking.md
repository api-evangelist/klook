---
name: Reserve and confirm a Klook booking
description: >-
  Take a Klook (OCTO) product from availability check through a two-phase
  reservation to a confirmed, ticketed booking, handling the ON_HOLD expiry
  window correctly.
api: openapi/klook-octo-openapi-original.json
operations:
  - GET /products
  - POST /availability
  - post-bookings
  - post-bookings-uuid-extend
  - post-bookings-uuid-confirm
  - get-bookings-:uuid
generated: '2026-07-19'
method: generated
source: >-
  Grounded in openapi/klook-octo-openapi-original.json and
  https://klook.gitbook.io/openapi. Every operationId listed above appears
  verbatim in the spec; operations the published spec leaves without an
  operationId are referenced by method and path.
---

# Reserve and confirm a Klook booking

Klook's Open API is an implementation of **OCTO**. It is **supplier-hosted**: you
call the supplier's own endpoint at `https://{supplier_endpoint}/octo/{path}`,
not a Klook host. Booking is deliberately **two-phase** — you reserve inventory,
then confirm it — because the API defines no idempotency key.

## Before you start

Every request needs three headers:

```http
Authorization: Bearer {your_API_key}
Content-Type: application/json
Octo-Capabilities: octo/pricing, octo/content
```

- The API key is issued by the **supplier**, not by Klook.
- `Octo-Capabilities` is **required**. Omitting `octo/pricing` means you get no
  prices back. If you cannot set headers, use `?_capabilities=octo/pricing,octo/content`.
- The response echoes an `Octo-Capabilities` header telling you which
  capabilities were actually initialized — check it rather than assuming.

## Steps

### 1. Find the product

`GET /products` returns every product available to you; `GET /products/{id}`
returns one. There is **no pagination** — expect the full list.

Read down the hierarchy: a **Product** has **Options**, an Option has **Units**
(`adult`, `child`, `senior`). You need a `productId`, an `optionId` and the
`unitId`s your guests map to. Note `allowFreesale` and `availabilityRequired` on
the product — they tell you whether step 2 is strictly mandatory.

### 2. Check availability — this is not optional in practice

`POST /availability` (Availability Check).

This is the **only** operation that returns an `availabilityId`, and an
`availabilityId` is the **only** valid input to a reservation. Even when
`allowFreesale` is true, run it anyway to catch closures.

Do **not** use `POST /availability/calendar` for this. It is struck through as
deprecated in Klook's documentation and returns one object per day rather than
per start time — it cannot give you a bookable `availabilityId`.

The `availabilityId` is an ISO 8601 timestamp with offset, e.g.
`2020-01-01T10:30+08:00`.

### 3. Reserve — phase one

`POST /bookings` (operationId `post-bookings`).

Send the `availabilityId` from step 2 plus your `unitItems`. The booking comes
back with `status: ON_HOLD` and a `uuid`. Inventory is now held for you while
you collect payment and contact details.

**The hold expires.** The TTL is set by the supplier, not fixed by the spec.

If this returns `UNPROCESSABLE_ENTITY` with `"Activity inventory not available"`,
the slot went away between step 2 and step 3. Go back to step 2 for a fresh
`availabilityId` — do not retry the reservation with the stale one.

### 4. If you need more time, extend — do not re-reserve

`POST /bookings/{uuid}/extend` (operationId `post-bookings-uuid-extend`).

Extending is always correct where re-reserving is not. Because there is no
idempotency key, a second `POST /bookings` creates a **second** hold on the same
inventory rather than deduplicating.

### 5. Confirm — phase two

`POST /bookings/{uuid}/confirm` (operationId `post-bookings-uuid-confirm`).

This finalizes the sale and makes the booking ready to use. Tickets/vouchers are
delivered per the product's `deliveryMethods` (`VOUCHER`, `TICKET`) and
`deliveryFormats` (`PDF_URL`, `QRCODE`).

If this returns `INVALID_BOOKING_UUID`, the reservation **already expired**.
Start again from step 2. Do not treat this as a transient error.

### 6. Verify

`GET /bookings/{uuid}` (operationId `get-bookings-:uuid`) returns the current
status and details. Use `GET /bookings` (operationId `get-bookings`) with filters
to reconcile in bulk.

## Rules that apply throughout

**Retry safety.** There is no `Idempotency-Key`. Never blindly retry a write. If
a `POST /bookings` times out, call `get-bookings` and check whether the
reservation actually landed before trying again.

**Error handling.** The API returns `200 OK` or `400 Bad Request` for almost
everything — the real signal is the `error` field in the body, never the HTTP
status. Branch on `error`, not on the status code:

| `error` | What to do |
|---|---|
| `INVALID_AVAILABILITY_ID` | Re-run step 2 |
| `INVALID_BOOKING_UUID` | Hold expired, or wrong uuid — start from step 2 |
| `UNPROCESSABLE_ENTITY` | No inventory, or past a cut-off — re-check availability |
| `BAD_REQUEST` | Your payload is malformed — fix and retry |
| `FORBIDDEN` (403) | Key revoked or out of scope — stop, do not retry |
| `UNAUTHORIZED` | No key sent — stop, do not retry |

Full catalog: `errors/klook-error-codes.yml`.

**Localization.** `errorMessage` is translated per the `Accept-Language` header.
Log the `error` code, not the message.

**Money.** Prices are integers in **minor units** with an explicit
`currencyPrecision` and ISO 4217 `currency`. `7999` with precision `2` is
`79.99`, not `7999`.

## Related

- `conventions/klook-conventions.yml` — cross-cutting semantics
- `errors/klook-error-codes.yml` — full error catalog
- `data-model/klook-data-model.yml` — entity graph and booking lifecycle
- `authentication/klook-authentication.yml` — auth profile
