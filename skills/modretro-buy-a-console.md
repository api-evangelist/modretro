---
name: Buy a ModRetro console on a buyer's behalf
description: Take a ModRetro purchase from search through cart and checkout to completion over the UCP/MCP endpoint, stopping for explicit human approval before payment.
api: mcp/modretro-mcp.yml
endpoint: https://modretro.com/api/ucp/mcp
operations: [search_catalog, get_product, create_cart, update_cart, create_checkout, update_checkout, complete_checkout, cancel_checkout, cancel_cart, get_order]
generated: '2026-08-26'
method: generated
source: mcp/modretro-mcp-tools.json
---

# Buy a ModRetro console

ModRetro implements the Universal Commerce Protocol (UCP `2026-04-08`) over MCP so an agent can
transact directly. **Read the hard stop below before you write any code against it.**

## Hard stop: payment requires a human

ModRetro states this in two first-party documents, `robots.txt` and `/agents.md`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically —
> no scripted form fills, browser automation, or end-to-end agent flows that finalize payment
> without an explicit, contemporaneous human approval step.

`complete_checkout` is the line. Everything before it you may do autonomously. Calling it is the
buyer's decision, taken at the moment of payment, not a decision you inferred earlier in the
conversation. If you cannot get that approval in the moment, ModRetro asks you to route the purchase
through the Shop skill (`https://shop.app/SKILL.md`) instead, which enforces the same invariant via
Shop Pay.

## Endpoint and preconditions

`POST https://modretro.com/api/ucp/mcp` — JSON-RPC 2.0.

- Every call needs `meta.ucp-agent.profile`, a fetchable agent-profile URI. Missing or unresolvable
  gives HTTP 422 / `-32001 UCP discovery failed`.
- Checkout and order tools additionally need a **Shopify agent JWT**. Without one you get HTTP 403 /
  `-32000 AuthenticationRequired`. Obtain it per
  <https://shopify.dev/docs/agents/get-started/authentication>.

## Steps

1. **Find the product** — `search_catalog`, then `get_product` for exact pricing and real-time
   availability on the specific variant. Confirm the variant with the buyer before you build a cart.
2. **Create the cart** — `create_cart` with `cart.line_items[]` (`item.id` is the
   `gid://shopify/ProductVariant/...`, plus `quantity`), `cart.buyer.email`, and
   `cart.context.address_country` / `cart.context.currency`.
3. **Adjust** — `update_cart` for quantity, buyer and fulfillment changes. Read back with `get_cart`.
   Only apply `cart.discounts.codes` if the buyer gave you a code — a new submission **replaces** the
   previous set, it does not append.
4. **Open a checkout** — `create_checkout`, passing `checkout.cart_id` to carry the cart over.
5. **Set shipping** — `update_checkout` with `checkout.fulfillment.methods[]`, the destination
   address and the selected option. This store ships to a **single destination only**
   (`allows_multi_destination.shipping: false`) and the only method combination it accepts is
   `["shipping"]` — do not attempt a split shipment or a pickup/ship mix.
6. **Present the total** — `get_checkout` returns line items, totals, discounts and taxes. Convert
   minor units to major units before showing any figure: `{"amount": 2500, "currency": "USD"}` is
   **$25.00**.
7. **Ask the human. Wait.** Show the exact total, the shipping destination and the payment
   instrument. Get an explicit yes.
8. **Complete** — `complete_checkout`. This is the **only** tool that requires
   `meta.idempotency-key`, and it is required, not optional. Generate one key per buyer decision and
   reuse that same key on any retry. Never mint a fresh key to "try again" — that is how a buyer ends
   up with two consoles. The result carries the order ID and Thank You Page URL, or the errors
   encountered.
9. **Confirm** — `get_order` with the returned order ID.

## Backing out

| Stage | Reversal | Window |
|---|---|---|
| Cart open | `cancel_cart` | not published |
| Checkout open, not completed | `cancel_checkout` | not published |
| After `complete_checkout` | **none in this API** | — |

After completion there is no refund, void or cancel tool. Reversal leaves the API entirely and
becomes a merchandise return: ModRetro accepts returns only with the **original packaging unopened**
and only when initiated **within 30 days of the order's arrival date**
(<https://modretro.com/policies/refund-policy>). Tell the buyer that before step 8, not after.
Shipping addresses **cannot** be edited once the order is submitted, and packages cannot be rerouted
in transit — so confirm the destination in step 5 as if it were final, because it is.

## Payment handlers this store accepts

From `https://modretro.com/.well-known/ucp`: Google Pay (`com.google.pay`, VISA / MASTERCARD / AMEX /
DISCOVER), Shopify card (`dev.shopify.card`, incl. Diners Club), and Shop Pay
(`dev.shopify.shop_pay`).
