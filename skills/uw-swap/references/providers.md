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

## Providers

| Provider | Type | Supported Chains / Notes |
|---|---|---|
| THORCHAIN | DEX | BTC, ETH, AVAX, BCH, LTC, DOGE, GAIA, BSC, and more |
| MAYACHAIN | DEX | BTC, ETH, DASH, KUJI, THOR, ARB, and more |
| ONEINCH | DEX aggregator | EVM same-chain: ETH, BSC, ARB, OP, AVAX, POL, BASE |
| BARTER | DEX aggregator | EVM same-chain |
| NEAR | DEX | NEAR ecosystem + cross-chain via 1Click |
| LETSEXCHANGE | P2P | Wide cross-chain coverage |
| STEALTHEX | P2P | Wide cross-chain coverage |
| QUICKEX | P2P | Wide cross-chain coverage |
| SWAPUZ | P2P | Wide cross-chain coverage |
| EXOLIX | P2P | Wide cross-chain coverage |

**DEX** = decentralized, on-chain execution.  
**P2P** = centralized order matching, requires `providerSwapId` for tracking.

## Token Lists

```
GET $USWAP_BASE_URL/v1/tokens?provider=THORCHAIN
X-Agent-Key: $USWAP_AGENT_KEY
```

Returns supported tokens for the given provider.  
**Note:** BARTER and ONEINCH do not expose a token list — use contract addresses directly.

## List Providers

```
GET $USWAP_BASE_URL/v1/providers
X-Agent-Key: $USWAP_AGENT_KEY
```

## Rate Limiting

- Starts at **15 req/hour** for new agents
- Scales up automatically with fulfillment ratio (up to **100 req/hour**)
- Agents with ≥50% fulfillment ratio receive a **3× bonus** (up to **200 req/hour**)
- Creating real orders (`dry: false`) but not sending funds lowers your ratio and rate limit
- After 3 suspensions an agent is permanently banned
