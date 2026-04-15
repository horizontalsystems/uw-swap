# Tracking Swaps

```
POST $USWAP_BASE_URL/v1/track
Content-Type: application/json
X-Agent-Key: $USWAP_AGENT_KEY
```

The request body varies by provider.

## P2P Providers (LETSEXCHANGE, STEALTHEX, QUICKEX, SWAPUZ, EXOLIX)

```json
{
  "provider": "LETSEXCHANGE",
  "providerSwapId": "<providerSwapId from quote response>"
}
```

## THORChain / Mayachain

```json
{
  "provider": "THORCHAIN",
  "hash": "<inbound tx hash>",
  "fromAsset": "BTC.BTC",
  "toAsset": "ETH.ETH",
  "toAddress": "0x..."
}
```

## NEAR

```json
{
  "provider": "NEAR",
  "depositAddress": "<targetAddress from quote response>"
}
```

## Response

```json
{
  "status": "swapping",
  "legs": [
    {
      "chainId": "bitcoin",
      "hash": "abc123...",
      "type": "inbound",
      "status": "completed",
      "fromAsset": "BTC.BTC",
      "fromAmount": "0.1",
      "fromAddress": "bc1q...",
      "toAsset": "ETH.ETH",
      "toAmount": "1.52",
      "toAddress": "0x..."
    }
  ]
}
```

## Statuses

| Status | Meaning |
|---|---|
| `not_started` | Funds not yet received |
| `pending` | Funds received, waiting to process |
| `swapping` | Swap in progress |
| `completed` | Swap finished, funds delivered |
| `refunded` | Swap failed, funds returned |
| `failed` | Swap failed, no refund |
| `unknown` | Status could not be determined |

**Terminal statuses** (stop polling): `completed`, `refunded`, `failed`.

Poll every 1-2 minutes until a terminal status is reached. THORChain swaps typically complete in 5–20 minutes. P2P swaps can take 10–60 minutes.
