---
name: viagogo-sync-the-event-catalog
description: Mirror the viagogo event catalog into your own system and keep it current — full sync, incremental sync by resource version, category and venue traversal, and mapping your own event identifiers onto viagogo events.
api: viagogo Catalog API
generated: '2026-08-05'
method: generated
source: openapi/viagogo-catalog.json, https://developer.viagogo.net/docs/overview/pagination
operations:
- Events_GetEvents
- Events_GetEventsByCategoryId
- Events_GetAllEventsByCategoryId
- Events_GetEventsByEventIds
- Events_SearchEvents
- Events_GetEvent
- Events_GetEventByExternalEventId
- Events_MapEvent
- Categories_MapCategory
- Venues_GetVenues
- Venues_GetVenue
- VenueConfigurations_GetVenueConfiguration
---

# Sync the viagogo event catalog

Base URL `https://api.viagogo.net` (the Catalog API is **not** under `/v2`).
Sandbox `https://sandbox.api.viagogo.net`. Responses are `application/hal+json`.

Scopes: `read:events` for events, `read:categories` for category search, `read:venues` for venues,
`read:venue_configurations` for seat maps. These are read-only and work with the **Application-Only
(client_credentials) flow** — no user login required.

## 1. Full sync

`Events_GetEvents` (`GET /catalog/events`) is the endpoint viagogo names for this:
"Use this endpoint to sync the entire viagogo event catalog with your application."

Paginate with `page` and `page_size`. Page numbering is **1-based**; omitting `page` returns the
first page. Default page size is 100. The response envelope carries `total_items`, `page`,
`page_size`, `_links` and `_embedded` — follow the `next` link rel rather than incrementing `page`
yourself.

Narrow the sweep with the filters the operation declares: `country_code`, `genre_id`,
`latitude` + `longitude` + `max_distance_in_meters`, and `exclude_parking_passes`.

## 2. Incremental sync — do this, not a nightly full crawl

Two cursors are available on `Events_GetEvents`:

- `updated_since` (date-time) — only items updated since that timestamp.
- `min_resource_version` (uint64) with `sort=resource_version` — a monotonic cursor. Sort by
  `resource_version`, keep the highest value you have seen, and pass it as `min_resource_version`
  on the next run.

Prefer `resource_version`: it is monotonic and does not suffer the clock-skew and
same-timestamp-boundary problems of `updated_since`.

## 3. Traverse by category

- `Categories_MapCategory` (`GET /catalog/categories/map`) matches a free-text `q` to a category.
- `Events_GetEventsByCategoryId` (`GET /catalog/categories/{categoryId}/events`) — events directly
  under a category.
- `Events_GetAllEventsByCategoryId` (`GET /catalog/categories/{categoryId}/allevents`) — every event
  under the category, transitively. Use this one when you want the whole subtree.

## 4. Reconcile your identifiers with viagogo's

Three different tools, and they are not interchangeable:

- `Events_GetEventByExternalEventId`
  (`GET /catalog/events/external_mappings/{platform}/{externalEventId}`) — you already know the
  event's id on a named platform and want the viagogo event. Cheapest and most reliable.
- `Events_GetEventsByEventIds` (`PUT /catalog/events`) — batch hydrate a set of viagogo event ids.
  Note the verb is **PUT**, not GET; the ids go in the request body so the batch is not limited by
  URL length.
- `Events_MapEvent` (`POST /catalog/mapevent`) — you have loose text (event name, date, venue name,
  city, country) and want viagogo to resolve it. Returns a `MapEventResponseModel` carrying the
  matched `EventResource`, `VenueResource` and `CategoryResource`. Use it when you have no id at all.

## 5. Venues and seat maps

`Venues_GetVenues` (`GET /catalog/venues`) and `Venues_GetVenue`
(`GET /catalog/venues/{venueId}`) for venue records. An `Event` carries `venue_config_id`; feed that
to `VenueConfigurations_GetVenueConfiguration`
(`GET /catalog/venues/configurations/{configId}`) to get the seat map — sections, their rows, and
ticket classes. You need this graph before you can place a section-scoped listing offer.

## 6. Keep the payload small

- Sparse fieldsets: `?fields=id,name,_embedded&fields[venue]=city` — comma-separated per type.
- Sorting: `?sort=event_date,-price` — a `-` prefix makes that field descending.

Both are documented cross-cutting conventions and work on paginated collections.

## Handle merges

`Event` and `SellerEvent` carry `merged_events` (`MergedEntity`). viagogo merges duplicate events;
if you cache event ids, resolve merges on each sync or you will hold pointers to ids that have been
folded into another event.

## Errors

`401` unauthenticated · `403` `insufficient_scope` · `404` unknown event/venue/category ·
`500` `internal_server_error` (back off; quote `vgg-correlation-id`).

## Cross-references

- `conventions/viagogo-conventions.yml` — pagination, sorting, sparse fieldsets, incremental sync
- `data-model/viagogo-data-model.yml` — the Category → Event → Venue → VenueConfiguration graph
