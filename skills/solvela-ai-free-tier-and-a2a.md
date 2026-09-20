---
name: solvela-ai-free-tier-and-a2a
description: Use Solvela at zero cost through the free model tier, and discover and quote it as an A2A agent — rehearsing the full contract, the 402 shape and the JSON-RPC task flow without holding a wallet or spending USDC.
api: openapi/solvela-ai-openapi.json
operations:
  - health
  - listModels
  - createChatCompletion
method: generated
generated: '2026-09-19'
grounding: operationIds exist verbatim in openapi/solvela-ai-openapi.json. Free-tier rules, limits and the A2A methods and error codes are quoted from the provider's free-tier and A2A documentation and from the agent card and JSON-RPC responses observed on api.solvela.ai on 2026-09-19.
---

# Try Solvela for free, and talk to it as an agent

## 1. Check the gateway is up

`GET /health` (`health`) — free, exempt from rate limits, returns `{"status":"ok"}`. `GET /v1/models` (`listModels`) is likewise free and exempt.

## 2. Use a zero-priced model with no wallet at all

A request is free if and only if its quoted cost is exactly 0 atomic USDC. Seventeen models were priced 0/0 on 2026-09-19 — fifteen NVIDIA NIM Nemotron models plus `google/gemini-3.1-flash-lite` and `openai/gpt-oss-120b`. Either name one directly or send `"model": "free"` (aliases `oss`, `open`): the router maps Simple → `nvidia/nvidia/llama-3.1-nemotron-nano-8b-v1`, Medium → `nvidia/nvidia/llama-3.3-nemotron-super-49b-v1`, Complex/Reasoning → `nvidia/nvidia/llama-3.1-nemotron-ultra-253b-v1`, with Gemini 3.1 Flash-Lite as fallback when NIM is down.

`POST /v1/chat/completions` (`createChatCompletion`) with such a model and **no** `PAYMENT-SIGNATURE` returns **200** directly — the 402 step is skipped because there is nothing to charge. A payment header sent with a free model is ignored.

Limits are deliberately tight: **5 requests / 60 s per IP** and **12 requests / 60 s across all free clients**, identified by the TCP peer IP (never `X-Forwarded-For`). A 429 carries `retry-after`. Free requests produce no receipt.

**Data-use caveat, in the provider's words:** free models are served on the upstream providers' free tiers, "whose terms may permit the provider to use submitted prompts and generated responses to improve their products (and human reviewers may process them) — do not send sensitive data through the free tier."

## 3. Rehearse a paid call without paying

Send a paid model without `PAYMENT-SIGNATURE`: the **402** body is the exact quote (`accepts[]`, `cost_breakdown`, `max_timeout_seconds: 300`) and on this route also `extensions.bazaar` with the input JSON Schema and an output example. Nothing runs and nothing is charged. This is the provider's dry run.

## 4. Discover Solvela as an A2A agent

Fetch `https://api.solvela.ai/.well-known/agent-card.json` (legacy alias `/.well-known/agent.json`). The card is A2A `protocolVersion 0.3.0`, `preferredTransport JSONRPC`, endpoint `https://api.solvela.ai/a2a`, `streaming: false`, `pushNotifications: false`, and declares two **required** extensions: AP2 (`roles: [merchant]`) and a2a-x402 (`network: solana`, `asset` = the USDC mint, `schemes: [exact, escrow]`). One skill: `chat-completion`, text/plain in and out.

## 5. Quote and (optionally) pay over JSON-RPC

- `message/send` with a text message and **no** `taskId` returns a Task in state `input-required` whose metadata `x402.payment.required` is the same `PaymentRequired` object as the REST 402. A blocked prompt is rejected here, before any funds move.
- To pay, send `message/send` again with the `taskId` and the signed payment in `x402.payment.payload`; a `completed` Task returns the completion as both status message and artifact plus `x402.payment.receipts` (`tx_signature`, and the `/v1/receipts/<uuid>` path).
- `tasks/get { id }` works for about 10 minutes after the last state change; the task id is a bearer capability, rate-limited per IP.
- `tasks/cancel { id }` succeeds only while the task is `input-required` — the "decline the quote" path. Unpaid tasks also simply expire at the 10-minute TTL.
- Do not send images: A2A is text-only and image parts are rejected as invalid params. `message/stream` is not routed (-32601); push-notification methods return -32003.
- Error codes: -32001 task not found/expired, -32002 not cancelable, -32007 payment failed, -32008 provider error, -32009 model not found. Observed live: an empty `params` returns -32602 `Invalid params: missing field "message"`.

## 6. Or install the MCP server instead

`npx -y @solvela/mcp-server` (or `solvela mcp install --host=claude-code`) exposes `chat`, `smart_chat`, `list_models`, `wallet_status`, `spending`, `web_search`, `solana_price` and, with `SOLVELA_ESCROW_MODE=enabled`, `deposit_escrow`. It creates a local wallet on first run; the free tier works before you fund it.
