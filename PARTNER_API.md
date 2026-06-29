# USwap Partner API

Integration guide for partners (affiliates) embedding the USwap cross-chain swap aggregator.

USwap aggregates quotes across THORChain, Mayachain, 1inch, Barter, NEAR, Circle, and six P2P
exchanges (LetsExchange, StealthEx, Quickex, Swapuz, Exolix, CCE), and gives you one contract to
compare routes, execute a swap with the provider you pick, and track it to completion.

- **Base URL:** `https://swap-api.unstoppable.money`
- **Auth:** every request needs an `X-API-Key` header
- **Content type:** `application/json` for all `POST` bodies

```
POST https://swap-api.unstoppable.money/v2/rate
Content-Type: application/json
X-API-Key: <your-api-key>
```

---

## 1. Getting an API key

Register as a partner at **https://swap.unstoppable.money/affiliate**. You receive an API key bound to
your partner account, where you also set your affiliate revenue share. The key is passed on every
request via the `X-API-Key` header. Keep it server-side — it identifies you, carries your fee
configuration, and accrues your affiliate revenue share (see [§8](#8-affiliate-revenue-share)).

New accounts may require approval before they go active; until then requests return `403`.

Auth responses:

| HTTP | Meaning |
|---|---|
| 401 | `X-API-Key` missing or not recognized |
| 403 | Account is not active (contact us) |

---

## 2. The swap flow

USwap separates **pricing** from **committing** so you can compare freely without creating orders:

1. **Rate** — `POST /v2/rate` prices the swap across every eligible provider. Read-only, no order,
   no side effects. Returns a list of `routes` to compare.
2. **Swap** — `POST /v2/swap` commits against the **one** provider you chose. This creates the real
   order and returns a single executable route carrying an `execution` block and a top-level `uuid`.
3. **Send funds** — do exactly what the route's `execution` block says (sign a tx, transfer to a
   deposit address, or deposit to a vault with a memo). Switch on `execution.method`.
4. **Track** — store the route's `uuid` and POST `{ uuid }` to `/v2/track` until the status is
   terminal (`completed`, `refunded`, `failed`, or `expired`); for DEX swaps also send your broadcast tx hash.

```
/v2/rate  ──compare──▶  pick best route  ──▶  /v2/swap (one provider)  ──▶  send funds  ──▶  /v2/track (poll)
```

---

## 3. Asset ID format

Assets are identified by a chain-prefixed string:

```
CHAIN.SYMBOL
CHAIN.SYMBOL-CONTRACT_ADDRESS
```

| Asset | ID |
|---|---|
| Bitcoin | `BTC.BTC` |
| Ether | `ETH.ETH` |
| Solana | `SOL.SOL` |
| BNB (native) | `BSC.BNB` |
| ARB (native gas) | `ARB.ETH` |
| USDC on Ethereum | `ETH.USDC-0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| ERC-20 for BARTER / ONEINCH (chain-prefixed) | `ETH.0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
| ERC-20 for BARTER / ONEINCH (bare address) | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` + `chainId` |

For **BARTER** and **ONEINCH** (same-chain EVM aggregators), `sellAsset` / `buyAsset` may be given as:
- a chain-prefixed contract address (`ETH.0x…`), or
- a **bare contract address** (`0x…`) with **no chain prefix** — in which case you must also send the
  request's `chainId` field (e.g. `"1"` for Ethereum) so the server knows the chain. For the native gas
  asset, pass the sentinel `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` as a bare address.

Both assets must be on the same chain.

### THORChain Secured Assets

THORChain Secured Assets use a **dash** instead of a dot — `CHAIN-SYMBOL[-CONTRACT]`. They are
1:1-backed claims on an L1 asset, held inside THORChain, denominated in 8 decimals regardless of the
L1 native decimals.

| Asset | ID |
|---|---|
| Secured ETH | `ETH-ETH` |
| Secured BTC | `BTC-BTC` |
| Secured USDC on Ethereum | `ETH-USDC-0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |

Constraints when quoting these via THORCHAIN:
- **L1 → Secured:** `destinationAddress` must be a `thor1…` address.
- **Secured → L1 / Secured:** `sourceAddress` must be the `thor1…` holder. The deposit is a
  `MsgDeposit` on THORChain — **the server does not build this transaction; you construct and sign
  the `MsgDeposit` yourself** with the returned `memo`. The inbound fee is reported as a small fixed
  RUNE gas amount.

Trade Assets (`~`), synthetics (`/`), and derived assets (`THOR.X`) are **not** supported.

---

## 4. `POST /v2/rate` — compare routes

Read-only. Prices the swap across all eligible providers and returns `routes` to compare. Creates
nothing. No addresses are needed because nothing is built yet.

### Request

| Field | Type | Required | Description |
|---|---|---|---|
| `sellAsset` | string | yes | Asset to sell (see [§3](#3-asset-id-format)) |
| `buyAsset` | string | yes | Asset to buy |
| `sellAmount` | string | yes | Decimal string, e.g. `"0.1"` |
| `slippage` | number | yes | Tolerance, `0`–`99` |
| `chainId` | string | conditional | Numeric EVM chain id as a string (`"1"` Ethereum, `"56"` BSC, `"137"` Polygon, `"42161"` Arbitrum, `"10"` Optimism, `"8453"` Base, `"43114"` Avalanche). **Required** when a BARTER/ONEINCH asset is a bare contract address (no chain prefix). |
| `providers` | string[] | no | Narrow the fan-out to specific providers |
| `floating` | boolean | no | P2P only. Default `true` |
| `streamingInterval` / `streamingQuantity` | number | no | THORChain / Mayachain streaming swaps only |

### Response

```jsonc
{ "routes": [ Route, ... ], "providerErrors": [ { "provider": "QUICKEX", "error": "...", "errorCode": "amountOutOfRange", "minimumAmount": 0.01, "maximumAmount": 5 } ] }
```

Each `Route` (from `/rate`) carries **economics only** — no `execution`, no `uuid` (those appear
only after you commit with `/swap`):

```jsonc
{
  "providers": ["THORCHAIN"],            // providers in the route (one today)
  "sellAsset": "BTC.BTC", "sellAmount": "0.1",
  "buyAsset": "ETH.ETH", "expectedBuyAmount": "1.52",   // already net of all fees
  "minBuyAmount": "1.49",                // promised floor used to score the provider
  "fees": [
    { "type": "liquidity", "protocol": "THORCHAIN", "chain": "BTC", "asset": "BTC.BTC", "amount": "0.0003" }
  ],
  "estimatedTime": { "inbound": 600, "swap": 60, "outbound": 600, "total": 1260 },  // seconds
  "expiresAt": 1718700000000,            // P2P rate-lock expiry (epoch ms), if any
  "rate": { "id": "...", "floating": true },   // P2P rate lock, if lockable
  "amlPolicy": "good",
  "accuracy": { "matched": 98, "total": 100, "avgDeviation": -0.4 },
  "amlErrors": [{ "message": "...", "level": 2 }],   // only if the provider flagged AML issues
  "approvalSpender": "0x1111…",                       // EVM (1inch/Barter/Circle) only — ERC20 spender to approve before swapping
  "meta": { "thorchain": { "slippageBps": 12, "totalBps": 30 } }   // provider-namespaced extras (advanced)
}
```

**Pick the route with the best `expectedBuyAmount`** — it is already net of every fee. `minBuyAmount`
is the floor the swap is expected to deliver (used to score provider accuracy).

### Fee types

You don't need to sum `fees[]` to know the output (`expectedBuyAmount` already nets them), but the
breakdown is useful for showing costs to your user:

| `type` | What it covers |
|---|---|
| `service` | The aggregator's service fee. |
| `liquidity` | Provider / protocol costs: the underlying provider's cut and any relayer/forwarder fee. |
| `inbound` / `outbound` | Source / destination chain network (gas) fees. |
| `affiliate` | Your partner revenue share, carved from our service fee (see [§8](#8-affiliate-revenue-share)). Present only when your account has an affiliate share configured. |

Each entry is denominated in its own `asset` and carries the `protocol` (provider) and `chain` it
applies to.

---

## 5. `POST /v2/swap` — commit with one provider

Commits a quote against the **single** provider you name. Creates the real order and a swap record,
and returns one executable route with `execution` + a top-level `uuid` populated.

### Request

| Field | Type | Required | Description |
|---|---|---|---|
| `sellAsset` / `buyAsset` / `sellAmount` / `slippage` / `chainId` | | yes / conditional | Same as `/rate` (`chainId` required for bare BARTER/ONEINCH addresses) |
| `provider` | string | yes | The **single** provider to create the order with |
| `destinationAddress` | string | yes | Where the bought funds are sent |
| `refundAddress` | string | no | Refund address on failure (required by most P2P providers) |
| `sourceAddress` | string | conditional | The build signal — see below. **Required** for `signed_transaction` providers (1inch / Barter / Circle — i.e. a provider whose `executionType` is `signed_transaction`, from [§9](#9-supporting-endpoints)), since the tx needs a `from`. |
| `rateId` | string | no | P2P only. Lock a previously quoted `rate.id` from `/rate` (do it before `expiresAt`). |
| `floating` | boolean | no | P2P only. Default `true` |
| `streamingInterval` / `streamingQuantity` | number | no | THORChain / Mayachain only |

**`sourceAddress` is the build signal.** Supply it and the server returns a ready-to-sign
`unsignedTx` for chains it can build for (good for thin wallets). Omit it and you get only the raw
instructions and build the tx yourself (good for HD wallets that build a better tx, e.g. multi-UTXO
sweeps). It is **orthogonal** to the execution method — it only adds an optional `unsignedTx` to
`transfer` and `thorchain_deposit` routes. `signed_transaction` routes always carry their tx (the
swap *is* a contract call), so those providers require `sourceAddress`.

### Response

`/swap` returns the **executable route directly** (no `{ routes }` wrapper) — the same `Route` shape
as `/rate`, now with an `execution` block and a top-level `uuid` tracking handle:

```jsonc
{ "providers": ["QUICKEX"], "expectedBuyAmount": "1.52", /* … */, "execution": { /* … */ }, "uuid": "b5b1b8c1-…" }
```

On failure the body is the provider error with a matching HTTP status:

```jsonc
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

---

## 6. `execution` — how to send funds

The committed route carries exactly one `execution` object. **Switch on `execution.method`** and
handle only that branch; nothing else applies.

### `signed_transaction` — 1inch, Barter, Circle

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

- If `approval` is present, first call `approve(spender, amount)` on the token and wait for
  confirmation, then submit the tx.
- `transactions` is an array; submit each (length 1 for EVM providers today).
- `kind` tells you how to decode each tx: `evm` (object fields above), `psbt` / `solana` / `near` /
  `stellar` (base64 string under `psbt` / `message` / `tx` / `xdr`), `cosmos` / `ripple` / `ton` /
  `tron` (object under `tx`).

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
  "unsignedTx": { "kind": "ripple", "tx": { /* … */ } },              // only if you sent sourceAddress
  "qr": { "str": "xrp:r…?dt=123456789", "dataURL": "data:image/png;base64,…" }  // optional deposit QR
}
```

- **`attachment`** is an order identifier the provider uses to credit your funds. **If present, you
  must include it**, or the deposit is lost.
  - `{ "type": "destination_tag", "value": "…" }` — XRP destination tag.
  - `{ "type": "text", "value": "…" }` — memo field (RUNE / GAIA / TON / NEAR).
- **`unsignedTx`** appears only when you sent `sourceAddress` and the chain is buildable. Sign and
  broadcast it, *or* ignore it and build your own transfer to `depositAddress`.
- **`qr`** is an optional server-rendered deposit QR (`str` = encoded payload, `dataURL` = image).

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
- **`evm_contract_call`** — the memo is encoded in a router call. Submit `unsignedTx` (an `evm` tx).
  `router` is the contract; if `approval` is present, approve first. A plain transfer to
  `inboundAddress` will **not** work.
- **`utxo_op_return`** — attach `memo` as an `OP_RETURN` output alongside the `inboundAddress` output.
  Either submit `unsignedTx` (a single-address PSBT) or build the multi-output tx yourself. For Maya
  ZEC, send the memo text to `shieldedMemoAddress` (a 2nd shielded output) instead of OP_RETURN.
- **`cosmos_memo`** — put `memo` in the transaction memo field of a standard send to `inboundAddress`.

---

## 7. `POST /v2/track` — track a swap

Track by the committed route's top-level **`uuid`**. The server tracks by it alone — it already knows
the provider and every swap detail from the record, so you never reconstruct the request; you just send
the `uuid` (plus, for DEX swaps, your broadcast tx hash).

### Request

| Field | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | yes | The committed route's `uuid` |
| `inboundTxHash` | string | conditional | Your broadcast tx hash. **Required for DEX swaps** (THORChain, Mayachain, 1inch, Barter, Circle — any route whose `execution.method` was `thorchain_deposit` or `signed_transaction`) once you have it; the server finds your tx on-chain by it. P2P / NEAR don't need it — the provider watches its own deposit address. |

- **P2P (LetsExchange, StealthEx, Quickex, Swapuz, Exolix, CCE) and NEAR:**

  ```json
  { "uuid": "b5b1b8c1-…" }
  ```

- **DEX swaps:** after you broadcast, add your tx hash:

  ```json
  { "uuid": "b5b1b8c1-…", "inboundTxHash": "0xabc123…" }
  ```

The server remembers the hash for subsequent polls.

### Response

```jsonc
{
  "status": "swapping",
  "providers": ["THORCHAIN"],          // provider(s) handling the swap (one today)
  "fromAsset": "BTC.BTC", "fromAmount": "0.1", "fromAddress": "bc1q…",
  "toAsset": "ETH.ETH", "toAmount": "1.52", "toAddress": "0x…",
  "legs": [
    {
      "chainId": "bitcoin",
      "hash": "abc123…",
      "type": "inbound",
      "status": "completed",
      "fromAsset": "BTC.BTC", "fromAmount": "0.1", "fromAddress": "bc1q…",
      "toAsset": "ETH.ETH", "toAmount": "1.52", "toAddress": "0x…"
    }
  ],
  "meta": { "sellAmountUsd": "6500.00" }   // optional; `pauseReason` appears here on action_required
}
```

`legs[]` renders multi-stage progress (deposit → swap → outbound); each provider's tracker fills the
legs over time.

- `409` `{ "message": "Swap not trackable yet", "uuid": "…" }` — recorded but not yet trackable (e.g.
  a DEX swap whose `inboundTxHash` you haven't sent). Send the hash, then poll again.
- `404` — the `uuid` is unknown.

### Statuses

| Status | Meaning |
|---|---|
| `not_started` | Funds not yet received |
| `pending` | Funds received, waiting to process |
| `swapping` | Swap in progress |
| `action_required` | Funds reached the provider but the swap is stuck and won't auto-resolve — the user must contact the provider. See below. |
| `completed` | Swap finished, funds delivered |
| `refunded` | Swap failed, funds returned |
| `failed` | Swap failed, no refund |
| `expired` | Timed out — no deposit was detected within ~72h, so tracking stopped. The user likely never sent funds (or sent too late). Treat as terminal; a late deposit may still be resolved server-side. |
| `unknown` | Status could not be determined |

**Terminal** (stop polling): `completed`, `refunded`, `failed`, `expired`. `action_required` is **non-terminal** —
keep polling; the provider may still resolve it after manual intervention.

Poll every 1–2 minutes. THORChain swaps typically complete in 5–20 minutes; P2P swaps in 10–60 minutes.

### Action required

When `status === "action_required"`, the response includes `meta.pauseReason`:

| Reason | Cause |
|---|---|
| `aml` | Provider's AML check blocked the deposit |
| `kyc_required` | Provider requires KYC submission before withdrawal |
| `frozen` | Provider froze the order pending review |
| `overdue_with_funds` | Order/rate window missed after the deposit arrived |
| `provider_error` | Provider-side error after a deposit (catch-all) |
| `manual_review` | Provider's generic "needs human attention" state |

To help the user recover funds, surface the provider's `contacts` from `GET /v2/providers` alongside
`meta.pauseReason`. The user contacts the provider directly; once resolved, the next poll reports a
terminal status.

---

## 8. Affiliate revenue share

As a partner you earn a revenue share automatically — there is **nothing to send per request.**

- Your share is a percentage of **our service fee** on each swap routed through your API key,
  which you set yourself at https://swap.unstoppable.money/affiliate (in basis points; e.g.
  2000 = 20% of the service fee). Changes take effect on your next quote.
- It is computed and recorded server-side at quote time on every committed (`/v2/swap`) swap, across
  all providers. The total fee shown to your user is unchanged — your share is carved out of our cut,
  not added on top.
- Earnings accrue against completed swaps and are paid out per your agreement. You don't pass any
  affiliate parameter to collect this; it is keyed off your API key.

> `/v2` takes **no** request-level affiliate parameter — your share is determined entirely by your API
> key and the percentage you set on the partner page. There is nothing to add to a request.

---

## 9. Supporting endpoints

All require the `X-API-Key` header.

### `GET /v2/providers`

Lists all providers with `amlPolicy`, `amlPolicyDescription`, `contacts` (`{ email?, telegram?, … }`),
`supportedChainIds`, `suspended`, an `accuracy` block (`{ matched, total, avgDeviation }`), and
`executionType`.

**`executionType`** is the single execution method a provider commits to — one of `transfer`,
`signed_transaction`, or `thorchain_deposit`. **If you have no wallet integration** (you can only relay
deposit details to your user, not sign or build a transaction), restrict yourself to `transfer`
providers: the user simply sends funds to a deposit address. `signed_transaction` and `thorchain_deposit`
require a wallet to sign/build the inbound tx. Filter on this **before quoting** so you only offer
providers you can actually fulfill.

### `GET /v2/tokens?provider=THORCHAIN`

Returns the supported token list for a provider. Supports `ETag` / `If-None-Match` (304). **BARTER and
ONEINCH do not expose a token list** — use contract addresses directly.

### `GET /v2/inbound_addresses`

Returns the current THORChain/Mayachain inbound (vault) addresses.

---

## 10. Providers & AML policy

| Provider | Type | AML Policy | Notes |
|---|---|---|---|
| THORCHAIN | DEX | `excellent` | BTC, ETH, AVAX, BCH, LTC, DOGE, GAIA, BSC, and more |
| MAYACHAIN | DEX | `excellent` | BTC, ETH, DASH, KUJI, THOR, ARB, and more |
| ONEINCH | DEX aggregator | `excellent` | Same-chain EVM: ETH, BSC, ARB, OP, AVAX, POL, BASE |
| BARTER | DEX aggregator | `excellent` | Same-chain EVM |
| CIRCLE | Bridge | `excellent` | USDC-only cross-chain bridge (CCTPv2). Source and destination chains must differ. |
| NEAR | DEX | `fair` | NEAR ecosystem + cross-chain via 1Click |
| LETSEXCHANGE | P2P | `good` | Wide cross-chain coverage |
| STEALTHEX | P2P | `fair` | Wide cross-chain coverage |
| QUICKEX | P2P | `good` | Wide cross-chain coverage. Supports AML address precheck. |
| SWAPUZ | P2P | `good` | Wide cross-chain coverage |
| EXOLIX | P2P | `good` | Wide cross-chain coverage |
| CCE | P2P | `good` | Wide cross-chain coverage |

**DEX** = decentralized, on-chain execution. **P2P** = centralized order matching; the provider
watches its own deposit address, so tracking needs only the route's `uuid` (no broadcast tx hash).

The `amlPolicy` describes how a provider handles AML checks. It is returned on every quote route and on
`GET /v2/providers`. Use it to set user expectations before a swap.

| Policy | Meaning |
|---|---|
| `excellent` | Direct on-chain execution. No provider checks or freezes. Automatic refunds if the swap fails. |
| `good` | Provider checks transactions automatically before completion. If issues are detected, the swap is rejected and funds are refunded. |
| `fair` | Additional verification may be required for some transactions. If issues are detected, funds are usually refunded automatically. |

---

## 11. Error reference

| HTTP | Meaning |
|---|---|
| 400 | Validation error / unsupported pair or asset / amount out of range |
| 401 | Missing or invalid `X-API-Key` |
| 403 | Account is not active |
| 404 | No route, or unknown `uuid` on `/v2/track` |
| 409 | Rate expired (re-quote), or swap not trackable yet |
| 502 | Upstream provider failure (network / bad response) |
| 503 | Provider suspended |
| 504 | Provider timed out |

Provider error bodies carry `{ error, provider, errorCode?, minimumAmount?, maximumAmount? }`.
