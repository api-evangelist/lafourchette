---
name: Connect a POS to TheFork Manager
description: Register a POS instance with TheFork, serve the order-open callback, and close bills back onto a reservation.
api: openapi/lafourchette-pos-api-openapi.yml
operations:
  - postV1Create
  - putV1PosuuidLogo
  - putV1OrdersOrderuuid
generated: '2026-08-17'
method: generated
source: openapi/lafourchette-pos-api-openapi.yml + https://docs.thefork.io/POS-API/how-it-works
---

# Connect a POS to TheFork Manager

This integration is two-directional, and only half of it lives in TheFork's contract. You call three
operations; TheFork calls two endpoints that **you** have to build.

## Before you start

Ask integrations@thefork.com for a POS API key (company name plus a description of your use case).
You send every request to `https://api.thefork.io/pos` with `X-Api-Key: <key>`. Separately, you
generate a secret of your own (`oauthClientSecret`) — TheFork presents it as a bearer token on every
call it makes back to you.

## Steps

1. **Register the POS instance** — `postV1Create`. The body carries `name`, `homepageUrl`,
   `type` (`jwt` is the only supported value today), your `oauthClientSecret`, and the two URLs you
   will serve: `receiptOpeningUrl` (where TheFork opens an order) and `oauthTokenUrl` (where TheFork
   links a restaurant to your instance).
   Store both identifiers from the response: `uuid` is your POS instance, `consumerId` is your
   company. The instance lands in **PENDING** on production.
2. **Upload your logo** — `putV1PosuuidLogo` with the `uuid` from step 1. `multipart/form-data`, PNG,
   minimum 300px wide, maximum 100Kb. Wrong content type is `415`; unknown uuid is `404`.
3. **Serve the order-open callback.** When a diner is marked ARRIVED or SEATED in TheFork Manager,
   TheFork POSTs to your `receiptOpeningUrl` with `Authorization: Bearer <oauthClientSecret>` and a
   `CustomerId` header carrying the restaurant UUID. The body is one order object or an array of
   them, and it is rich: `orderId`, `dateOfMeal`, `startTime`, `partySize`, `reservationStatus`,
   `mealStatus`, `offer`, `loyaltyAmount`, `prepayment`, and a `customer` object with allergies,
   dietary restrictions, seating preferences and notes. Key the order on `orderId` — you send it back.
   Ignore the deprecated `customerName`, `customerUuid`, `offerName` and `updatedAt` fields; use
   `customer.firstName`/`customer.lastName`, `customer.id` and `offer.name`.
4. **Serve the link callback.** On go-live, the restaurant pastes a token you generated into TheFork
   Manager; TheFork then calls your `oauthTokenUrl` with `Authorization: Bearer <your-secret>`, the
   `CustomerId` header and `{"token":"..."}`. Bind that `CustomerId` to the restaurant's POS instance
   — it is how every later order is routed.
5. **Close the bill** — `putV1OrdersOrderuuid` using the `orderId` TheFork sent you. Body is
   `locale`, `currency`, `totalAmount` and `lines[]` of `{label, category, totalPrice, quantity}`.
   Success is **204 with an empty body**.

## Rules that will bite you

- **Money is in minor units.** `totalAmount` and every `totalPrice` are multiplied by 100 — `7600` is
  76.00. Same for `loyaltyAmount`.
- **Split bills collapse to one order.** If your POS splits a bill, send TheFork a single order
  carrying the total, never one per split.
- **There is no sandbox you can enter alone.** The PENDING instance runs on production; TheFork
  creates a test restaurant and sends its URL and TFM credentials over a secure channel, then runs a
  joint test session before activating you. (`api.preprod.thefork.io` appears in one curl example in
  the docs but is never granted or explained — do not assume access.)
- **You cannot push reservations.** The POS may not create reservations or update reservation status
  in TheFork, and floorplan sync is out of scope. Orders flow POS → TheFork; reservations flow
  TheFork → POS.
- The POS contract declares no error responses. In prose: `400` bad payload, `401` no authorization,
  `404` unknown uuid, `415` wrong content type, `500` general error — including the case where you
  try to create a POS whose name already exists.
