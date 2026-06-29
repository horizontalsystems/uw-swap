# Tracking Swaps

```
POST https://swap-api.unstoppable.money/agent/v2/track
Content-Type: application/json
X-Agent-Key: $USWAP_AGENT_KEY
```

Track by the committed route's top-level **`uuid`**. The `/v2/swap` response gives you:

```json
{
  "providers": ["THORCHAIN"], …, "execution": { … },
  "uuid": "b5b1b8c1-4a18-42a0-ab84-374d36f68f17"
}
```

Store the `uuid`. The server tracks by it alone — it already knows the provider and every swap detail from
the record, so you never build a request beyond the uuid (and, for DEX swaps, your broadcast tx hash).

## Request

**P2P (LetsExchange, StealthEx, Quickex, Swapuz, Exolix, CCE) and NEAR** — the provider sees your deposit
on its own, so the `uuid` alone is enough:

```json
{ "uuid": "b5b1b8c1-…" }
```

**DEX swaps (THORChain, Mayachain, 1inch, Barter, Circle)** — i.e. any route whose `execution.method` was
`thorchain_deposit` or `signed_transaction`. After you broadcast the tx, send its hash as `inboundTxHash`:

```json
{ "uuid": "b5b1b8c1-…", "inboundTxHash": "0xabc123…" }
```

That's the whole flow: **store `uuid` → (DEX) add `inboundTxHash` → POST.** The server remembers the hash
for subsequent polls. Everything else the tracker needs was stored when the swap was committed, so the
request never carries anything beyond `{ uuid, inboundTxHash? }`.

## Response

```json
{
  "status": "swapping",
  "providers": ["THORCHAIN"],
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

If the swap is recorded but not yet trackable (e.g. a DEX swap you haven't sent `inboundTxHash`
for), you get `409` with `{ "message": "Swap not trackable yet", "uuid": "…" }` — send the hash, then
poll again. A `404` means the `uuid` is unknown.

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
| `expired` | Timed out — no deposit was detected within ~72h, so tracking stopped. The user likely never sent funds (or sent too late). Treat as terminal; a late deposit may still be resolved server-side. |
| `unknown` | Status could not be determined |

**Terminal statuses** (stop polling): `completed`, `refunded`, `failed`, `expired`.

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

To help the user recover funds, surface the provider's contact info from `GET /v2/providers` (the `contacts` field — `{ email?, telegram?, ... }`) alongside `meta.pauseReason`. The user reaches out to the provider directly; the provider then completes or refunds the swap, after which the next poll will report a terminal status.
