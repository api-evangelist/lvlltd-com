---
generated: '2026-09-19'
method: generated
name: ESP evaluate, settle and prove a skill purchase
description: Find a sealed skill pack on lvlltd.com, evaluate it for free, pay the live x402 challenge in USDC on Base, unlock the pack idempotently, and confirm the unlock on the public proof ledger.
api: openapi/lvlltd-com-openapi.yml
operations: [getReadiness, getEspProtocol, searchCatalog, getSkillOutline, getSkillSample, getPaymentChallenge, unlockSealedPack, getProofLedger]
source: >-
  Grounded in openapi/lvlltd-com-openapi.yml (method + path verified; the published spec carries no
  operationIds, so the ids above are the ones overlays/lvlltd-com-openapi-overlay.yaml assigns),
  conventions/lvlltd-com-conventions.yml, errors/lvlltd-com-problem-types.yml and the provider's
  https://lvlltd.com/docs/REFERENCE.md.
---

# ESP evaluate, settle and prove a skill purchase

Buy one sealed skill pack from LVL LTD the way the provider's own ESP protocol describes it: evaluate for free, settle exactly once, and trust only the public ledger.

## Auth
- None for discovery and evaluation. There is no API key. Payment proof is the credential: `X-PAYMENT: {"txHash":"0x…","skill":"<id>"}` on the unlock POST, or a signed `PAYMENT-SIGNATURE` for the Coinbase CDP facilitator path. See `authentication/lvlltd-com-authentication.yml`.

## Idempotency and reversibility
- The unlock is keyed on `(txHash, skill)`: re-POSTing the same proof returns the same pack and never double-charges. There is no `Idempotency-Key` header. See `conventions/lvlltd-com-conventions.yml`.
- USDC transfers are final. Refunds exist only for failed delivery, duplicate settle, overpayment or operator error, decided within 3 business days of the confirmed tx — read the free outline before paying.

## Steps
1. **Check the rails** — `getReadiness` (`GET /api/ready`). Proceed only when `ready: true`; `ready: false` means the challenge or sealed-pack rails are broken and you must not buy.
2. **Read the protocol root** — `getEspProtocol` (`GET /api/esp`) to confirm the invariants (`free_eval_always`, `settlement_atomic`, `proof_public_only`) and the current treasury.
3. **Find a skill** — `searchCatalog` (`GET /api/catalog?q=…&max_price=…&limit=…`). Prefer rows whose tags include `depth:deep`; skip `template-outline` / `depth:outline_only` rows unless a human has reviewed them.
4. **Evaluate for free** — `getSkillOutline` (`GET /skills/{id}/outline.json`) and `getSkillSample` (`GET /skills/{id}/sample.md`). If the outline is empty or boilerplate, stop here — outlines exist so you can refuse before paying.
5. **Get the live quote** — `getPaymentChallenge` (`GET /api/pay?skill={id}`). Expect **HTTP 402**; it is the quote, not an error. Read `maxAmountRequired` (atomic USDC, 6 decimals), `payTo`, `network` (`eip155:8453`) and `assetContract` from the body or `accepts[0]` — never from cached docs.
6. **Settle** — transfer exactly `maxAmountRequired` USDC on Base (chain 8453) to `payTo`. Keep the transaction hash. The wallet needs a little ETH on Base for gas; USDC alone cannot pay it.
7. **Unlock** — `unlockSealedPack` (`POST /api/pay`) with header `X-PAYMENT: {"txHash":"0x…","skill":"{id}"}` and the same JSON body. Success is `ok: true` plus `sealed_pack.files` (a map of path → text). Persist the files and the txHash.
8. **Prove** — `getProofLedger` (`GET /api/proof?skill={id}`). Only a row here is a confirmed unlock. Do not report success from the unlock response alone, and never invent volume from anything but this ledger.

## Errors
- `402 PAYMENT_VERIFICATION_FAILED` with `retry: true` — wait 2–5 s after broadcast and retry with backoff (2 s, 4 s, 8 s, ≤30 s); confirm on BaseScan that the transfer is USDC on chain 8453 to the challenge `payTo` for the exact amount.
- `402 PROOF_REQUIRED` — you POSTed without a txHash.
- Paid but no files — `POST /api/recover` with the same `{txHash, skill}` (idempotent; moves no funds). Then, if still undelivered, the written refund policy applies. Full table in `errors/lvlltd-com-problem-types.yml`.

## Notes
- Start with the $0.05 canary `agent-x402-first-buy` to prove the loop before a larger buy; it is a real micropayment, not a sandbox.
- Pay only the treasury in the live challenge / `contracts.json`; two legacy addresses are banned there, including the one still advertised by the swarm.lvlltd.com agent card.
- The same loop is available as MCP tools (`esp_evaluate` → `quote_skill` → `purchase_skill` → `verify_unlock`) and as the A2A skills `quote_price` → `initiate_x402_purchase` → `verify_unlock`; all three route through `POST /api/pay`.
