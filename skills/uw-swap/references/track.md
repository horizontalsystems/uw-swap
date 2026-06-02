# Tracking Swaps

```
POST https://swap-api.unstoppable.money/agent/v1/track
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

## CIRCLE

```json
{
  "provider": "CIRCLE",
  "hash": "<burn tx hash on the source chain>",
  "chainId": "<source chainId, e.g. ethereum>",
  "fromAsset": "ETH.USDC-0X...",
  "toAsset": "BASE.USDC-0X...",
  "toAddress": "0x...",
  "providerSwapId": "<providerSwapId from quote response>",
  "fromAddress": "0x...",
  "fromAmount": "100.0"
}
```

`fromAddress` (source EOA) and `fromAmount` (gross sell amount) are optional — Circle's API can't supply them, so pass them through to have them echoed back on the response legs; otherwise those fields come back empty.

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
| `action_required` | Funds reached the provider but the swap is stuck and won't auto-resolve. The user must contact the provider directly to request a manual refund or unblock. See "Action required" below. |
| `completed` | Swap finished, funds delivered |
| `refunded` | Swap failed, funds returned |
| `failed` | Swap failed, no refund |
| `unknown` | Status could not be determined |

**Terminal statuses** (stop polling): `completed`, `refunded`, `failed`.

`action_required` is **non-terminal** — keep polling, because the provider may still resolve the swap to `completed` / `refunded` / `failed` after manual intervention.

Poll every 1-2 minutes until a terminal status is reached. THORChain swaps typically complete in 5–20 minutes. P2P swaps can take 10–60 minutes.

## Action required

When `status === "action_required"`, the response includes `meta.pauseReason` describing why the swap is stuck:

| Reason | Cause |
|---|---|
| `aml` | Provider's AML check blocked the deposit |
| `kyc_required` | Provider requires user KYC submission before withdrawal |
| `frozen` | Provider explicitly froze the order pending review |
| `overdue_with_funds` | Order/rate window missed after the deposit arrived |
| `provider_error` | Provider-side error after a deposit (catch-all) |
| `manual_review` | Provider's generic "needs human attention" state |

To help the user recover funds, surface the provider's contact info from `GET /v1/providers` (the `contacts` field — `{ email?, telegram?, ... }`) alongside `meta.pauseReason`. The user reaches out to the provider directly; the provider then completes or refunds the swap, after which the next poll will report a terminal status.
