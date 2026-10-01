# Tracking Swaps

```
POST https://swap-api.unstoppable.money/agent/v3/track
Content-Type: application/json
X-Agent-Key: $USWAP_AGENT_KEY
```

Track by the **`uuid`** that `/v3/commit` returned:

```json
{ "status": "ok", "uuid": "b5b1b8c1-4a18-42a0-ab84-374d36f68f17", "sellAmount": "0.1", … }
```

Store the `uuid`. The server tracks by it alone — it already knows the provider and every swap detail from
the record, so you never build a request beyond the uuid (and, for DEX swaps, your broadcast tx hash).

## Request

**P2P (LetsExchange, StealthEx, Quickex, Swapuz, Exolix, CCE, Lizex, Bitania) and NEAR** — the provider sees your deposit
on its own, so the `uuid` alone is enough:

```json
{ "uuid": "b5b1b8c1-…" }
```

**DEX swaps (THORChain, Mayachain, 1inch, Barter, Circle, Jupiter, LI.FI, Soroswap, Aquarius, Stellar DEX,
StellarBroker, Axelar ITS)** — i.e. any route whose `execution.method` was `thorchain_deposit`,
`signed_transaction`, or `stellar_broker`. You signed the preview's tx, so you had the hash at commit
time and should have passed it to `/v3/commit` as `inboundTxHash` **before broadcasting**. If you did not,
send it on the first `/track`:

```json
{ "uuid": "b5b1b8c1-…", "inboundTxHash": "0xabc123…" }   // Solana: the tx signature; Tron: the tx id (txID); Stellar: the tx hash (fee-bumped ⇒ the outer hash works)
```

**P2P / NEAR**: send the hash of the transfer you built on your first `/track`, before broadcasting it.
It is not needed to track (the provider watches its own address) but it is kept on the record.

The server remembers the hash for subsequent polls; the same hash again is a no-op, and a different one
replaces it as long as nothing has been observed yet (a fee bump or speed-up), after which it is a `409`.
Everything else the tracker needs was stored at commit, so the request never carries anything beyond
`{ uuid, inboundTxHash? }`.

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

Before anything has been observed on-chain or at the provider you get `200` with the record's own
status instead of a tracker report — `{ "uuid", "status": "signed" | "not_started", "providers", "hash"?,
"legs": [] }`. Keep polling. A `404` means the `uuid` is unknown; a `409` means you sent a different
`inboundTxHash` than the one already on a swap whose deposit was already seen.

## Statuses

| Status | Meaning |
|---|---|
| `not_started` | Order created, no inbound hash reported, funds not yet received |
| `signed` | You reported the inbound tx hash; nothing observed yet |
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

To help the user recover funds, surface the provider's contact info from `GET /v3/providers` (the `contacts` field — `{ email?, telegram?, ... }`) alongside `meta.pauseReason`. The user reaches out to the provider directly; the provider then completes or refunds the swap, after which the next poll will report a terminal status.
