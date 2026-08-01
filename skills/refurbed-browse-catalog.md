---
name: Browse the refurbed catalog (Affiliate Partner API)
description: As an affiliate, list markets and products and read instance BuyBox pricing and purchase-redirect links from the refurbed marketplace.
api: openapi/refurbed-affiliate-api-openapi.json
operations: [ListMarkets, GetMarket, ListProducts, GetProduct, ListInstancesByProduct, GetInstance]
---

# Browse the refurbed catalog (Affiliate Partner API)

Use the refurbed Affiliate Partner API to read the public marketplace catalog
and surface BuyBox offers to your audience.

## Auth
All calls use `Authorization: Plain <token>` (see `authentication/refurbed-authentication.yml`). Host `api.refurbed.com`.

## Steps
1. Discover markets with `ListMarkets` — filter with `is_site` to get only site markets; you must specify a site market when reading products/instances.
2. List products with `ListProducts` (rich filters) or load one with `GetProduct`. Each product carries BuyBox data for the best current offer.
3. Drill into variants with `ListInstancesByProduct`, or resolve a single variant with `GetInstance` (by GTIN or refurbed instance id).
4. Read each instance's BuyBox for price, grading/warranty conditions, and the redirect link to send a customer to complete the purchase.

## Rules
- Always scope catalog reads to a site market obtained from `ListMarkets`.
- List calls are cursor-paginated (`pagination.limit` + `starting_after`, continue while `has_more`).
- Errors are gRPC status objects (`errors/refurbed-error-codes.yml`).
