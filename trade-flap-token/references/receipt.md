# Receipt

Paths A and B (Portal) only. Path C uses `market-order list` terminal status in `references/graduated.md`.

A submitted hash is not a fill. If the receipt is not available yet, say the transaction is still processing. Do not report a fill, and do not send a second one.

For Binance Agentic Wallet, `PENDING_CONFIRMATION` with an empty hash is also not a fill. Follow `references/errors.md`: check `tx-lock`, then pending history. A `LOCKED` wallet does not get another trade.

Read the receipt for `$TX_HASH` (`eth_getTransactionReceipt`, or your library's equivalent).

About 30 seconds without a receipt means still processing. Do not invent a speed-up transaction on this Portal path.

## What must match

### The transaction is this order

The hash must come from this send: the hash the user pasted back from their own signer, the `execute` output, or a `tx-history --type pending` entry that appeared after this `execute`. A hash from an older trade is not this order, even when its event matches.

Read the transaction itself for `$TX_HASH` (`eth_getTransactionByHash`).

- The receipt block is after the quote block. If its timestamp is more than 60 seconds after `$QUOTE_TS`, the broadcast was late: do not send another trade; still judge this hash with the fill rules below, and tell the user it ran outside the confirmed window.
- `from` is the trader, `to` is Portal, `value` is the confirmed `value`, and `input` is exactly the confirmed calldata.

This applies to Binance Agentic Wallet too. Observed BAW `contract-call` fills were plain transactions: `from` the BAW address, `to` Portal, the confirmed calldata unchanged. If a BAW transaction does not match, do not report a fill. Say the outer transaction differs from the confirmed call, show both, and do not send another trade.

### The Portal event

Decode logs from Portal `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0`. `TokenBought` and `TokenSold` have no indexed fields, so a matching log has exactly one topic, the event hash below, and all seven fields are ABI-encoded in the log `data` in this order. If a future Portal version indexes those fields, every receipt will fail this check. Say that the event shape may have changed. Do not report a fill.

```text
TokenBought(uint256 ts, address token, address buyer, uint256 amount, uint256 eth, uint256 fee, uint256 postPrice)
  topic0 0xa800a2038683844fac66747f771bfdfae862eb28b16bcfa387afa9fbacce8ff7
TokenSold(uint256 ts, address token, address seller, uint256 amount, uint256 eth, uint256 fee, uint256 postPrice)
  topic0 0x03a4693e592f5e75dc7c136acb39b146d2b4966c0e509c34f362dee02b3b861a
```

Accept the receipt only when all of these are true:

- The transaction checks above pass.
- There is exactly one `TokenBought` on a buy, or one `TokenSold` on a sell, whose `token` is this order.
- `buyer` or `seller` is the trader. This holds on a BNB buy of an ERC-20-quoted token too: the hop inside Portal still records the wallet as `buyer`. Another buyer means the fill is not verified. Do not report success, and do not send a replacement buy.
- A buy compares `amount` with `minOutputAmount` and with the quote. `amount` is tokens received. A sell compares `eth` the same way.

`eth` and `fee` are in the curve's quote asset, not in what the user paid. On a native-quoted token that is BNB. On an ERC-20-quoted token it is that token's base units, also on a Pay-with-BNB buy: a 0.005 BNB buy logged `eth` 0.6644 BNCB. Report them with the quote asset's `decimals()` and symbol. `fee` is the Portal fee in the same unit as `eth` (observed 1% of `eth`); the token tax is separate.

| Comparison | What to tell the user |
|---|---|
| No matching event | Not a fill |
| More than one matching event | Do not guess. Show the logs |
| Received amount below `minOutputAmount` | The fill missed the confirmed floor |
| Received amount below the quote but at least the floor, no buy cap | Filled with slippage. Give both numbers and the difference |
| Buy quota was the cap | Capped fill. Unused input may have been refunded. Give the received amount and `remaining` |

On a capped buy, check the refund with balances, not with `TokenBought.eth`. On a BNB buy of an ERC-20-quoted token, never compare `eth` with the BNB spent: they are different assets. Compare the wallet's BNB balance before and after, minus gas.

Then read the trader's token balance. For BAW, `baw wallet balance --tokenAddress $TOKEN --binanceChainId 56` is enough. If the read fails, say the balance was not checked. Do not invent it.

After a sell, read what is left of the approval: `allowance($TRADER, Portal)` on `$TOKEN` (selector `0xdd62ed3e`).

An exact approve that the sell used up is 0: say "Portal allowance is 0, nothing to revoke." Only when it is above 0, say the user can revoke Portal in a new transaction with its own confirmation. Do not start that revoke.

Do not submit a replacement or a speed-up from this step.
