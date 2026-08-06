---
name: viagogo-fulfill-a-sale
description: Work a viagogo sale from notification to payout — read the sale, drive it through its documented state machine, deliver the tickets by the method the state demands, and confirm the seller payment.
api: viagogo Sales API
generated: '2026-08-05'
method: generated
source: openapi/viagogo-sales.json, https://developer.viagogo.net/docs/overview/transaction-states
operations:
- Sales_GetSalesRecentUpdates
- Sales_GetSales
- Sales_Get
- Sales_Patch
- Sales_Delete
- TicketHolders_GetSaleTicketHolderDetails
- ETicket_PostSaleETickets
- ETicket_UploadSaleETicket
- ETicket_UploadSaleTransfer
- Sales_UploadTransferStatusProof
- Shipments_PutOrGetSaleShipmentLabel
- Shipments_GetSaleShippingLabel
- Shipments_PatchSaleShipment
- Payments_GetPayments
- Payments_GetNextPayment
---

# Fulfill a viagogo sale

Base URL `https://api.viagogo.net/v2`, sandbox `https://sandbox.api.viagogo.net/v2`.
Responses are `application/hal+json`.

Scopes: reads need `read:user` + `read:sales`; every write needs `write:user` + `write:sales`.
Payments additionally need `read:payment`.

## 1. Learn that a sale happened

Two ways, and you want both:

- **Push** — subscribe a webhook to the sales topics. `SalesTopic` fires when something affects a
  sale; `ProvisionalSaleTopic` fires on a provisional sale (the normal `SalesTopic` confirms it — to
  reject, ignore the provisional and reject the sale when it arrives normally);
  `CancelProvisionalSaleTopic` fires when a provisional sale is cancelled. See
  `asyncapi/viagogo-webhooks.yml`.
- **Pull** — `Sales_GetSalesRecentUpdates` (`GET /sales/recentupdates`) as a reconciliation sweep,
  or `Sales_GetSales` (`GET /sales`) with `updated_since` and `min_resource_version`.

Barcodes behave differently in the two channels: they are embedded **within the sale** on
`Sales_Get`, but **within the notification object itself** on the webhook.

## 2. Read the sale and branch on `status`

`Sales_Get` (`GET /sales/{saleId}`).

The `status` field drives everything. Do not invent your own workflow — viagogo publishes the state
machine and the action each state requires.

**Initial states**

| `status` | What it means | Your move |
|---|---|---|
| `UnderReview` | The buyer is being screened by the fraud process | Hold the tickets in your POS |
| `ConfirmSale` | The order is awaiting confirmation | Auto-confirm if the tickets are available |
| `Issue` | There is a problem with the order | Contact customer service |

**Delivery states — each one names a different mechanism**

| `status` | Your move |
|---|---|
| `UploadETickets` | `ETicket_PostSaleETickets` (`POST /sales/{saleId}/etickets`) |
| `UploadTicketLinks` | `Sales_Patch` with `eticket_urls` |
| `TransferTDT` | **First** `TicketHolders_GetSaleTicketHolderDetails` and transfer the tickets to that holder. **Then** record success — `Sales_Patch` with `eticket_urls`, or `Sales_Patch` with `transfer_confirmation_number`, or `ETicket_UploadSaleTransfer` / `Sales_UploadTransferStatusProof` |
| `UploadQRCodes` | `ETicket_UploadSaleETicket` (`POST /sales/{saleId}/eticketuploads`) |
| `UploadBarcodes` | `Sales_Patch` with `barcodes` |
| `ReenterBarcodes` | The barcodes first allocated could not be re-issued — `Sales_Patch` with new `barcodes`. The order may pass briefly back through `UploadBarcodes` |
| `PrintShippingLabels` | `Shipments_PutOrGetSaleShipmentLabel` (`PUT /sales/{saleId}/shipments`), then `Shipments_GetSaleShippingLabel` for the file |
| `WaitForCourierPickup` | Hand tickets + airbill to the courier |
| `EnterTrackingNumbers` | `Shipments_PatchSaleShipment` with the tracking number |
| `DropOffTickets` | Hand-deliver to the buyer |

**Terminal / no-action states**: `OnTheWay`, `ProcessingTransfer`, `WaitingForBuyerPickup`,
`WaitForShippingDetails`, `WaitUntil2DaysBeforeTheEvent`, `GetPaid`, `Complete`, `Cancelled`.

Treat any `status` you do not recognise as no-action-and-alert rather than as an error — viagogo
states other states are possible.

## 3. Reject a sale you cannot fill

`Sales_Delete` (`DELETE /sales/{saleId}`) rejects the sale. Returns `204`. Needs `write:sales`.

## 4. Watch for asynchronous failures you did not cause

`SaleUpdatesTopic` fires when something changes on a sale you did not initiate. Filter on `action`:
`FailedBarcodeValidation` means the barcodes you uploaded failed validation and must be re-uploaded.
`ReTransferTicketTopic` means the buyer never received the ticket and it must be re-transferred from
the third-party provider.

## 5. Get paid

`Payments_GetPayments` (`GET /payments`) lists seller payments; `Payments_Get` reads one;
`Payments_GetNextPayment` (`GET /payments/next`) previews the next payout without side effects. A
`Payment` carries `payment_amount`, `proceeds`, `charges` and `credits` as `Money`, with the
contributing sales embedded and each charge tied back by `sale_id`.

## Retry discipline

There is no idempotency key on this API. `Sales_Patch` is a PATCH and is naturally re-appliable, but
the upload operations (`POST /sales/{saleId}/etickets`, `/eticketuploads`, `/transferuploads`) are
not. On a timeout, read back with `ETicket_GetSaleETickets` or `ETicket_GetSaleETicketUploads`
before re-posting, or you will duplicate the upload.

## Cross-references

- `conventions/viagogo-conventions.yml` · `errors/viagogo-problem-types.yml`
- `asyncapi/viagogo-webhooks.yml` · `data-model/viagogo-data-model.yml`
