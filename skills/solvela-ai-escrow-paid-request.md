---
name: solvela-ai-escrow-paid-request
description: Pay for a Solvela completion through the trustless on-chain escrow scheme so the gateway can only claim what it actually used, the remainder returns automatically, and an undelivered request is refundable after the stated window.
api: openapi/solvela-ai-openapi.json
operations:
  - createChatCompletion
  - getReceipt
method: generated
generated: '2026-09-19'
grounding: operationIds exist verbatim in openapi/solvela-ai-openapi.json. The escrow parameters, PDA derivation, claim and refund behaviour and the 300 s window are quoted from the provider's escrow documentation, the x402Payment securityScheme description and the live 402 challenge of 2026-09-19. GET /v1/escrow/config and POST /v1/escrow/settle are named by path because the provider documents them but does not declare them in the OpenAPI.
---

# Pay through escrow instead of paying up front

Choose escrow when you want protection against paying for something the gateway then fails to deliver, or when you want to be billed the actual cost rather than the quoted ceiling. It costs one extra on-chain transaction per request (deposit, then the gateway's claim).

## 1. Get the quote and confirm escrow is offered

`POST /v1/chat/completions` (`createChatCompletion`) without `PAYMENT-SIGNATURE`. The 402 `accepts[]` on the hosted gateway lists two entries; take the one with `scheme: "escrow"`. It carries the same `amount`, `asset`, `pay_to`, `max_timeout_seconds` (300) plus `escrow_program_id` (`9neDHouXgEgHZDde5SpmqqEZ9Uv35hFcjtFEPxomtHLU` on 2026-09-19). If no escrow entry is present, the deployment has escrow disabled — fall back to the exact scheme.

`GET /v1/escrow/config` (documented, not in the OpenAPI) returns `escrow_program_id`, `current_slot`, `network`, `usdc_mint` and `provider_wallet`; use `current_slot` when you need an expiry slot.

## 2. Build the deposit

Generate a 32-byte `service_id` per request (base64). The vault PDA is derived from seeds `["escrow", <your pubkey bytes>, <service_id bytes>]` under the escrow program; it holds a USDC Associated Token Account owned by the program, so neither you nor the gateway can withdraw unilaterally. Sign a deposit of at least the quoted `amount` with an expiry. The `solvela-escrow-tx` crate and the TypeScript/Python/Go SDKs (0.2.3+) build this transaction; the MCP server's `deposit_escrow` tool does it with client-side caps of $5 per call and $20 per session by default.

## 3. Send the request with the escrow payload

Resend the byte-identical request with `PAYMENT-SIGNATURE` carrying `{ x402_version: 2, resource, accepted: <the escrow accepts[] entry>, payload: { deposit_tx: <base64 signed deposit>, service_id: <base64>, agent_pubkey: <your base58 pubkey> } }`. The gateway verifies the deposit is confirmed and covers the cost, then proxies to the model and returns the completion with `X-Solvela-Receipt`.

## 4. Let the claim happen — or trigger it

After the response the gateway fires a claim: in one transaction the program pays the gateway the **actual** token cost, refunds `deposit − actual` to your token account, and closes the vault and escrow accounts (rent back to you). A request that hit the semantic cache is claimed for less than quoted. For streaming responses you may call `POST /v1/escrow/settle` (documented; not in the OpenAPI) with `service_id`, `agent_pubkey`, `model`, `status: "completed"` and the actual token counts so the claim fires immediately instead of at the timeout; the program rejects over-claims on-chain, and with PostgreSQL configured the claim queue dedupes by `(service_id, agent)`.

## 5. Reclaim if the gateway never claims

This is the reversal path and it has a stated window: "If the gateway fails to claim within `max_timeout_seconds`, the agent can invoke the `refund` instruction on the escrow program to reclaim their deposit. The refund path is enforced on-chain by the Anchor program — no trust in the gateway is required." Invoke `refund` against the same PDA after the 300 s window. Nothing is reversible once the claim has landed — a delivered completion cannot be un-delivered.

## 6. Verify with the receipt

`GET /v1/receipts/{receipt_id}` (`getReceipt`) shows `payment_scheme: "escrow"`, `amount_paid_atomic` (what was actually claimed) against `cost_breakdown.total_atomic` (what was quoted), and the `tx_signature`.

## Cautions the provider publishes

- The escrow program is "trusted-but-upgradeable": its upgrade authority is a single-sig key held by the operator, with a Squads multisig migration planned (SECURITY.md).
- Replay rules still apply: a deposit transaction is accepted once; a new request needs a new `service_id` and deposit.
- The same 60/min per-wallet rate limit and the 402 error classes from `solvela-ai-pay-per-call-chat` apply.
