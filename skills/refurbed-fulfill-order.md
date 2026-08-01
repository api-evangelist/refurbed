---
name: Fulfill a refurbed order
description: List released refurbed orders, inspect their items, advance item state, and generate a shipping label.
api: openapi/refurbed-merchant-api-openapi.json
operations: [ListOrders, GetOrder, ListOrderItemsByOrder, GetOrderItem, UpdateOrderItemState, BatchUpdateOrderItemsState, CreateShippingLabel, ListShippingLabels, GetOrderInvoice]
---

# Fulfill a refurbed order

Use the refurbed Merchant API to process orders that have been released to you
for fulfillment.

## Auth
All calls use `Authorization: Plain <token>` (see `authentication/refurbed-authentication.yml`). Host `api.refurbed.com`.

## Steps
1. Poll `ListOrders` for orders released to you; use its filters to select what you need. Page with `pagination.limit` + `starting_after` while `has_more` is true.
2. Load a specific order with `GetOrder`, and its line items with `ListOrderItemsByOrder` (or `GetOrderItem` for one item).
3. Advance an item through its lifecycle with `UpdateOrderItemState` (or `BatchUpdateOrderItemsState` for many items at once).
4. Generate a shipping label with `CreateShippingLabel`; retrieve existing labels with `ListShippingLabels`.
5. Fetch the order invoice with `GetOrderInvoice` when you need the document.

## Rules
- Item state transitions are validated server-side; an invalid transition returns `FAILED_PRECONDITION` (`errors/refurbed-error-codes.yml`).
- Use the batch variants to stay within rate limits (`rate-limits/refurbed-rate-limits.yml`).
- To refund, use the refund skill (`refurbed-refund-order-item.md`) — do the dry-run first.
