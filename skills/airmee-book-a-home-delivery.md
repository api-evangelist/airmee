---
name: Book an Airmee home delivery
description: Check that Airmee serves an address, quote the delivery windows a consumer can pick from at checkout, confirm the parcels are within the place's size limits, and book the delivery.
api: openapi/airmee-integration-api-openapi.yml
operations:
  - serviceAreaAvailability
  - deliveryIntervalsForCheckout
  - deliveryIntervals
  - dimensionsAndWeight
  - requestDelivery
  - cancelDelivery
generated: '2026-09-19'
method: generated
source: openapi/airmee-integration-api-openapi.yml + http://integration.docs.airmee.com.s3-website-eu-west-1.amazonaws.com/
---

# Book an Airmee home delivery

Use this when a Swedish e-commerce order needs to be handed to Airmee for same-day or next-day
delivery to the consumer's door.

## Before you start

- Base URL is `https://api.airmee.com/integration`. Staging is the same path on
  `https://staging-api.airmee.com/integration` — the docs state the rule as "for respective
  production url omit 'staging-' prefix".
- Send the JWT as the **raw** value of the `Authorization` header. No `Bearer` prefix:
  `Authorization: <your JWT>`. The token is long-lived and scoped to one **pickup place**; it comes
  from Airmee onboarding, and there is no token endpoint to call.
- You need the `place_id` (a UUID) for the store or warehouse the parcels leave from. Ten of the
  twelve operations take it.
- **There is no idempotency mechanism.** If `requestDelivery` times out you cannot safely retry it
  blind — you may create a second physical delivery. Decide up front how you will handle that
  (see step 5).

## Steps

1. **Check the address is serviceable.** Call `serviceAreaAvailability`
   (`GET /service_area_availability_for_zip_code`) with `place_id`, `zip_code` and `country` (e.g.
   `SE`). Pass `street_and_number` and `city` as well when you have them — with an address it
   geocodes the exact point rather than the zip code. Read `inside_service_area`. If it is `false`,
   stop; Airmee is not an option for this order.

2. **Quote the windows.** At checkout call `deliveryIntervalsForCheckout`
   (`GET /checkout_delivery_intervals_for_zip_code`) with `place_id`, `zip_code`, `country` and a
   `date` offset — the offset is "current time + however long it takes you to pack", in
   `Europe/Stockholm`. You get back `list_of_schedules[]`, each with a `pickup_interval` and a
   `dropoff_interval` (epoch seconds `start`/`end`, plus `formatted_as_schedule` to display).
   An **empty list is a documented outcome**: it means no window can be allocated for that address.
   Use `deliveryIntervals` (`/delivery_intervals_for_zip_code`) instead when you are a TA system
   revalidating a slot the consumer already chose rather than offering a choice.

3. **Check the parcels fit.** Call `dimensionsAndWeight` (`GET /product_threshold_for_place`) with
   `place_id` and compare each parcel against `threshold_values.height`, `.length`, `.width` and
   `.weight`. The contract does not state the units — read them from your onboarding material, do
   not assume. These limits are per place, so cache them per `place_id`, not globally.

4. **Book it.** `POST /request_delivery` (`requestDelivery`) with:
   - `place_id`
   - `recipient`: `name`, `phone_number` (national number, integer),
     `phone_number_country_code` (e.g. `46`), optional `email`
   - `ecomm_id`: your own order/shipment id (the TA sändnings-ID)
   - `dropoff_address`: `street_and_number`, `city`, `zip_code`, `country`, plus optional
     `apartment`, `floor`, `door_code`, and `latitude`/`longitude` if you geocoded it yourself
   - `items[]`: **one object per physical parcel**, each with `parcel_id` (the kolli-ID — mandatory
     if you print labels yourself) and optionally `length`, `width`, `height`, `weight`, `volume`,
     `name`, `unit_price`, `quantity`
   - `pickup_interval` and `dropoff_interval`: pass the pair from step 2 back **verbatim**
   - optional `checks`: `min_age`, `verify_id`, `take_signature` for age-restricted or high-value
     goods; optional `message_to_courier`
   Keep `order.order_id` and `order.tracking_url` from the response. That is all you get — **there
   is no operation to read an order back**, so persist both immediately against your `ecomm_id`.

5. **Handle a failed or uncertain booking.**
   - `400 ValidationError` — `extraMessage` names the missing parameter. Fix and resend.
   - `412 GeocodeError` — the address could not be geocoded. Correct it, or supply
     `dropoff_address.latitude` and `.longitude` and resend.
   - `404 NotFoundError` — the `place_id` is wrong for this token.
   - `401 Unauthorized` — the JWT is missing or rejected. There is nothing to refresh; escalate.
   - `500 DatabaseConnectionError` — server-side. **Do not blind-retry a write.** Because there is
     no idempotency key and no order read, a retry can double-book. Quote the `x-amzn-requestid`
     response header to Airmee support, or reconcile with the retailer before resending.

6. **Cancel if you must, early.** `POST /cancel_delivery` (`cancelDelivery`) with `place_id` and
   `order_id` returns `deletion_status`. It only works for "a delivery that was not already picked
   up", and **Airmee publishes no time window for that** — there is no cutoff you can compute and
   no status you can read first, so cancel as soon as you know, and treat a `404 NotFoundError` as
   "too late or wrong ids".

## Limits and gotchas

- No rate limits are documented and no `RateLimit-*` or `Retry-After` headers are returned. Be
  conservative on the read operations; they are the ones you will call per page view.
- No webhooks. Delivery progress reaches the consumer by SMS and the Airmee app, and reaches you
  only through the tracking URL — plan for no machine-readable status push.
- The published contract dates from 2022-11-02 and the first-party PHP SDK from 2017. Treat the SDK
  as a reference, not a maintained client.
