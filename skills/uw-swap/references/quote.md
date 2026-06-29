# Quotes & Swap Execution

> A committed route carries one **`execution`** object that tells you exactly — and only — what you must
> do to send funds. You switch on `execution.method` and handle that one branch; nothing else applies.
> There is no flat set of optional fields to reconstruct intent from. (A `/rate` route is economics only
> — no `execution` until you commit with `/swap`.)

## Endpoints

The quote contract is split by intent into two endpoints:

```
POST https://swap-api.unstoppable.money/agent/v2/rate    ← compare routes, no order
POST https://swap-api.unstoppable.money/agent/v2/swap    ← commit: create the order with ONE provider
Content-Type: application/json
X-Agent-Key: $USWAP_AGENT_KEY
```

`/rate` is read-only — it prices the swap across providers and creates nothing. `/swap` commits: it
creates the real order with the single provider you name and returns the executable route.

**Flow:** call `/rate` to compare → pick the best route → call `/swap` with that provider to create the
order → send funds per its `execution` → poll `/track`.

## `/rate` request

| Field | Type | Required | Description |
|---|---|---|---|
| sellAsset | string | yes | Asset to sell — see [providers.md](providers.md) for format |
| buyAsset | string | yes | Asset to buy |
| sellAmount | string | yes | Decimal string, e.g. `"0.1"` |
| slippage | number | yes | Tolerance 0–99 |
| providers | string[] | no | Filter to specific providers |
| floating | boolean | no | P2P only. Default `true` |
| streamingInterval / streamingQuantity | number | no | THORChain/Mayachain only |

A rate needs no addresses — it builds nothing.

## `/swap` request

| Field | Type | Required | Description |
|---|---|---|---|
| sellAsset / buyAsset / sellAmount / slippage | | yes | Same as `/rate` |
| **provider** | string | yes | The **single** provider to create the order with |
| **destinationAddress** | string | yes | Receiving address — funds must land somewhere |
| refundAddress | string | no | Refund address on failure (required by most P2P providers) |
| sourceAddress | string | conditional | The build signal — see below. Required for `signed_transaction` providers (1inch/Barter/Circle) since the tx needs a `from`. |
| rateId | string | no | P2P only. Lock a previously quoted rate |
| floating | boolean | no | P2P only. Default `true` |
| streamingInterval / streamingQuantity | number | no | THORChain/Mayachain only |

**`sourceAddress` is the build signal.** Supply it and the server returns a ready-to-sign `unsignedTx`
for chains it can build for (thin wallets). Omit it and you get only the raw instructions and build the
tx yourself (HD wallets that build a better tx — e.g. multi-UTXO sweeps). It is **orthogonal** to the
execution method: it only adds an optional `unsignedTx` to `transfer` and `thorchain_deposit` routes.
`signed_transaction` routes always carry their tx because the swap *is* a contract call — there is no
address-transfer alternative — so those providers require `sourceAddress`.

## `/rate` response

```json
{ "routes": [ Route, ... ], "providerErrors": [ ... ] }
```

Each `providerErrors` entry is `{ provider, error, errorCode?, minimumAmount?, maximumAmount? }`. Each
`Route`:

```jsonc
{
  "providers": ["THORCHAIN"],          // all providers in the route (one today; multiple for future multistep)
  "sellAsset": "BTC.BTC", "sellAmount": "0.1",
  "buyAsset": "ETH.ETH", "expectedBuyAmount": "1.52",
  "minBuyAmount": "1.49",              // promised floor used to score the provider
  "fees": [{ "type": "liquidity", "protocol": "THORCHAIN", "chain": "BTC", "asset": "BTC.BTC", "amount": "0.0003" }],
  "estimatedTime": { "inbound": 600, "swap": 60, "outbound": 600, "total": 1260 },
  "expiresAt": 1718700000000,          // P2P rate-lock expiry (epoch ms), if any
  "rate": { "id": "...", "floating": true },  // P2P rate lock, if lockable
  "amlPolicy": "good",
  "accuracy": { "matched": 98, "total": 100, "avgDeviation": -0.4 },
  "amlErrors": [{ "message": "...", "level": 2 }],   // present only if the provider flagged AML issues
  "approvalSpender": "0x1111…",                       // EVM providers (1inch/Barter/Circle) only — the ERC20 spender to approve before swapping
  "meta": { "thorchain": { "slippageBps": 12, "totalBps": 30 } }  // provider-namespaced extras (advanced); not needed to execute
}
```

`expectedBuyAmount` is already net of all fees. Pick the route with the best `expectedBuyAmount`.

A `/rate` route carries economics only: amounts, `fees`, `estimatedTime`, `minBuyAmount`,
`amlPolicy`/`accuracy`, and for P2P a `rate` + `expiresAt` lock. It has **no `execution`** and no `uuid`
tracking handle — there's no order, deposit address, or tx yet. Those appear **only** on the `/v2/swap`
response (below), which creates the order. (`approvalSpender` is the one exception — present on a rate so
you can pre-check the ERC20 allowance before committing; on `/swap` the same spender rides
`execution.approval`.)

### Fee types

`expectedBuyAmount` is already net of every fee, so you don't need to sum `fees[]` to know the output —
but the breakdown is useful when explaining costs to a user. Each entry's `type`:

| Type | What it covers |
|---|---|
| `service` | The aggregator's service fee. |
| `liquidity` | Provider / protocol costs: the underlying provider's cut and any relayer/forwarder fee (e.g. Circle's mint relayer). |
| `inbound` / `outbound` | Source-/destination-chain network (gas) fees. |

A route may carry several entries of different types. Each amount is denominated in that entry's `asset`,
and each carries the `protocol` (which provider) and `chain` it applies to.

## `/swap` response

`/swap` returns the **single executable route directly** (no `{ routes }` wrapper) — the same `Route`,
now carrying the `execution` block and a top-level `uuid` tracking handle because the order exists:

```jsonc
{ "providers": ["QUICKEX"], "expectedBuyAmount": "1.52", …, "execution": { … }, "uuid": "b5b1b8c1-…" }
```

On failure it returns the provider error as the body with a matching HTTP status:

```jsonc
// HTTP 400 / 404 / 409 / 502 / 503 / 504
{ "error": "Amount out of range", "provider": "QUICKEX", "errorCode": "amountOutOfRange" }
```

| HTTP | When |
|---|---|
| 400 | amount out of range / unsupported pair or asset / invalid params |
| 404 | no route |
| 409 | rate expired (re-quote) |
| 503 | provider suspended |
| 504 | provider timed out |
| 502 | upstream provider failure (network / bad response) |

## `execution` — the only thing you act on

Exactly one of three shapes. Switch on `method`.

### `signed_transaction` — 1inch, Barter, Circle, StellarBroker

The swap is a contract call. **Sign and broadcast the provided transaction(s) verbatim.** There is no
deposit address; do not construct your own tx.

```jsonc
{
  "method": "signed_transaction",
  "chain": "ETH",
  "transactions": [
    { "kind": "evm", "to": "0x1111…", "from": "0xYou", "value": "0x0", "data": "0x…", "gas": "0x…", "gasPrice": "0x…" }
  ],
  "approval": { "token": "ETH.USDT-0x…", "spender": "0x1111…", "amount": "1000000" }  // only when selling a token
}
```

- If `approval` is present, first call `approve(spender, amount)` on the token and wait for confirmation, then submit the tx.
- `transactions` is an array (StellarBroker returns several to submit in parallel). Today it's length 1 for EVM providers; submit each.
- `kind` decodes the tx: `evm` (object), `psbt`/`solana`/`near`/`stellar` (base64 string under `psbt`/`message`/`tx`/`xdr`), `cosmos`/`ripple`/`ton`/`tron` (object under `tx`).

### `transfer` — P2P providers, NEAR

Make a **standard transfer** of `amount` of `asset` to `depositAddress`.

```jsonc
{
  "method": "transfer",
  "chain": "XRP",
  "depositAddress": "r…",
  "amount": "0.1",
  "asset": "XRP.XRP",
  "attachment": { "type": "destination_tag", "value": "123456789" },  // include EXACTLY, or funds are lost
  "unsignedTx": { "kind": "ripple", "tx": { … } },                    // only if you sent sourceAddress
  "qr": { "str": "xrp:r…?dt=123456789", "dataURL": "data:image/png;base64,…" }  // optional deposit QR
}
```

- **`attachment`** is an order identifier the provider uses to credit your funds. **If present, you must include it**, or the deposit is lost.
  - `{ "type": "destination_tag", "value": "…" }` — XRP destination tag.
  - `{ "type": "text", "value": "…" }` — memo field (RUNE / GAIA / TON / NEAR).
- **`unsignedTx`** appears only when you sent `sourceAddress` and the chain is buildable. Sign and broadcast it, *or* ignore it and build your own transfer to `depositAddress`.
- **`qr`** is an optional server-rendered deposit QR (`str` = encoded payload, `dataURL` = image). SDKs may render their own from `depositAddress`/`amount`/`attachment` instead.

### `thorchain_deposit` — THORChain, Mayachain

Deposit `amount` of `asset` to the protocol vault `inboundAddress` with the **`memo`** bound to the
deposit. The memo **is** the swap order — never omit it. `delivery.kind` tells you how to bind it.

```jsonc
{
  "method": "thorchain_deposit",
  "protocol": "THORCHAIN",
  "chain": "BTC",
  "inboundAddress": "bc1q…",
  "amount": "0.1",
  "asset": "BTC.BTC",
  "memo": "=:ETH.ETH:0x…",
  "expiry": "…",
  "delivery": {
    "kind": "utxo_op_return",
    "shieldedMemoAddress": "u1…",      // Maya ZEC only: send the memo to this 2nd shielded output
    "unsignedTx": { "kind": "psbt", "psbt": "…" }  // only if you sent sourceAddress
  }
}
```

`delivery.kind`:
- **`evm_contract_call`** — the memo is encoded in the router call. Submit `unsignedTx` (an `evm` tx). `router` is the contract; if `approval` is present, approve first. A plain transfer to `inboundAddress` will **not** work.
- **`utxo_op_return`** — attach `memo` as an `OP_RETURN` output alongside the `inboundAddress` output. A plain wallet send cannot do this; either submit `unsignedTx` (a single-address PSBT) or build the multi-output tx yourself. For Maya ZEC, send the memo text to `shieldedMemoAddress` (a 2nd shielded output) instead of OP_RETURN.
- **`cosmos_memo`** — put `memo` in the transaction memo field of a standard send to `inboundAddress`.

## Tracking

The committed route carries a top-level **`uuid`** — the tracking handle. Store it and track by it; the
server knows the provider and all swap details from the record. For DEX routes (`execution.method` of
`thorchain_deposit` or `signed_transaction`) also send your broadcast tx hash as `inboundTxHash` on
`/v2/track`; P2P/NEAR (`transfer`) need only the `uuid`.

```jsonc
{ "providers": ["BARTER"], …, "execution": { … }, "uuid": "b5b1b8c1-4a18-42a0-ab84-374d36f68f17" }
```

See [track.md](track.md) for the full request/response shapes.

## Rules

- Compare with `/rate`, then commit the winner with `/swap` — `/swap` names exactly one `provider` and requires `destinationAddress`.
- If you call `/swap` but never send funds, your fulfillment ratio drops and your rate limit is reduced.
- `signed_transaction` providers (1inch/Barter/Circle) require `sourceAddress` on `/swap`.
- A route's `execution.method` is fixed by the provider+chain; it never depends on whether you sent `sourceAddress`. Sending `sourceAddress` only adds an `unsignedTx` to `transfer`/`thorchain_deposit` routes.
- For P2P providers, lock a rate by passing the `rate.id` from a `/rate` route as `rateId` on `/swap` before `expiresAt`.
- Handle the `execution` branch for the route you chose and ignore the others — the fields you need are all and only in that branch.
