# Providers, Chains & Asset Format

## Asset ID Format

```
CHAIN.SYMBOL
CHAIN.SYMBOL-CONTRACT_ADDRESS
```

**Examples:**

| Asset | ID |
|---|---|
| Bitcoin | `BTC.BTC` |
| Ether | `ETH.ETH` |
| Solana | `SOL.SOL` |
| USDC on Ethereum | `ETH.USDC-0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| Any ERC-20 (BARTER/ONEINCH) | `ETH.0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| BNB native | `BSC.BNB` |
| ARB native | `ARB.ETH` |

For BARTER and ONEINCH, use the contract address directly without a symbol — both assets must be on the same chain.

### THORCHAIN Secured Assets

THORChain Secured Assets use a **dash** instead of a dot — `CHAIN-SYMBOL[-CONTRACT]`. They're 1:1-backed claims on L1, held in THORChain's `x/bank` module, denominated in 8 decimals regardless of L1 native decimals.

| Asset | ID |
|---|---|
| Secured ETH | `ETH-ETH` |
| Secured BTC | `BTC-BTC` |
| Secured USDC on Ethereum | `ETH-USDC-0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |

**Constraints when quoting via THORCHAIN:**
- **L1 → Secured:** `destinationAddress` must be a `thor1…` address.
- **Secured → L1 / Secured:** `sourceAddress` must be the `thor1…` holder. The deposit is a `MsgDeposit` on THORChain — **the server does not build this transaction; the agent must construct and sign the `MsgDeposit` with the returned `memo`**. Inbound fee is reported as `0.02 RUNE` (a fixed gas approximation).

Trade Assets (`~`), synthetics (`/`), and derived assets (`THOR.X`) are **not** supported.

## Providers

| Provider | Type | AML Policy | Supported Chains / Notes |
|---|---|---|---|
| THORCHAIN | DEX | `excellent` | BTC, ETH, AVAX, BCH, LTC, DOGE, GAIA, BSC, and more |
| MAYACHAIN | DEX | `excellent` | BTC, ETH, DASH, KUJI, THOR, ARB, and more |
| ONEINCH | DEX aggregator | `excellent` | EVM same-chain: ETH, BSC, ARB, OP, AVAX, POL, BASE |
| BARTER | DEX aggregator | `excellent` | EVM same-chain |
| NEAR | DEX | `fair` | NEAR ecosystem + cross-chain via 1Click |
| LETSEXCHANGE | P2P | `good` | Wide cross-chain coverage |
| STEALTHEX | P2P | `fair` | Wide cross-chain coverage |
| QUICKEX | P2P | `good` | Wide cross-chain coverage. Supports AML address precheck (see below). |
| SWAPUZ | P2P | `good` | Wide cross-chain coverage |
| EXOLIX | P2P | `good` | Wide cross-chain coverage |
| CCE | P2P | `good` | Wide cross-chain coverage |

**DEX** = decentralized, on-chain execution.  
**P2P** = centralized order matching, requires `providerSwapId` for tracking.

## AML Policy

Every provider is classified with an `amlPolicy` describing how the provider handles AML (anti-money-laundering) checks. The policy is returned on both the provider entity (`GET /v1/providers`) and on every quote's route (`amlPolicy` field).

Use it to set user expectations before a swap — especially when privacy or the risk of funds being held matters.

| Policy | Meaning |
|---|---|
| `excellent` | Direct on-chain execution. No provider checks or freezes. Automatic refunds if swap fails. |
| `good` | Provider checks transactions automatically before completion. If issues are detected, the swap is rejected and funds are refunded. |
| `fair` | Additional verification may be required for some transactions. If issues are detected, funds are usually refunded automatically. |

Providers may also expose a `contact` field (typically an email) on the provider entity — use this to direct users when a swap is held for review.

### Running an AML Precheck

When the chosen route's provider is `QUICKEX`, run the address precheck **before** the user sends funds. Precheck both the `sourceAddress` (sender) and `destinationAddress` (receiver).

```
GET https://swap-api.unstoppable.money/agent/v1/quote/check-addresses?addresses=<csv>
X-Agent-Key: $USWAP_AGENT_KEY
```

Pass one or more addresses as a comma-separated list, e.g. `addresses=bc1q...,0xabc...`.

**Response:**

```json
{
  "passedAmlCheck": true,
  "results": [
    { "address": "bc1q...", "passed": true, "completed": true },
    { "address": "0xabc...", "passed": true, "completed": true }
  ]
}
```

| `passedAmlCheck` | Meaning | Action |
|---|---|---|
| `true` | All addresses passed | Safe to send funds |
| `false` | At least one address failed | **Do not send funds** — inform the user |
| `null` | Check incomplete/inconclusive | Retry later or warn the user |

A 502 response means the AML check service is unavailable — retry, or warn the user before proceeding.

This endpoint is powered by Quickex's AML checker. Only QuickEx currently triggers a pre-funding address check; the endpoint is also useful as a general due-diligence tool for any cross-chain swap.

## Token Lists

```
GET https://swap-api.unstoppable.money/agent/v1/tokens?provider=THORCHAIN
X-Agent-Key: $USWAP_AGENT_KEY
```

Returns supported tokens for the given provider.  
**Note:** BARTER and ONEINCH do not expose a token list — use contract addresses directly.

## List Providers

```
GET https://swap-api.unstoppable.money/agent/v1/providers
X-Agent-Key: $USWAP_AGENT_KEY
```

## Rate Limiting

- Starts at **15 req/hour** for new agents
- Scales up automatically with fulfillment ratio (up to **100 req/hour**)
- Agents with ≥50% fulfillment ratio receive a **3× bonus** (up to **200 req/hour**)
- Creating real orders (`dry: false`) but not sending funds lowers your ratio and rate limit
- After 3 suspensions an agent is permanently banned
