---
name: Cancel or amend a Klook booking
description: >-
  Cancel a confirmed Klook (OCTO) booking within its cut-off window, or amend one
  in place, handling the cancellable/cut-off preconditions correctly.
api: openapi/klook-octo-openapi-original.json
operations:
  - get-bookings-:uuid
  - get-bookings
  - patch-bookings-:uuid
  - delete-bookings-:uuid
generated: '2026-07-19'
method: generated
source: >-
  Grounded in openapi/klook-octo-openapi-original.json and
  https://klook.gitbook.io/openapi. Every operationId listed appears verbatim in
  the spec.
---

# Cancel or amend a Klook booking

Post-sale operations on the Klook (OCTO) API. Both cancellation and amendment
are **conditional** — the booking itself tells you whether the operation is
permitted, and the answer changes with time.

## Headers

```http
Authorization: Bearer {your_API_key}
Content-Type: application/json
Octo-Capabilities: octo/pricing
```

## Steps

### 1. Read the booking first — always

`GET /bookings/{uuid}` (operationId `get-bookings-:uuid`).

Never cancel blind. Check:

- `booking.cancellable` — if this is not `TRUE`, cancellation will be refused.
- The booking's current `status`. An `ON_HOLD` reservation that simply expires
  needs no cancellation at all.
- The cancellation cut-off window for the product.

To find a booking when you only have your own reference, use `GET /bookings`
(operationId `get-bookings`) with filters rather than scanning.

### 2. Amend in place where you can

`PATCH /bookings/{uuid}` (operationId `patch-bookings-:uuid`).

Amending is preferable to cancel-and-rebook: it preserves the held inventory. A
cancel followed by a new reservation can fail at the reservation step and leave
the guest with nothing, because the released slot may be taken in between.

### 3. Cancel only when the preconditions hold

`POST /bookings/{uuid}/cancel` (operationId `delete-bookings-:uuid`).

Note the mismatch: the published spec names this operation
`delete-bookings-:uuid`, but it is a **POST** to `/bookings/{uuid}/cancel`. Use
the path and method; the operationId is a legacy artifact of the spec.

Preconditions, both required:

1. `booking.cancellable` is `TRUE`
2. You are within the booking cancellation cut-off window

Fail either and you get `UNPROCESSABLE_ENTITY` — for example
`"errorMessage": "..."` after the cut-off has elapsed.

### 4. Confirm the outcome

Re-read with `get-bookings-:uuid`. Do not infer success from a `200` alone —
the OCTO envelope returns `200 OK` on success and `400` with an `error` body on
failure, so a `200` with an `error` field present is still a failure.

## Rules that apply throughout

**`UNPROCESSABLE_ENTITY` is terminal, not transient.** It means the request was
well-formed but the business rule said no — past the cut-off, not cancellable.
Retrying will not help. Surface it to a human.

**`INVALID_BOOKING_UUID` on a cancel** usually means you are holding a uuid for
a reservation that already expired. Nothing to cancel.

**No idempotency key.** If a cancel request times out, re-read the booking
before retrying — the cancellation may have landed.

**No refund surface.** The OCTO core endpoints model cancellation, not refunds.
Money movement back to the guest is settled through the commercial relationship,
not through this API.

## Error reference

| `error` | Meaning | Action |
|---|---|---|
| `UNPROCESSABLE_ENTITY` | Not cancellable, or past cut-off | Stop — escalate to a human |
| `INVALID_BOOKING_UUID` | Unknown uuid, or reservation expired | Re-read; likely nothing to do |
| `BAD_REQUEST` | Malformed payload | Fix and retry |
| `FORBIDDEN` (403) | Key revoked or out of scope | Stop |

Full catalog: `errors/klook-error-codes.yml`.

## Related

- `skills/klook-reserve-and-confirm-booking.md` — the forward flow
- `data-model/klook-data-model.yml` — booking lifecycle state transitions
- `conventions/klook-conventions.yml` — error envelope and retry posture
