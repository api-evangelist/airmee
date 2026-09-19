---
name: Offer Airmee locker or collection-point delivery
description: Find the parcel lockers or collection points nearest a consumer's zip code, show their addresses, opening hours and delivery windows, and book the parcel to the one the consumer picks.
api: openapi/airmee-integration-api-openapi.yml
operations:
  - getParcelLockersForZipCode
  - parcelLockerDetails
  - getCollectionPointsForZipCode
  - collectionPointDetails
  - dimensionsAndWeight
  - requestDelivery
generated: '2026-09-19'
method: generated
source: openapi/airmee-integration-api-openapi.yml + http://integration.docs.airmee.com.s3-website-eu-west-1.amazonaws.com/
---

# Offer Airmee locker or collection-point delivery

Use this when the checkout should offer pickup at an outdoor Airmee locker (24/7 access) or at a
staffed collection point instead of delivery to the door.

## Before you start

- Same base URL and auth as every Airmee operation: `https://api.airmee.com/integration`, with the
  place-scoped JWT as the raw `Authorization` header value.
- Lockers and collection points are two separate surfaces with the same shape. Pick the pair that
  matches the product you sell:
  - lockers → `getParcelLockersForZipCode` + `parcelLockerDetails`
  - collection points → `getCollectionPointsForZipCode` + `collectionPointDetails`

## Steps

1. **List what is near the consumer.** `GET /get_parcel_lockers_for_zip_code` (or
   `/get_collection_points_for_zip_code`) with `place_id`, `zip_code`, `country` and an optional
   `limit` — "the number of closest parcel lockers to retrieve". Each entry carries `name`,
   `address_details` (street, city, zip, country, country_code and `coordinates`), `distance`,
   `operating_hours`, and a bookable `pickup_interval` / `dropoff_interval` pair. Locker
   `dropoff_interval` also carries `eta` and `eta_formatted`; locker `operating_hours` uses
   `open_all_day` where collection points use `opening_time` / `closing_time` per `day_of_week`.

2. **Render the choice.** Sort by `distance`, show `formatted_as_schedule` for the window rather
   than formatting the epoch seconds yourself, and show the opening hours — for collection points
   they are per weekday, for lockers usually all-day.

3. **Re-read details on selection (optional).** `GET /get_parcel_locker_details` (or
   `/get_collection_point_details`) with the id(s) returns name, address, coordinates and operating
   hours for one or several ids. Use it if the consumer's session outlived the list response.

4. **Check the parcels fit.** `dimensionsAndWeight` (`GET /product_threshold_for_place`) as for a
   home delivery. Locker delivery has the harder physical constraint in practice — an oversized
   parcel that a courier could hand over at a door will not go into a locker.

5. **Book it.** `POST /request_delivery` — the **same operation and the same path as a home
   delivery**. There is no product field: what makes this a locker or collection-point booking is
   the dropoff you supply and the `pickup_interval` / `dropoff_interval` pair you carry over from
   step 1 verbatim. Send `place_id`, `recipient`, `ecomm_id`, `items[]` (one per parcel, with
   `parcel_id`) as usual. The parcel-locker response may include `order.message` alongside
   `order.order_id` and `order.tracking_url`; surface it — it is the only place the API adds
   free-text guidance.

## Limits and gotchas

- `cancelDelivery` is documented under home deliveries only. Locker and collection-point bookings
  return the same `order_id` shape, but the docs do not say whether cancel applies to them —
  confirm with Airmee before relying on it.
- Consumer-side locker access is BankID + Bluetooth through the iBoxen app, and a parcel sits in the
  locker for 7 days before returning to the merchant. None of that is in the API; it is in the
  Airmee support pages, and it is what your support team will be asked about.
- No webhooks and no order read operation: after booking you hold `order_id` and the tracking URL.
