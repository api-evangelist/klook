# Klook

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

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
