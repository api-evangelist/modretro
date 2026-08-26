---
name: Look up ModRetro products, prices and availability
description: Search and read ModRetro's Chromatic and M64 catalog over its UCP/MCP endpoint, and convert prices correctly before quoting them to a buyer.
api: mcp/modretro-mcp.yml
endpoint: https://modretro.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product]
generated: '2026-08-26'
method: generated
source: mcp/modretro-mcp-tools.json
---

# Look up ModRetro products

ModRetro sells retro gaming hardware — the **Chromatic** handheld and the **M64** console — plus
cartridges and accessories. The catalog is readable over a single MCP endpoint. No purchase, no
account, no order data is involved in this skill.

## Endpoint

`POST https://modretro.com/api/ucp/mcp`, `Content-Type: application/json`,
`Accept: application/json, text/event-stream`. JSON-RPC 2.0.

## Before you call anything

Every `tools/call` requires `meta.ucp-agent.profile` — a **fetchable URI** describing your agent.
The server retrieves it. Omit it, or point it at something unreachable, and you get
HTTP 422 with JSON-RPC `-32001 UCP discovery failed` (`data.code: invalid_profile_url`).

Catalog tools need nothing more. They do **not** require the Shopify agent JWT that the order and
checkout tools require.

## Steps

1. **Search** — call `search_catalog` with `catalog.query` (natural language), `catalog.filters`, or
   both. At least one of query or filters is mandatory. Pass `catalog.context.address_country`
   (ISO 3166-1 alpha-2) and `catalog.context.currency` (ISO 4217) so pricing and availability come
   back correct for the buyer.
2. **Page** — results are cursor-paginated. `catalog.pagination.limit` defaults to 10 (minimum 1).
   To get more, send the `pagination.cursor` from the previous response. Do not page speculatively;
   page when the buyer asks for more.
3. **Resolve in bulk** — when you already hold identifiers, call `lookup_catalog` instead of looping
   `get_product`. It takes `gid://shopify/Product/...` and `gid://shopify/ProductVariant/...` ids,
   **maximum 10 per request**, and groups results by product. Each variant carries an `inputs` array
   showing which request id resolved to it and whether the match was `exact` or `featured`.
4. **Read one product** — call `get_product` for the detail page: full variant set, exact pricing,
   real-time availability. Use `selected` and `preferences` to narrow variants interactively.

## The one rule that will bite you

**Prices are integers in ISO 4217 minor units, paired with a currency code.**
`{"amount": 2500, "currency": "USD"}` is **$25.00**, not $2,500. Divide by 100 for two-decimal
currencies before you ever say a number to a buyer. Zero-decimal currencies such as JPY are already
whole units — do not divide those.

## Errors and backoff

- `-32001` / HTTP 422 — your `meta.ucp-agent.profile` is missing or unfetchable. Fix the profile.
- HTTP 429 — the endpoint is rate-limited per IP. Back off; ModRetro publishes no budget or window,
  so use exponential backoff rather than assuming a ceiling.

## When you do not need MCP at all

For plain catalog reads the storefront also serves JSON with no protocol overhead:
`GET https://modretro.com/products.json`, `GET https://modretro.com/collections/{handle}/products.json`,
`GET https://modretro.com/products/{handle}.json`. Prices there are decimal strings, not minor units.
Do not mix the two conventions in one answer.
