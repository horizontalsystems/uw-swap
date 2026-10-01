---
name: uw-swap
description: Use this skill to get cryptocurrency swap quotes, execute cross-chain swaps, and track swap status across THORChain, Mayachain, 1Inch, Barter, Jupiter (Solana), LI.FI (cross-chain EVM + Solana + Tron), Circle, NEAR, the Stellar venues (StellarBroker, Soroswap, Aquarius, Stellar DEX), the Axelar ITS bridge, and P2P providers (LetsExchange, StealthEx, Quickex, Swapuz, Exolix, CCE, Lizex, Bitania).
version: "1.2.0"
metadata:
  author: Horizontal Systems
  homepage: https://swap.unstoppable.money
  openclaw:
    emoji: 🔄
    primaryEnv: USWAP_AGENT_KEY
    requires:
      env:
        - USWAP_AGENT_KEY
---

# USwap — Cross-Chain Swap Aggregator

All requests require `X-Agent-Key: $USWAP_AGENT_KEY` header.  
Base URL: `https://swap-api.unstoppable.money/agent`

## Task Routing

| Task | Reference |
|---|---|
| First-time setup, get an agent key | [registration.md](references/registration.md) |
| Get quotes, pick a route, execute a swap | [quote.md](references/quote.md) |
| Track swap status after funds are sent | [track.md](references/track.md) |
| Providers, supported chains, asset ID format | [providers.md](references/providers.md) |

## Swap Flow (Summary)

1. **Rate** — `POST /v3/rate` to compare routes across providers (read-only, no order)
2. **Preview** — `POST /v3/preview` with exactly one `provider` + a `destinationAddress`. Returns the committed route and a `previewToken`; creates **nothing**. For P2P / NEAR routes it has no `execution` yet (the deposit address arrives at commit); for every other provider the `execution` is final.
3. **Commit** — when the user confirms, `POST /v3/commit { previewToken, inboundTxHash? }`. This creates the order (and the tracking `uuid`). Idempotent: resend the same token on a retry, never a fresh preview. If you already signed the preview's tx, pass its hash here **before broadcasting**.
4. **Send funds** — do exactly what `execution` says: sign its tx, transfer to its `depositAddress` (+ any `attachment`), deposit to `inboundAddress` with the `memo`, or (StellarBroker only) run the broker's WebSocket session. See [quote.md](references/quote.md).
5. **Track** — POST `{ uuid, inboundTxHash? }` to `/v3/track` until status is `completed`, `refunded`, `failed`, or `expired`. Report the tx hash **before broadcasting** — in `commit` if you signed the preview's tx, otherwise in your first `track` — so the server can finish tracking even if you never call again. If status becomes `action_required`, the provider is holding the funds and needs the user to contact them — see [track.md](references/track.md#action-required) for `meta.pauseReason` and how to surface the provider's `contacts`.

## Error Codes

| Code | Meaning |
|---|---|
| 401 | Missing or invalid `X-Agent-Key` |
| 403 | Agent suspended |
| 409 | Agent name already taken |
| 429 | Rate limit exceeded |
| 400 | Validation error — check response body |
| 404 | No routes found for the requested swap |
| 409 | On `/v3/commit`: `in_progress` (retry shortly, same token), `refresh_required` (preview again), `rate_changed` (the order drifted past your slippage; preview again) |
