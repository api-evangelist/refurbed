---
name: Manage refurbed offers and market prices
description: Create and update a merchant offer on an instance, then set or adjust per-market prices on the refurbed marketplace.
api: openapi/refurbed-merchant-api-openapi.json
operations: [ListMarkets, ListCurrencies, GetInstance, ListShippingProfiles, CreateOffer, UpdateOffer, ListOffers, CreateMarketOffer, UpdateMarketOffer, ListMarketOffersByOffer]
---

# Manage refurbed offers and market prices

Use the refurbed Merchant API to list a product instance, publish an offer for
it, and manage its market-specific prices.

## Auth
All calls use `Authorization: Plain <token>` (see `authentication/refurbed-authentication.yml`).
Host `api.refurbed.com`. Monetary values are strings with up to 2 decimals (e.g. `"123.45"`).

## Steps
1. Resolve the instance you want to sell with `GetInstance` (by refurbed instance id or GTIN).
2. Look up valid targets: `ListMarkets` (which markets you can sell into), `ListCurrencies` (allowed currencies), `ListShippingProfiles` (shipping profile ids).
3. Create the market-independent offer with `CreateOffer` — set `instance_id`, `grading`, `warranty`, `stock`, `shipping_profile_id`, and reference prices. The offer returns default market prices automatically.
4. Adjust offer configuration or stock later with `UpdateOffer`; list your offers with `ListOffers`.
5. To set a price manually for a specific market (instead of the auto-calculated one), use `CreateMarketOffer` / `UpdateMarketOffer` on the `MarketOfferService`.
6. Inspect the per-market prices and BuyBox state for an offer with `ListMarketOffersByOffer`.

## Rules
- Prefer batch operations (`BatchCreateOffers`, `BatchUpdateOffers`) when acting on many offers to respect rate limits (see `rate-limits/refurbed-rate-limits.yml`).
- Errors come back as a gRPC status (`errors/refurbed-error-codes.yml`): `INVALID_ARGUMENT` for bad fields, `RESOURCE_EXHAUSTED` when rate-limited.
- List calls are cursor-paginated: pass `pagination.limit` + `starting_after`, and keep going while `has_more` is true.
