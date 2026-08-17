---
name: Book a table through TheFork
description: Find bookable capacity at a TheFork restaurant and create, modify or cancel a reservation using TheFork B2B API.
api: openapi/lafourchette-b2b-api-openapi.yml
operations:
  - getV1RestaurantsIdAvailabilities
  - getV1RestaurantsIdPartySizes
  - getV1RestaurantsIdTimeslots
  - getV1RestaurantsIdOffers
  - postV1RestaurantsIdReservations
  - patchV1ReservationsId
  - patchV1ReservationsIdCancel
  - getV1ReservationsId
generated: '2026-08-17'
method: generated
source: openapi/lafourchette-b2b-api-openapi.yml + https://docs.thefork.io/B2B-API/introduction
---

# Book a table through TheFork

Use this to build a booking funnel on top of TheFork's B2B API. You need a `restaurantUuid`, and
credentials issued by TheFork's integrations team — there is no self-serve signup.

## Authenticate first

Mint an Auth0 client-credentials token, then send it as a bearer token. The B2B contract declares no
security scheme, so nothing in the spec will tell you this:

```
POST https://auth.thefork.io/oauth/token
  audience=https://api.thefork.io
  grant_type=client_credentials
  client_id=...&client_secret=...
```

The token is valid for 8600 seconds. Cache it and reuse it — TheFork explicitly asks integrators not
to request a new token while the previous one is still valid. Send it on every call as
`Authorization: Bearer <access_token>` against `https://api.thefork.io/manager`.

## Steps

1. **Check what the restaurant can take.** Call `getV1RestaurantsIdAvailabilities` for the date range
   you care about. Each entry is a date with `hasNormalStock` and an `offerList`.
2. **Narrow the party size.** Call `getV1RestaurantsIdPartySizes` for the date to see which party
   sizes actually have availability. Do not guess — a party size that is not returned will not book.
3. **Pick a time.** Call `getV1RestaurantsIdTimeslots` for the date. Each slot carries `datetime`,
   `hasNormalStock`, its `offers`, and `hasPaymentGuaranteeRequirement` with a
   `guaranteeRequirementList`. If the guarantee flag is set, the diner will have to give a card
   guarantee — surface that before you take the booking, not after.
4. **Attach an offer if one applies.** `getV1RestaurantsIdOffers` returns preset menus and promotions
   with `uuid`, `name`, `description` and `price`. Pass the offer's uuid on the create call.
5. **Create the reservation.** `postV1RestaurantsIdReservations` on the restaurant. On success you get
   the reservation back with its `reservationUuid` and `status` (normally `RECORDED`).
6. **Modify or cancel.** `patchV1ReservationsId` changes the meal date and party size.
   `patchV1ReservationsIdCancel` cancels. Re-read with `getV1ReservationsId` when you need the full
   record — including `mealStatus`, `billAmount` and `reservationChannel`.

## Rules that will bite you

- **There is no idempotency key.** If `postV1RestaurantsIdReservations` times out, you cannot safely
  retry: a duplicate will surface as `409 DOUBLE_BOOKING` only if the slot is genuinely taken. Re-read
  availabilities and reconcile before retrying, and log the outbound attempt yourself.
- **409 is a routine outcome, not a bug.** `DOUBLE_BOOKING` on create/update means the slot went while
  you were deciding — go back to step 3. `DOUBLE_CANCELLATION` on cancel means it was already
  cancelled; treat that as success.
- **Match on `data.code`, not on the message.** Errors are `{ statusCode, error, data: { code } }`.
  One operation breaks the pattern: `postV1RestaurantsIdReservations` returns the sentence
  `"Restaurant not found"` where everything else returns `RESTAURANT_NOT_FOUND`.
- **Every operation can return 401.** Mint a fresh token and retry once; do not loop.
- **429 exists but the limits do not.** Rate limits are contractual and unpublished, and no
  rate-limit response headers are sent. Back off exponentially on 429.
- Every identifier is a UUID; there are no id prefixes to tell entity types apart.
