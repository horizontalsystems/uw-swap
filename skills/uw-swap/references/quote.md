# Quotes & Swap Execution

## Step 1 — Dry Quote (compare routes)

```
POST $USWAP_BASE_URL/v1/quote
Content-Type: application/json
X-Agent-Key: $USWAP_AGENT_KEY
```

```json
{
  "sellAsset": "BTC.BTC",
  "buyAsset": "ETH.ETH",
  "sellAmount": "0.1",
  "slippage": 3,
  "destinationAddress": "0x...",
  "refundAddress": "bc1q...",
  "sourceAddress": "bc1q..."
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| sellAsset | string | yes | Asset to sell — see [providers.md](providers.md) for format |
| buyAsset | string | yes | Asset to buy |
| sellAmount | string | yes | Decimal string, e.g. `"0.1"` |
| slippage | number | yes | Tolerance 0–99 (e.g. `3` = 3%) |
| destinationAddress | string | no | Receiving address |
| refundAddress | string | no | Refund address on failure |
| sourceAddress | string | no | Sending address |
| providers | string[] | no | Filter to specific providers. If omitted, all are queried |
| dry | boolean | no | Default `true`. `false` = create real order |
| floating | boolean | no | **P2P providers only.** Default `true`. Whether the rate can float. Set `false` to lock the rate (fixed-rate swap). |
| rateId | string | no | **P2P providers only.** Lock a previously quoted rate. Use together with `rateIdExpiresAt` returned in a prior dry quote. |
| streamingInterval | number | no | **THORChain/Mayachain only.** Number of blocks between each streaming sub-swap. |
| streamingQuantity | number | no | **THORChain/Mayachain only.** Number of sub-swaps to split the swap into. Higher = better price, slower execution. |

**Response:**

```json
{
  "routes": [
    {
      "sellAsset": "BTC.BTC",
      "buyAsset": "ETH.ETH",
      "sellAmount": "0.1",
      "expectedBuyAmount": "1.52",
      "inboundAddress": "bc1q...",
      "targetAddress": "bc1q...",
      "destinationAddress": "0x...",
      "fees": [{ "type": "liquidity", "asset": "BTC.BTC", "amount": "0.0003" }],
      "estimatedTime": { "inbound": 600, "swap": 60, "outbound": 600, "total": 1260 },
      "providers": ["THORCHAIN"],
      "memo": "=:ETH.ETH:0x...",
      "providerSwapId": "..."
    }
  ],
  "providerErrors": [
    { "provider": "LETSEXCHANGE", "error": "Amount too small", "errorCode": "amountTooSmall", "minimumAmount": 0.001 }
  ]
}
```

Pick the route with the best `expectedBuyAmount` or lowest fees.

## Step 2 — Real Quote (create order)

Call the same endpoint with `dry: false` and exactly one provider:

```json
{
  "sellAsset": "BTC.BTC",
  "buyAsset": "ETH.ETH",
  "sellAmount": "0.1",
  "slippage": 3,
  "destinationAddress": "0x...",
  "providers": ["THORCHAIN"],
  "dry": false
}
```

The response is identical in shape. Now read the following fields from the chosen route carefully before sending funds.

## Step 3 — Send Funds

> **Read this section fully. Missing any required field will result in lost funds.**

### ONEINCH and BARTER — use the `tx` field

For ONEINCH and BARTER, the response includes a `tx` field containing pre-built EVM transaction data. **You must submit this exact transaction** — do not construct your own. Sending funds directly to `inboundAddress` without using `tx` will result in lost funds.

```json
{
  "tx": {
    "to": "0x1111111254EEB25477B68fb85Ed929f73A960582",
    "data": "0x...",
    "value": "0x0",
    "gasLimit": "0x...",
    "gasPrice": "0x..."
  }
}
```

Sign and broadcast this transaction using your EVM wallet. The `to` address is the router contract; `data` encodes the swap parameters.

### ERC-20 Approval (ONEINCH and BARTER)

When selling an ERC-20 token (not the native gas token) via ONEINCH or BARTER, the route's `meta.approvalAddress` field will be set. You **must approve this address to spend your tokens** before submitting the swap transaction.

```json
{
  "meta": {
    "approvalAddress": "0x1111111254EEB25477B68fb85Ed929f73A960582"
  }
}
```

Steps:
1. Check if `meta.approvalAddress` is present in the route.
2. If yes, call `approve(approvalAddress, sellAmount)` on the ERC-20 token contract from `sourceAddress`.
3. Wait for the approval transaction to be confirmed.
4. Then submit the `tx` transaction from step above.

If the current allowance is already sufficient, you can skip the approval. Selling native gas tokens (ETH, BNB, etc.) never requires approval.

### THORChain and Mayachain — `memo` is required

Send exactly `sellAmount` of `sellAsset` to `inboundAddress`. Include the `memo` field as a transaction memo/OP_RETURN. **Omitting the memo will cause a refund** (and you will pay network fees twice).

### P2P providers — `txExtraAttribute`

Some assets require an extra identifier attached to the inbound transaction (e.g. destination tag for XRP, memo for RUNE/GAIA/TON). When present, `txExtraAttribute` in the route response contains the required field(s):

```json
{
  "txExtraAttribute": {
    "destinationTag": "123456789"
  }
}
```

```json
{
  "txExtraAttribute": {
    "memo": "abc123"
  }
}
```

Include the key-value pair from `txExtraAttribute` in your inbound transaction exactly as returned. **Omitting it will result in lost funds.** Affected assets include:

| Asset | Extra field | Required on |
|---|---|---|
| XRP.XRP | `destinationTag` (integer) | All P2P providers |
| RUNE.RUNE | `memo` (string) | All P2P providers |
| GAIA.ATOM | `memo` (string) | All P2P providers |
| TON.TON | `memo` (string) | All P2P providers |
| NEAR.NEAR | `memo` (string) | NEAR provider |

If `txExtraAttribute` is absent or empty, no extra field is needed.

### Mayachain + ZCash — `shielded_memo_address`

When selling `ZEC.ZEC` via Mayachain with `dry: false`, the route may include a `shielded_memo_address` field. This is a Zcash unified address that encodes the memo using the shielded protocol. Use this as the destination instead of `inboundAddress` when present — it enables private memo delivery required by Mayachain's ZCash integration.

```json
{
  "shielded_memo_address": "u1zcash..."
}
```

## Step 4 — Track

See [track.md](track.md).

## Important Rules

- **`dry: false` requires exactly one provider** in the `providers` array.
- If you request a real quote but never send funds, your fulfillment ratio drops and your rate limit is reduced.
- `providerSwapId` is what you use to track P2P swaps.
- `rateId` and `rateIdExpiresAt` are returned in dry quote responses for P2P providers — use them to lock the rate in the real quote before they expire.

