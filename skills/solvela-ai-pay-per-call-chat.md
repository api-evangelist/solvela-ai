---
name: solvela-ai-pay-per-call-chat
description: Get a chat completion from any of Solvela's 44 models by paying the exact quoted USDC on Solana through the x402 402-then-retry handshake, then keep the receipt — with the quote, replay, retry and rate-limit rules the contract states.
api: openapi/solvela-ai-openapi.json
operations:
  - listModels
  - createChatCompletion
  - getReceipt
method: generated
generated: '2026-09-19'
grounding: Every operationId above exists verbatim in openapi/solvela-ai-openapi.json; header names, field names, statuses and limits are quoted from the contract, the provider's x402/errors/rate-limits docs and the 402 challenge observed live on 2026-09-19.
---

# Pay per call for a chat completion (exact scheme)

No account, no API key. The wallet that signs the payment is the identity, and the payment is the authorization. Every paid call is real USDC on Solana mainnet; there is no test network on the hosted gateway.

## 1. Pick a model and know the price before you ask

- `GET /v1/models` (`listModels`) — free, no rate limit. Each entry carries `pricing.input_per_million` / `output_per_million` in USDC, `context_window` and `capabilities` (`streaming`, `tools`, `vision`, `reasoning`). Ids are provider-prefixed: `openai/gpt-4o-mini`, `anthropic/claude-sonnet-4-5-20250929`. Aliases (`sonnet`, `gpt5`, `haiku`, `opus`, `gemini`, `flash`, `grok`, `deepseek`) and routing profiles (`auto`, `eco`, `premium`, `free`) are also valid `model` values.
- Send `image_url` content only to a model with `capabilities.vision: true`; otherwise the gateway answers **415** `unsupported_media_type`.

## 2. Ask once without paying to get the quote

`POST /v1/chat/completions` (`createChatCompletion`) with the request body and **no** `PAYMENT-SIGNATURE` header. The gateway answers **402** whose body is the x402 `PaymentRequired` object at the top level (not inside `error`): `accepts[]` with `scheme`, `network` (`solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`), `amount` (atomic USDC, 6 decimals, as a string), `asset` (the USDC mint), `pay_to` and `max_timeout_seconds` (300); and `cost_breakdown` with `total`, `currency` and `fee_percent`. The same challenge is in the `PAYMENT-REQUIRED` response header, base64-encoded, in canonical camelCase.

The quote is priced at the model's **full completion-token ceiling**, so a one-word prompt to Claude Sonnet 4.5 was quoted 0.122883 USDC on 2026-09-19. Set `max_tokens` to what you actually need and the quote drops accordingly. An unknown model id returns **404** `model_not_found` here, before any payment.

## 3. Sign exactly what was quoted

Build a Solana `TransferChecked` of at least `accepts[].amount` of `asset` to `pay_to`, sign it with your wallet, and wrap it as `{ x402_version: 2, resource: {url, method}, accepted: <the exact accepts[] entry you chose>, payload: { transaction: <base64 signed versioned tx> } }`. Read `pay_to`, `asset` and `network` from **this** 402 every time — a mismatch is rejected with **400** (`"Payment recipient does not match..."`, `"Payment asset is unsupported..."`, `"Payment network is unsupported..."`). The SDKs (`@solvela/sdk`, `solvela-sdk`, Go `sdks/go`, `solvela-client`) do this step for you.

## 4. Retry the byte-identical request with the payment

Resend the same `POST /v1/chat/completions` with `PAYMENT-SIGNATURE: <base64 of the payload JSON>` within `max_timeout_seconds`. A **200** is an OpenAI-shaped `ChatCompletionResponse` (or an SSE stream when `stream: true`) and carries `X-Solvela-Receipt: <uuid>` on paid responses. Free ($0) models never need this step.

## 5. Handle the failure classes correctly — this is where money is

- **402** with `error.type: invalid_payment` — verification failed. If the message says the transaction has already been used, you replayed a signature: build a **new** transaction with a fresh blockhash; never resubmit. If it says the transaction could not be confirmed, wait a few seconds and retry the same signed transaction once.
- **429** — honor `retry-after` (seconds). The limit is 60 requests per 60 s per payer wallet; `x-ratelimit-reset` is the window length, not a timestamp.
- **503** `upstream_unavailable` — no provider could serve it; the provider states payment is **not** charged.
- **502** `provider_error`, **500** `settlement_failed` / `internal_error` — do not blindly re-pay. There is **no idempotency key** on this operation; check for a receipt first (step 6) because a second exact payment is a second charge.

## 6. Keep the receipt

`GET /v1/receipts/{receipt_id}` (`getReceipt`) with the UUID from `X-Solvela-Receipt` returns `payer_wallet`, `payment_scheme`, `tx_signature`, `amount_paid_atomic` / `amount_paid_usdc` and the `cost_breakdown` that produced the bill. The id is a bearer capability: unknown and malformed ids both return the same **404**, there is no listing endpoint, and the route is capped at 20 lookups/min per IP. Free requests produce no receipt.

## Rules that apply throughout

- Exact payments are **irreversible** — the transfer is wallet-to-wallet on-chain and overpayment is not refunded. If you need a refund path, use the escrow scheme (see `solvela-ai-escrow-paid-request`).
- Every response carries `x-solvela-request-id`; quote it to partnerships@solvela.ai if funds moved and nothing came back.
- The hosted gateway runs a 0% platform fee today; always read `fee_percent` from the 402 rather than assuming it.
