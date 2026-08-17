---
name: Sync guests and reservations into a CRM
description: Keep a CRM in step with TheFork using the webhook event flow plus the B2B list-then-fetch read operations.
api: openapi/lafourchette-b2b-api-openapi.yml
operations:
  - getV1Customers
  - getV1CustomersId
  - getV1Reservations
  - getV1ReservationsId
  - getV1RestaurantsIdCustomersPhonePhoneNumber
generated: '2026-08-17'
method: generated
source: openapi/lafourchette-b2b-api-openapi.yml + https://docs.thefork.io/B2B-API/event-webhook-flow
---

# Sync guests and reservations into a CRM

Two mechanisms, and you want both: webhooks tell you *that* something changed, the read operations
tell you *what* it is. TheFork's webhook payloads are deliberately thin.

## Real-time path

1. Stand up an HTTPS POST endpoint with a token you generate in the query string, e.g.
   `https://crm.example.com/api/thefork?token=<your-token>`. Send the URL to TheFork; they configure
   it by hand. That token is the only authentication on the flow — there is no signature header.
2. **Acknowledge within a few seconds** with HTTP 200 and `{"data":{}}`. TheFork retries on timeout,
   so anything slower turns into duplicate deliveries.
3. Process asynchronously. The payload carries `entityType`, `eventType`, `uuid`, `groupUuid` and —
   for reservations and reviews — `restaurantUuid`.
4. Fetch the record: `customerCreated`/`customerUpdated` → `getV1CustomersId`;
   `reservationCreated`/`reservationUpdated` → `getV1ReservationsId`.

## Backfill and reconciliation path

- `getV1Customers` and `getV1Reservations` both take `startDate` + `endDate` (**required**), plus
  either `groupUuid` or `restaurantUuid` (**one of the two is required**), and page with
  `page`/`limit` (limit default 100, max 10000).
- On reservations, `filterBy` chooses which date the range applies to: `updatedDate` (the default,
  and the one you want for sync) or `mealDate` (the one you want for a service-day view).
- These lists return **UUIDs only** — `{ data: [...uuids], totalCount, page, limit }`. Loop the page,
  then call the detail operation per record. Budget 1 + N requests per page.
- Run a nightly `updatedDate` sweep over the last 24–48 hours to catch webhook deliveries you dropped.
  There is no replay or backfill endpoint.

## Phone lookup

`getV1RestaurantsIdCustomersPhonePhoneNumber` resolves a caller to a customer at one restaurant, and
returns `404 PHONE_NUMBER_NOT_FOUND` when there is no match. Pair it with
`postV1CallCenterCallRecognitions` if you are wiring a call centre, and
`patchV1IntegrationStatus` to report integration health back to TheFork.

## Handle the data properly

`getV1CustomersId` returns `allergiesAndIntolerances`, `dietaryRestrictions`, `riskLevel`,
`birthDate`, `email`, `phone` and marketing `optins` about named diners. That is health-adjacent and
consent-bearing personal data under the partner contract. Store the minimum you need, honour the
`optins` object, and expect a `reviewValidityChanged` webhook when a diner exercises a GDPR deletion.

## Rules

- Errors are `{ statusCode, error, data: { code } }` — match `data.code`
  (`CUSTOMER_NOT_FOUND`, `RESERVATION_NOT_FOUND`, `PHONE_NUMBER_NOT_FOUND`).
- 401 on any operation means the Auth0 token expired (8600s); mint a new one and retry once.
- Rate limits are contractual and unpublished; back off on 429 and keep the sweep serial.
