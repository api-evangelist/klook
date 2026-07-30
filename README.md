# Klook

Klook is a Hong Kong-headquartered travel and experiences booking platform for the "things to do" sector — attractions, tours and activities, theme parks, food and beverage, WiFi and SIM cards, and transportation passes.

Klook publishes an **Open API specification** for merchants, suppliers, reservation systems and channel managers. The notable finding is that Klook did not invent a proprietary contract: its API **is an implementation of [OCTO](https://www.octo.travel/)** (Open Connectivity for Tours, Activities and Attractions), the open standard for the in-destination experiences industry, and Klook distributes the OCTO 1.0 OpenAPI document itself as its reference schema.

## Integration direction

This is an inverted integration. The **supplier implements** the OCTO core endpoints at their own host and **Klook is the API consumer**:

```
https://{your endpoint name}/octo/{path}
```

There is no Klook-hosted API base URL, no Klook-issued API key (the supplier issues it), and no Klook SDK — because there is no Klook-side client for a partner to install. Onboarding is a managed, human-in-the-loop process run with a Klook BD team.

## API surface

Core endpoints (mandatory): **Supplier**, **Products**, **Availability**, **Bookings** — 12 operations across 10 paths, 22 schemas, OpenAPI 3.1.0.

Optional behaviour is negotiated per-request through the required `Octo-Capabilities` header (`octo/pricing`, `octo/content`, and `octo/pickups` / `octo/webhooks` / `notifications` / `questions` marked "Coming Soon").

Booking is deliberately **two-phase** — reserve into `ON_HOLD`, then confirm — which stands in for the idempotency keys the API does not define.

## Artifacts

| Artifact | Method |
|---|---|
| `openapi/` | searched — OCTO 1.0 OpenAPI 3.1.0, published from Klook's docs |
| `llms/` | searched — Klook publishes a real `/llms.txt` |
| `authentication/` | searched — Bearer API key, supplier-issued |
| `errors/` | searched + derived — 10-code registry, custom `{error, errorMessage}` envelope (not RFC 9457) |
| `conventions/` | searched — capability negotiation, two-phase booking, no pagination, no idempotency |
| `conformance/` | searched — OCTO 1.0, OpenAPI 3.1, ISO 8601/4217, RFC 4122 |
| `changelog/` | searched — dated Specs Updates table |
| `lifecycle/` | searched — no versioning policy, no status page, no SLA |
| `data-model/` | derived — Supplier > Product > Option > Unit hierarchy + booking lifecycle |
| `asyncapi/` | searched — Notifications webhook catalog (3 event types, "Coming Soon") |
| `overlays/` | generated — adds missing operationIds, uniform security, deprecation marker |
| `mcp/` | derived — candidate tool surface (no official Klook MCP server exists) |
| `skills/` | generated — 3 Agent Skills grounded in real operationIds |
| `packages/` | searched — **no first-party SDKs**; third-party packages only |
| `well-known/` | searched — no `/.well-known/` documents published (all 404) |
| `security/` | probed — TLS 1.3, DMARC quarantine, no DNSSEC, no CAA |

## Notable gaps

- No public status page, SLA, deprecation policy or Sunset header support.
- No published rate limits, quota headers or 429 responses.
- No pagination on `GET /products`.
- No official GitHub organization; no first-party SDKs.
- No `/.well-known/` discovery, no verified bug bounty or trust center (the `klook.com/bugbounty/` page now 404s and no HackerOne program exists).
- The Specs Updates changelog has not been updated since February 2025.
- `POST /availability/calendar` is struck through as deprecated in the docs but remains undeprecated in the spec.

Backed by: bessemer-venture-partners, hongshan, softbank-vision-fund — https://www.klook.com/en-US/

Docs: https://klook.gitbook.io/openapi
