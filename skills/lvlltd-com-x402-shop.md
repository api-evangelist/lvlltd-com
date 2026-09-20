---
name: lvlltd-x402-shop
description: Buy sealed AI skill packs on lvlltd.com with x402 (Base USDC). Free outline first. No API key. No hosted keys.
metadata:
  origin: lvlltd.com
  protocol: x402
  network: eip155:8453
  canary: agent-x402-first-buy
---

# LVL LTD x402 shop skill

Use this when an agent needs a priced capability from https://lvlltd.com.

## Hard rules

1. Always evaluate free first. Never invent unlock counts. Success is only `GET https://lvlltd.com/api/proof`.
2. Prefer verified / deep packs. Default catalog search hides outline-only stubs.
3. Do not ask the human for an API key. Pay USDC on Base to the `payTo` in the 402.
4. This merchant never holds keys. Sign `wallet_next_action` locally or send an ERC-20 transfer then POST.

## Flow

1. GET https://lvlltd.com/api/ready
2. GET https://lvlltd.com/api/shop  (or MCP search_skills)
3. GET the winning skill outline, e.g. https://lvlltd.com/skills/agent-x402-first-buy/outline.json
4. GET https://lvlltd.com/api/pay?skill=agent-x402-first-buy  → HTTP 402
5. Preferred: sign `wallet_next_action.sign.typed_data` (`eth_signTypedData_v4`) then POST `PAYMENT-SIGNATURE`
6. Fallback: transfer `maxAmountRequired` USDC on Base to `payTo`, then POST `X-PAYMENT: {"txHash":"0x…","skill":"agent-x402-first-buy"}`
7. GET https://lvlltd.com/api/proof?skill=agent-x402-first-buy  — only success signal
8. If paid but no files: POST https://lvlltd.com/api/recover {"txHash":"0x…","skill":"agent-x402-first-buy"}

Find-by-job: https://lvlltd.com/api/find?need=x402+canary

## Machine entry points

- Shop: https://lvlltd.com/api/shop
- Install: https://lvlltd.com/install/
- Networks (Base live only): https://lvlltd.com/api/networks
- Bazaar status (honest): https://lvlltd.com/api/bazaar
- Find-by-job: https://lvlltd.com/api/find?need=x402+canary
- Catalog search: https://lvlltd.com/api/catalog
- MCP: https://lvlltd.com/api/mcp  (alias https://lvlltd.com/mcp)
- Manifest: https://lvlltd.com/.well-known/x402
- OpenAPI: https://lvlltd.com/openapi.json
- Canary $0.05: skill id `agent-x402-first-buy`
- Tripwire $0.99: `aie-premium-access-token`
- Flagship live list cap $0.99: `lvl-agent-revenue-os-pack` (do not advertise $49 while 402 is 990000)
