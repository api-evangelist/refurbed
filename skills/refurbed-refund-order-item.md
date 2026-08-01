---
name: Refund a refurbed order item
description: Preview a refund amount with the dry-run calculation, then issue the refund for a single order item or a whole order.
api: openapi/refurbed-merchant-api-openapi.json
operations: [GetOrderItem, CalculateRefundOrderItem, RefundOrderItem, CalculateRefundOrder, RefundOrder]
---

# Refund a refurbed order item

Use the refurbed Merchant API to refund an order item (or an entire order),
previewing the amount before committing.

## Auth
All calls use `Authorization: Plain <token>` (see `authentication/refurbed-authentication.yml`). Host `api.refurbed.com`. Amounts are strings with up to 2 decimals.

## Steps
1. Load the item with `GetOrderItem` to confirm its current state and charged totals.
2. Dry-run the refund with `CalculateRefundOrderItem` to see the exact amount that will be refunded — this does NOT move money.
3. If correct, commit with `RefundOrderItem`.
4. To refund every item of an order at once, use `CalculateRefundOrder` (dry-run) then `RefundOrder` (commit) on the `OrderService`.

## Rules
- Always call the `Calculate*` dry-run before the committing call — the calculate methods exist precisely so you can verify amounts safely.
- Refund errors surface as gRPC status (`errors/refurbed-error-codes.yml`); `FAILED_PRECONDITION` means the item is not in a refundable state.
- There is no idempotency-key contract (`conventions/refurbed-conventions.yml`) — do not blindly retry a `RefundOrderItem`; re-check state with `GetOrderItem` first.
