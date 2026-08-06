---
name: viagogo-subscribe-to-webhooks
description: Stand up a viagogo webhook receiver — create the subscription, choose only the topics you handle, verify delivery with a ping, and process each topic payload correctly.
api: viagogo Webhooks API
generated: '2026-08-05'
method: generated
source: openapi/viagogo-webhooks.json, asyncapi/viagogo-webhooks.yml
operations:
- Webhooks_Post
- Webhooks_Get
- Webhooks_Get2
- Webhooks_Patch
- Webhooks_Delete
- Webhooks_PingWebhook
---

# Subscribe to viagogo webhooks

Base URL `https://api.viagogo.net/v2`, sandbox `https://sandbox.api.viagogo.net/v2`.
Scopes: `read:webhooks` to read, `write:webhooks` to create, update, delete or ping.

## 1. Create the subscription

`Webhooks_Post` (`POST /webhooks`) → `201`.

```json
{
  "name": "orders-receiver",
  "url": "https://your-host.example/viagogo/hooks",
  "authorization_header": "<a long random secret you generate>",
  "topics": ["<topic>", "..."]
}
```

viagogo's own guidance: "You should only subscribe to the specific topics that you plan on handling
so that you can limit the number of HTTP requests to your server."

**On `authorization_header`, know what you are getting.** viagogo sends this value back to you as the
`Authorization` header on every delivery. That is the *entire* authentication mechanism — there is no
HMAC signature header, no timestamp, no replay protection and no mTLS option. So:

- Generate it with a CSPRNG and treat it as a bearer secret.
- Compare it in constant time on receipt, and reject anything that does not match.
- Terminate TLS yourself; never accept deliveries over plain HTTP.
- Rotate it with `Webhooks_Patch` on a schedule and after any suspected exposure.
- Do not treat receipt as proof of origin the way an HMAC signature would let you.

## 2. Verify the receiver before you trust it

`Webhooks_PingWebhook` (`POST /webhooks/{webhookId}/ping`) → `202`. It delivers a `PingTopic` payload
to your configured url. Do this in sandbox first, and again in production immediately after creating
the subscription. It is the only self-service delivery test viagogo publishes.

## 3. Handle the topics

Every payload carries `topic`, `action`, `_links` and `_embedded`; most also carry `barcodes`.
Branch on `topic` first, then on `action`.

| Topic payload | Fires when | What you do |
|---|---|---|
| `PingTopic` | You ping the webhook | Return 2xx — this is your health check |
| `SalesTopic` | Something affects a sale, e.g. you sell tickets | Start the fulfillment flow. **Barcodes are in the notification object, not fetched from the sale** |
| `ProvisionalSaleTopic` | A provisional sale occurs | The normal `SalesTopic` will confirm it. To reject: ignore this, and reject the sale when it arrives normally |
| `CancelProvisionalSaleTopic` | A provisional sale is cancelled | Release the hold |
| `SaleUpdatesTopic` | A sale changed and you did not initiate it | Filter on `action`. `FailedBarcodeValidation` → re-upload barcodes for that order |
| `SellerListingUpdatesTopic` | A listing changed and you did not initiate it | Filter on `action`. `FailedBarcodeValidation` → re-upload barcodes for that listing. `ListingDeliverabilityExpired` → the listing is no longer deliverable; preupload barcodes/PDFs for electronic tickets, or opt paper tickets into the LMS program |
| `ReTransferTicketTopic` | The buyer never received the ticket | Re-transfer from the third-party provider |

`topic` is typed as a plain string in the published specification with no enum, so the exact wire
values are not documented. Read them off a real delivery — a ping in sandbox is the cheapest way —
and keep an unrecognised-topic branch that logs and 2xxs rather than erroring.

## 4. Receiver rules

- **Respond fast, process later.** Acknowledge with 2xx and hand off to a queue.
- **Assume redelivery and out-of-order arrival.** viagogo publishes no retry policy and no delivery
  ordering guarantee, so make your handler idempotent on your side — key on the embedded resource id
  plus `topic` plus `action`.
- **Never predict barcode allocation.** The docs are explicit that attempting to will cause double
  sales. Always use the barcodes provided in the API and the webhooks.
- **Reconcile anyway.** Pair the push channel with a periodic `Sales_GetSalesRecentUpdates` and
  `SellerListings_GetSellerListingsRecentUpdates` sweep so a silently dropped delivery cannot leave
  you inconsistent.

## 5. Manage the subscription

`Webhooks_Get` (`GET /webhooks`) lists yours; `Webhooks_Get2` (`GET /webhooks/{webhookId}`) reads
one; `Webhooks_Patch` (`PATCH /webhooks/{webhookId}`) updates the url, name, topics or
`authorization_header`; `Webhooks_Delete` (`DELETE /webhooks/{webhookId}`) removes it (`204`).

## Cross-references

- `asyncapi/viagogo-webhooks.yml` — the full topic catalog and subscription model
- `skills/viagogo-fulfill-a-sale.md` — what to do once `SalesTopic` fires
