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
| sourceAddress | string | conditional | The build signal — see below. Required for `signed_transaction` providers (1inch/Barter/Circle/Jupiter/LI.FI/Soroswap/Aquarius/Stellar DEX/Axelar ITS) since the tx needs a `from` / signer, and for STELLARBROKER, whose session trades from that account. For STELLARBROKER and AQUARIUS, `destinationAddress` must **equal** it — both settle on the trader's own account. |
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
  "minBuyAmount": "1.49",              // ENFORCED floor, or null when the amount is only an estimate
  "fees": [{ "type": "liquidity", "protocol": "THORCHAIN", "chain": "BTC", "asset": "BTC.BTC", "amount": "0.0003" }],
  "estimatedTime": { "inbound": 600, "swap": 60, "outbound": 600, "total": 1260 },
  "expiresAt": 1718700000000,          // P2P rate-lock expiry (epoch ms), if any
  "rate": { "id": "...", "floating": true },  // P2P rate lock, if lockable
  "amlPolicy": "good",
  "accuracy": { "matched": 98, "total": 100, "avgDeviation": -0.4 },
  "amlErrors": [{ "message": "...", "level": 2 }],   // present only if the provider flagged AML issues
  "approvalSpender": "0x1111…",                       // token sells needing an allowance — EVM ERC20 (1inch/Barter/Circle/LI.FI) or LI.FI Tron TRC20 (base58 spender) — approve before swapping
  "meta": { "thorchain": { "slippageBps": 12, "totalBps": 30 } }  // provider-namespaced extras (advanced); not needed to execute
}
```

`expectedBuyAmount` is already net of all fees. Pick the route with the best `expectedBuyAmount`.

`minBuyAmount` (`string | null`) is the **guaranteed minimum**, present only when something enforces
it (on-chain minReturn for 1inch/Barter, Jupiter's on-chain slippage threshold, LI.FI's on-chain
minimum-received, the on-chain floor built into the Stellar tx for Soroswap/Aquarius/Stellar DEX, the
1:1 bridge amount for Axelar ITS, THORChain/Mayachain memo limit, NEAR intent minimum, or a locked
fixed-rate P2P quote). It is `null` when nothing enforces the number: floating-rate P2P routes
(LetsExchange, StealthEx, Quickex, Swapuz, Exolix, CCE, Pegasus), where the quote is an **estimate
re-priced when the deposit arrives**; **STELLARBROKER**, which re-quotes live inside its execution
session; and **CIRCLE**, whose forwarder-fee tier can shift. Tell the user the received amount can
differ from the quote in either direction.

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

Exactly one of four shapes. Switch on `method`.

### `signed_transaction` — 1inch, Barter, Circle, Jupiter, LI.FI, Soroswap, Aquarius, Stellar DEX, Axelar ITS

The swap is a contract call. **Sign and broadcast the provided transaction(s) verbatim.** There is no
deposit address; do not construct your own tx. `chain` is the **source** chain and decides the tx shape
(`evm` on EVM chains, `solana` on Solana, `tron` on Tron, `stellar` on Stellar). LI.FI and Axelar ITS are
cross-chain — the source chain picks the shape and the bought asset lands on its own chain automatically
(one signature, no second tx).

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
- `transactions` is an array; today it is length 1 for every provider — submit each.
- **Jupiter (Solana):** the entry is `{ "kind": "solana", "message": "<base64>" }` — deserialize as a `VersionedTransaction`, sign, broadcast **promptly** (the embedded blockhash expires in ~a minute; re-quote if it lapses). The quoted minimum is enforced on-chain — the tx reverts instead of under-delivering. No approval step on Solana. Third-party `destinationAddress` works for SPL outputs (recipient token account auto-created, signer pays ~0.002 SOL rent); native-SOL output goes to the signer.
- **LI.FI (cross-chain):** sign **one** tx on the source chain — `evm` when selling an EVM asset, `solana` (base64 `VersionedTransaction`) when selling on Solana, `tron` when selling on Tron. For **Tron**, `tx` is a TronGrid **created-transaction** envelope `{ visible, txID, raw_data, raw_data_hex }` — sign its `txID` and broadcast `{ …, signature }` to a full node; approve first if `approval` is present (TRC-20, a base58 `spender`). The bridge delivers the bought asset to `destinationAddress` on the destination chain automatically (no second signature). Broadcast promptly (source txs expire); for EVM sells approve first if `approval` is present. The minimum-received is enforced on-chain (the tx reverts below it). Track by `uuid` + your source-chain `inboundTxHash`.
- **Soroswap / Aquarius / Stellar DEX:** the entry is `{ "kind": "stellar", "xdr": "<base64>" }` — decode as a `TransactionEnvelope`, sign with the `sourceAddress` key, submit to Horizon. Broadcast **promptly**: the envelope carries its own time bounds (and, for the Soroban routes, a simulated resource footprint) and stops being valid once they lapse — re-quote if that happens. The output floor is enforced on-chain, so a price move past the slippage makes the tx fail rather than under-deliver. No approval step on Stellar. Do **not** re-sequence, re-fee, or rebuild the envelope — the Soroban routes were simulated for exactly this transaction.
- **Axelar ITS (bridge):** one tx on the source chain, no second signature, same token 1:1 on the other side. Selling on **Stellar** → a `stellar` Soroban envelope (gas prepayment included); selling on **Ethereum** → an `evm` tx calling `interchainTransfer` **on the token contract itself** with the gas prepayment as `value`, and **no approval step** (the token burns from the caller), so no `approval` object appears. Delivery ~0.5–3 min from Stellar, ~17 min from Ethereum. Track by `uuid` + the source `inboundTxHash`.
- `kind` decodes the tx: `evm` (object), `psbt`/`solana`/`near`/`stellar` (base64 string under `psbt`/`message`/`tx`/`xdr`), `cosmos`/`ripple`/`ton`/`tron` (object under `tx`; for LI.FI Tron a TronGrid created-transaction envelope).

### `stellar_broker` — StellarBroker

**No transaction is returned.** Execution is an interactive **WebSocket session** you run yourself: the
broker builds each tx, streams it to you, you sign it, and **the broker submits it**. The route carries
the session parameters instead.

```jsonc
{
  "method": "stellar_broker",
  "chain": "XLM",
  "sellingAsset": "XLM",                    // SB wire form: 'XLM' native, 'CODE-GISSUER…' classic
  "buyingAsset": "USDC-GA5ZSEJY…",
  "sellingAmount": "100.0000000",
  "slippageTolerance": 0.01,                 // FRACTION (0–0.5), not bps
  "partnerKey": "…"                          // pass as ?partner= on the WS URL
}
```

Connect to `wss://api.stellar.broker/ws?partner=<partnerKey>` and drive the session with these values.

- **Skip this provider unless you can hold a live WebSocket session.** There is no tx to hand the user.
- **The committed numbers are a snapshot** — SB re-quotes live in-session, which is why `minBuyAmount` is `null` here. `slippageTolerance` is the only bound.
- **Validate every transaction before signing it:** each operation should spend only the **selling asset** from the trader's account, and no single tx should debit more than the quoted `sellingAmount` plus a little headroom for the broker's fee leg. Cap how many txs you will sign in one session.
- **A trade may span several transactions** (SB chunks large orders). Report a submitted tx hash as `inboundTxHash`; if there were several, the reported one yields a lower-bound amount.
- `destinationAddress` must equal `sourceAddress`.

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

- **`attachment`** — a secondary identifier the receiving side uses to match your deposit to this order. Present **only** on `transfer` routes and only for chains that need one (absent ⇒ send a plain transfer). **If present, include `value` exactly (verbatim)** or the funds arrive unmatched and are typically unrecoverable. The `type` says **which transaction field** it goes in — the two are **not** interchangeable:
  - `{ "type": "destination_tag", "value": "…" }` — a numeric tag → the transaction's dedicated **destination-tag** field (a distinct protocol field), **not** a memo. (e.g. XRP.)
  - `{ "type": "text", "value": "…" }` — a free-form string → the transaction's **memo / comment** field. (e.g. Cosmos-, TON-, RUNE-style memo chains.)
  - A `destination_tag` value placed in a memo field, or a `text` value in the tag field, is a mismatched deposit and loses the funds. If you submit the route's `unsignedTx`, the attachment is already embedded — do not re-add it.
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
`thorchain_deposit`, `signed_transaction`, or `stellar_broker`) also send your broadcast tx hash as
`inboundTxHash` on `/v2/track`; P2P/NEAR (`transfer`) need only the `uuid`.

```jsonc
{ "providers": ["BARTER"], …, "execution": { … }, "uuid": "b5b1b8c1-4a18-42a0-ab84-374d36f68f17" }
```

See [track.md](track.md) for the full request/response shapes.

## Rules

- Compare with `/rate`, then commit the winner with `/swap` — `/swap` names exactly one `provider` and requires `destinationAddress`.
- If you call `/swap` but never send funds, your fulfillment ratio drops and your rate limit is reduced.
- `signed_transaction` providers (1inch/Barter/Circle/Jupiter/LI.FI/Soroswap/Aquarius/Stellar DEX/Axelar ITS) and STELLARBROKER require `sourceAddress` on `/swap`.
- STELLARBROKER and AQUARIUS require `destinationAddress == sourceAddress`; a Stellar classic buy also requires the recipient's trustline to exist first, or `/swap` returns `400`.
- A route's `execution.method` is fixed by the provider+chain; it never depends on whether you sent `sourceAddress`. Sending `sourceAddress` only adds an `unsignedTx` to `transfer`/`thorchain_deposit` routes.
- For P2P providers, lock a rate by passing the `rate.id` from a `/rate` route as `rateId` on `/swap` before `expiresAt`.
- Handle the `execution` branch for the route you chose and ignore the others — the fields you need are all and only in that branch.
