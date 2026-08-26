---
name: Check a ModRetro order and explain what can still be changed
description: Read a ModRetro order over MCP and answer accurately about shipping, address changes, rerouting and returns using ModRetro's published policies.
api: mcp/modretro-mcp.yml
endpoint: https://modretro.com/api/ucp/mcp
operations: [get_order]
generated: '2026-08-26'
method: generated
source: mcp/modretro-mcp-tools.json
---

# Check a ModRetro order

## Read the order

`POST https://modretro.com/api/ucp/mcp`, tool `get_order`, with the
`gid://shopify/Order/...` identifier.

This tool is **authenticated**. It requires both `meta.ucp-agent.profile` and a valid Shopify agent
JWT. Calling it without a token returns HTTP 403 and JSON-RPC `-32000 AuthenticationRequired` — the
message itself points at <https://shopify.dev/docs/agents/get-started/authentication>. A customer
signing in for themselves goes through ModRetro's own OIDC endpoints
(`https://modretro.com/.well-known/openid-configuration`, authorization code + PKCE S256) with the
`customer-account-api:full` scope.

Prices in the response are integers in ISO 4217 minor units. Divide by 100 for USD/EUR before
quoting; JPY is already whole units.

## Answer these correctly — ModRetro publishes the answers and they are all "no"

Do not soften these. Getting them wrong sets up a buyer for a bad surprise.

- **Change the shipping address after ordering?** No. Once the order is submitted the shipping
  address cannot be edited.
- **Reroute a package in transit?** No. It goes to the address entered at checkout.
- **Partial shipment?** Expected behaviour, not an error. ModRetro partially ships based on stock so
  the buyer is not waiting on the slowest item.
- **International duties and fees?** For the regions marked on the product pages (incl. EU and UK)
  the listed price already includes taxes, duties and fees.
- **Return it?** Only with the **original packaging unopened**, and only if initiated **within 30
  days of the arrival date** — <https://modretro.com/policies/refund-policy>.
- **Marked delivered but missing?** ModRetro's guidance is that carriers sometimes scan early; wait
  before escalating.

## Where to send the buyer

- Support centre: <https://support.modretro.com>
- Community forum: <https://forums.modretro.com/>
- Firmware updates and known issues:
  <https://support.modretro.com/en_us/chromatic-firmware-updater-ryhoYnzCx.md>

There is no ModRetro status page. If the endpoint is failing, that is all you can say.
