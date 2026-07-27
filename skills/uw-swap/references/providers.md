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
| Any ERC-20 (BARTER/ONEINCH/LI.FI) | `ETH.0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| Any SPL token (JUPITER/LI.FI) | `SOL.EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` or the bare mint |
| BNB native | `BSC.BNB` |
| ARB native | `ARB.ETH` |
| Native gas for LI.FI (sentinel) | `ETH.0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` |
| Native TRX for LI.FI | `TRON.TRX` |
| TRC-20 for LI.FI (base58 contract) | `TRON.TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t` |
| Stellar native XLM | `XLM.XLM` |
| Stellar classic asset (`CODE-ISSUER`) | `XLM.USDC-GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN` |

For BARTER and ONEINCH, use the contract address directly without a symbol — both assets must be on the same chain.

For JUPITER, use a known identifier (`SOL.SOL`, `SOL.USDC-EPJF…`) or the SPL mint address — `SOL.<mint>` or a bare mint, no `chainId` needed. **Mints are case-sensitive base58 — pass them verbatim** (never upper/lowercase them). The wSOL mint (`So11111111111111111111111111111111111111112`) means native SOL.

For LI.FI (cross-chain bridge/DEX aggregator, EVM + Solana + Tron), assets are **self-describing**, so `sellAsset` and `buyAsset` may be on **different chains** — no `chainId` field needed. Encode each side by its chain: EVM token → `CHAIN.<contract>` (e.g. `BASE.0x833589…`) or a known identifier; EVM native gas → the chain-prefixed sentinel `CHAIN.0xEeee…EEeE` (e.g. `ETH.0xEeee…EEeE`); Solana → `SOL.<mint>` (wSOL mint = native SOL); Tron → `TRON.TRX` (native) or `TRON.<contract>` (TRC-20, base58 — **case-sensitive, pass verbatim**). LI.FI keeps **no token list** — pass any supported token; unroutable pairs just return no route. (Bitcoin is **not** supported by LI.FI.)

### Stellar Assets

The four Stellar-native providers (STELLARBROKER, SOROSWAP, AQUARIUS, STELLAR_DEX) use the same `CHAIN.SYMBOL-CONTRACT` shape, where the "contract" is the issuer account: native XLM → `XLM.XLM`; a classic asset → `XLM.<CODE>-<GISSUER…>` (the `CODE:ISSUER` form and a chain-less `CODE-GISSUER` are also accepted).

**Asset codes are CASE-SENSITIVE** (`yXLM` ≠ `YXLM`) — pass the code exactly as the issuer defines it; the issuer itself is uppercase-only StrKey. Classic assets are always **7 decimals**. **Soroban-only tokens (`C…` contract ids) are not supported.** None of these providers exposes a token list.

Two constraints to check before quoting:
- **STELLARBROKER and AQUARIUS settle on the trader's own account** — `destinationAddress` must equal `sourceAddress`, or `/v2/swap` returns `400`. SOROSWAP and STELLAR_DEX accept a third-party destination.
- **Buying a classic asset requires the recipient to already hold that asset's trustline** (and the destination account to exist). The server pre-flights this and fails with `400` rather than letting the transaction revert — tell the user to add the trustline first.

### Axelar ITS (bridge, not a swap)

AXELAR_ITS moves **one token 1:1** between Stellar and Ethereum, so `sellAsset` and `buyAsset` must be the **same asset on different chains**. Only two assets, either direction:

| Asset | Stellar side | Ethereum side |
|---|---|---|
| XLM | `XLM.XLM` | `ETH.XLM-0X8CF74FC1EC7B2187DDA77EA289F78CC54E2B7C8B` |
| SHX | `XLM.SHX-GDSTRSHXHGJ7ZIVRBXEYE5Q74XUVCUSEKEBR7UCHEUUEK72N7I7KJ6JH` | `ETH.SHX-0X516D31321928700C6B4FB0DB0C8C6BC5D6799787` |

`expectedBuyAmount` equals `sellAmount` exactly, `minBuyAmount` equals it too, and `slippage` doesn't apply. The cost is the Axelar gas prepayment, shown as a `liquidity` fee in the source chain's native asset. Delivery takes ~0.5–3 min from Stellar, ~17 min from Ethereum.

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
| JUPITER | DEX aggregator | `excellent` | Solana same-chain: SOL + SPL tokens |
| LI.FI | Bridge / DEX aggregator | `excellent` | Cross-chain EVM + Solana + Tron (bridges + DEXs): ETH, BSC, POL, ARB, OP, BASE, AVAX, Solana, Tron. Sell and buy may be on different chains. No token list. (Bitcoin not supported.) |
| CIRCLE | Bridge | `excellent` | USDC-only cross-chain bridge (Circle CCTPv2). EVM chains: ETH, BASE, ARB, OP, POL, AVAX, and more. Source and destination chains must differ. |
| STELLARBROKER | DEX aggregator | `excellent` | Stellar only. Executes as an interactive WebSocket session (`stellar_broker`), not a signed tx — you sign, the broker submits. `destinationAddress` must equal `sourceAddress`. `minBuyAmount` is always `null`. No token list. |
| SOROSWAP | DEX aggregator | `excellent` | Stellar only. Routes Soroswap + Phoenix + SDEX. Third-party `destinationAddress` supported. No token list. |
| AQUARIUS | DEX (AMM) | `excellent` | Stellar only. Soroban AMM router. `destinationAddress` must equal `sourceAddress`. No token list. |
| STELLAR_DEX | DEX | `excellent` | Stellar only. Native order book + classic liquidity pools via Horizon path payments. Third-party `destinationAddress` supported. No token list. |
| AXELAR_ITS | Bridge | `excellent` | Same-token 1:1 bridge, Stellar ↔ Ethereum, XLM and SHX only. Not a swap — both sides are the same asset. |
| NEAR | DEX | `fair` | NEAR ecosystem + cross-chain via 1Click |
| LETSEXCHANGE | P2P | `good` | Wide cross-chain coverage |
| STEALTHEX | P2P | `fair` | Wide cross-chain coverage |
| QUICKEX | P2P | `good` | Wide cross-chain coverage. Supports AML address precheck (see below). |
| SWAPUZ | P2P | `good` | Wide cross-chain coverage |
| EXOLIX | P2P | `good` | Wide cross-chain coverage |
| CCE | P2P | `good` | Wide cross-chain coverage |

**DEX** = decentralized, on-chain execution.  
**P2P** = centralized order matching; the provider watches its deposit address, so tracking needs only the route's `uuid` (no broadcast tx hash).

## AML Policy

Every provider is classified with an `amlPolicy` describing how the provider handles AML (anti-money-laundering) checks. The policy is returned on both the provider entity (`GET /v2/providers`) and on every quote's route (`amlPolicy` field).

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
GET https://swap-api.unstoppable.money/agent/v2/check-addresses?addresses=<csv>
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
GET https://swap-api.unstoppable.money/agent/v2/tokens?provider=THORCHAIN
X-Agent-Key: $USWAP_AGENT_KEY
```

Returns supported tokens for the given provider.  
**Note:** BARTER, ONEINCH, JUPITER, LI.FI, STELLARBROKER, SOROSWAP, AQUARIUS and STELLAR_DEX do not expose a token list — encode assets directly (contract address for EVM, mint for Solana, `XLM.CODE-GISSUER…` for Stellar). AXELAR_ITS does publish one.

## List Providers

```
GET https://swap-api.unstoppable.money/agent/v2/providers
X-Agent-Key: $USWAP_AGENT_KEY
```

Each provider includes `executionType` — the single execution method it commits to: `transfer`,
`signed_transaction`, `thorchain_deposit`, or `stellar_broker`. If you can only relay a deposit address
to the user (no wallet to sign/build a tx), use `transfer` providers only, and filter on this **before
quoting**. `stellar_broker` (STELLARBROKER) is the most demanding: it needs a live WebSocket session
that signs transactions on demand, so skip it unless you can drive one.

## Rate Limiting

- Starts at **15 req/hour** for new agents
- Scales up automatically with fulfillment ratio (up to **100 req/hour**)
- Agents with ≥50% fulfillment ratio receive a **3× bonus** (up to **200 req/hour**)
- Creating real orders (via `/v2/swap`) but not sending funds lowers your ratio and rate limit
- After 3 suspensions an agent is permanently banned
