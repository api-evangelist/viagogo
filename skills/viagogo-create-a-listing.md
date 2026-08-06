---
name: viagogo-create-a-listing
description: List tickets for sale on the viagogo platform — check what an event allows, preview the price and proceeds, then create the listing, including for events that do not yet exist in the viagogo catalog.
api: viagogo Inventory API
generated: '2026-08-05'
method: generated
source: openapi/viagogo-inventory.json, openapi/viagogo-catalog.json, https://developer.viagogo.net/docs/guides/creating-a-listing
operations:
- Events_SearchEvents
- Events_GetEvent
- ListingConstraints_GetEventListingConstraints
- ListingConstraints_PutRequestedEventListingConstraints
- SellerListings_CreateListingPreview
- SellerListings_CreateListingPreviewForRequestedEvent
- SellerListings_CreateListing
- SellerListings_CreateListingForRequestedEvent
- SellerListings_Get
- SellerListings_GetByExternalListingId
---

# Create a viagogo listing

Base URL `https://api.viagogo.net/v2` (Catalog reads are on `https://api.viagogo.net`).
Sandbox `https://sandbox.api.viagogo.net`. All responses are `application/hal+json`.

## Before you call anything

- Get an OAuth2 token from `https://account.viagogo.com/oauth2/token`. Creating listings needs
  `write:sellerlistings`; creating one for an event that is not yet on the platform additionally
  needs `write:requestedevents`. Send it as `Authorization: Bearer <access_token>`.
- Send a real `User-Agent`. A missing or invalid one is rejected with `user_agent_required` (400).
- **There is no idempotency key.** Retrying a create is not safe. Always send your own `external_id`
  (see step 4) and check before you retry.

## 1. Find the event

Call `Events_SearchEvents` (`GET /catalog/events/search`) or `Events_GetEvent`
(`GET /catalog/events/{eventId}`) with scope `read:events`.

If you already hold the event's id on another platform, call `Events_GetEventByExternalEventId`
(`GET /catalog/events/external_mappings/{platform}/{externalEventId}`) instead of searching by text.

If the event does not exist on viagogo, do not stop — skip to the requested-event path in step 3.

## 2. Read what the event will accept

Call `ListingConstraints_GetEventListingConstraints`
(`GET /events/{eventId}/listingconstraints`).

This returns `min_ticket_price` / `max_ticket_price` (as `Money`), the permitted `sections` and their
`rows`, and the allowed ticket and split types. Validate your listing against it locally — creating a
listing an event does not permit returns `create_listing_not_allowed` (403), and a price outside the
band returns `validation_failed` (400) with the offending property named in `errors`.

For an event that is not yet on the platform, call
`ListingConstraints_PutRequestedEventListingConstraints` (`PUT /listingconstraints`) with the event
and venue text instead.

## 3. Preview before you commit

Never create blind. Preview returns the pricing viagogo will actually apply:

- Existing event: `SellerListings_CreateListingPreview`
  (`POST /events/{eventId}/sellerlistingpreview`)
- Requested event: `SellerListings_CreateListingPreviewForRequestedEvent`
  (`POST /sellerlistingpreview`)

Send either `ticket_price` **or** `ticket_proceeds` — not neither. Sending neither is the documented
`validation_failed` example: `seller_listing.ticket_price: ["You must provide either 'ticket_price'
or 'ticket_proceeds'"]`.

## 4. Create the listing

- Existing event: `SellerListings_CreateListing` (`POST /events/{eventId}/sellerlistings`)
- Event not yet on the platform (**the recommended path**):
  `SellerListings_CreateListingForRequestedEvent` (`POST /sellerlistings`)

viagogo recommends the requested-event endpoint because it accepts text values for the event name,
local date, venue name, city and country and maps them onto the platform. The listing is created even
when the event does not exist yet; the event is created asynchronously and goes live with your
listing attached.

Always send `external_id` — your own id for this listing. viagogo's published behaviour:

> If an attempt is made to create a listing with an `external_id` that already exists for your user,
> then we will delete the old listing and create the new one.

Read that carefully before you build retries. It de-duplicates, but it is **not** idempotent: a replay
destroys the previous listing and mints a new viagogo listing id. If a create times out, call
`SellerListings_GetByExternalListingId` (`GET /externalsellerlistings/{externallistingId}`) to find
out whether it landed — do not blind-retry.

Both create operations return `201`.

## 5. Preupload barcodes when the event needs them

Set the `barcodes` property on create (or later via `SellerListings_Patch`). Barcodes are allocated
to sales from the **lowest available `seat_ordinal` upward**; reverse `seat_ordinal` against `seat` to
force high-to-low allocation.

Never predict which barcodes viagogo will allocate — the docs warn this causes double sales. Always
read the allocated barcodes back from the sale or the webhook.

If an event requires preuploaded barcodes and you create without them, `undeliverable` is set to
`true` on the listing and it is not purchasable. Subscribe to the seller-listing-updates topic
(`ListingDeliverabilityExpired`) to be told when this happens.

## 6. Confirm

`SellerListings_Get` (`GET /sellerlistings/{listingId}`) with `read:sellerlistings`, or
`SellerListings_GetByExternalListingId` if you only kept your own id.

## Errors you will actually hit

| Code | Status | What to do |
|---|---|---|
| `validation_failed` | 400 | Read the `errors` map, fix the named properties, retry. |
| `create_listing_not_allowed` | 403 | The event cannot be listed for right now — re-read listing constraints. |
| `invalid_seller_listing_action` | 403 | The listing's current state does not support this action. |
| `insufficient_scope` | 403 | Re-request the token with `write:sellerlistings` (+ `write:requestedevents`). |
| `internal_server_error` | 500 | Back off and retry; quote the `vgg-correlation-id` response header to support. |

## Cross-references

- `conventions/viagogo-conventions.yml` — HAL, pagination, sorting, sparse fieldsets, no idempotency
- `scopes/viagogo-scopes.yml` — the full 17-scope union
- `errors/viagogo-problem-types.yml` — the nine published error codes
- `asyncapi/viagogo-webhooks.yml` — the seven webhook topics
