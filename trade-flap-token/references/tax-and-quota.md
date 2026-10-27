# Tax, audit, and buy quota

## Tax

`buyTaxRate` and `sellTaxRate` from `getTokenV8Safe` are basis points. 100 bps is 1%. The two rates can differ, and either one can be 0.

Show them on two lines. On a buy, still show the sell tax. Do not add the rates together. `quoteExactInput` is already net of tax and Portal fee (a real 3%-tax sell filled exactly at the quote). Do not subtract the tax again, and do not add a protocol fee back onto it to recompute `minOutputAmount`.

| bps | What to say |
|---|---|
| 0–500 | No extra warning beyond the raw bps |
| 501–1000 | Warn. 1000 bps is 10% |
| above 1000 | Say the tax is high, and repeat the raw bps |

A high tax is a warning. It is not a reason to edit the calldata, and it is not a silent skip. A sell tax above 500 bps must appear in the buy confirmation.

## Audit

Follow `references/safety.md`. Try the audit on a buy; if unavailable, warn in the trade summary and continue — do not add a separate `继续` round.

## Buy quota

Read this for a **buy** on path A or B (inner market). Skip it for a sell. Skip it on path C (graduated); market-order has no Portal quota.

Two Portal views (types in `references/interface.md`):

| Call | Selector | Returns |
|---|---|---|
| `maxBuyPerOrigin(token)` | `0x4b02c8e9` | `(uint16 bps, uint256 maxBuyAmount)` |
| `buyQuotaOf(token, $TRADER)` | `0x929b098c` | `(uint256 bought, uint256 remaining)` |

`buyQuotaOf` returns `(bought, remaining)` in token base units. `$TRADER` is the address that will send the transaction. Quota is keyed by `tx.origin`. A sell does not restore a non-gated holding-style view the same way; on Agent ID-gated tokens, sells reduce `netBought` (see below). A Binance Agentic Wallet `contract-call` is sent from the BAW address itself (observed in real fills), so `buyQuotaOf(token, <BAW address>)` is this user's quota.

**No cap wins.** There is no cap when any of these is true: `bps` is 0, `maxBuyAmount` is 0, or `remaining` is `2^256 - 1`. A `remaining` of 0 does not override that. Do not print `2^256 - 1` as tokens left. Never hardcode "2%" — Agent ID-gated launches often use 200 bps, but the live `maxBuyPerOrigin` is authoritative for every token.

**Gated vs non-gated (when a cap exists):**

- Non-gated: `bought` tracks holding-style usage (balance-based). Transfers move the live balance component.
- Agent ID-gated: `bought` is `max(netBought, balanceOf(origin))`. Curve sells reduce `netBought` (clamped at zero); transferring tokens out does not free `netBought`. Path B sells are **not** stopped by buy-cap remaining; only the NFT check applies (`references/agent-id-gate.md`).

Only when a cap exists:

- `remaining` of 0: this trader's inner-market buy quota is used up. Stop the **buy**. Do not shrink the order or split it.
- The `quoteExactInput` result is larger than `remaining`: the user may not get the full quote. The first simulation in `references/swap.md` uses `remaining` as the floor basis. If that simulation returns more than `remaining`, the quota view did not reduce the swap: the floor moves up to the simulated output, and the swap is simulated again. If it returns no more than `remaining`, the floor stays on `remaining`. Say unused input may be refunded. Check the receipt before calling it a full fill.

Going over the quota does not always revert. That is why the simulation decides the final floor. Do not lower the floor and then skip the simulation.
