---
name: uw-swap
description: Use this skill to get cryptocurrency swap quotes, execute cross-chain swaps, and track swap status across THORChain, Mayachain, 1Inch, and P2P providers (LetsExchange, StealthEx, Quickex, Swapuz, Exolix, CCE).
version: "1.0.0"
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

1. **Rate** — `POST /v2/rate` to compare routes across providers (read-only, no order)
2. **Swap** — `POST /v2/swap` with exactly one `provider` + a `destinationAddress` to create the order
3. **Send funds** — do exactly what the route's `execution` block says: sign its tx, transfer to its `depositAddress` (+ any `attachment`), or deposit to `inboundAddress` with the `memo`. See [quote.md](references/quote.md).
4. **Track** — store the route's top-level `uuid` and POST `{ uuid }` to `/v2/track` until status is `completed`, `refunded`, `failed`, or `expired`; for DEX routes also send your broadcast tx hash as `inboundTxHash`. If status becomes `action_required`, the provider is holding the funds and needs the user to contact them — see [track.md](references/track.md#action-required) for `meta.pauseReason` and how to surface the provider's `contacts`.

## Error Codes

| Code | Meaning |
|---|---|
| 401 | Missing or invalid `X-Agent-Key` |
| 403 | Agent suspended |
| 409 | Agent name already taken |
| 429 | Rate limit exceeded |
| 400 | Validation error — check response body |
| 404 | No routes found for the requested swap |
