# Quote

`quoteExactInput` is not a `view` function (it is `nonpayable`). Read it with an `eth_call`. Never send it as a transaction.

Pin the quote to one block and record that block's timestamp:

1. Read the latest block number → `$QUOTE_BLOCK`.
2. Read block `$QUOTE_BLOCK` and take its `timestamp` → `$QUOTE_TS`.
3. `eth_call` Portal at block `$QUOTE_BLOCK`, with `from` set to `$TRADER` and `value` 0:

   ```text
   quoteExactInput((address inputToken, address outputToken, uint256 inputAmount)) returns (uint256 outputAmount)
   selector 0xfc847c2b
   argument: ($INPUT_TOKEN, $OUTPUT_TOKEN, $INPUT_AMOUNT)
   ```

Illustration (any equivalent eth_call works — see `references/interface.md`):

```typescript
const QUOTE_BLOCK = await client.getBlockNumber();
const QUOTE_TS = (await client.getBlock({ blockNumber: QUOTE_BLOCK })).timestamp;
const quote = await ethCall({
  to: PORTAL, abi: PORTAL_ABI, functionName: "quoteExactInput",
  args: [{ inputToken: INPUT_TOKEN, outputToken: OUTPUT_TOKEN, inputAmount: INPUT_AMOUNT }],
  from: TRADER, blockNumber: QUOTE_BLOCK,
});
```

`$TRADER` is the address that will send the swap. Show `$QUOTE_BLOCK` in the confirmation. The freshness window is 60 seconds of chain time after `$QUOTE_TS`, measured with the latest block timestamp, not the chat clock.

**Timing for BAW Portal trades:** do not open the confirmation until `contract-call preview` has succeeded. Then set `$QUOTE_TS` to the latest block timestamp at preview success (keep `$QUOTE_BLOCK` from this quote pass). That way prep time does not burn the 60s confirm window. In the summary, say the user should confirm within ~50s.

If the user confirms after that window, a preview expires, or you have not yet broadcast when the window ends: that confirmation is void — do **not** send. Silently quote again, simulate again, preview again, and present **one** new summary + chips. Do not ask whether to retry. If a transaction was already broadcast late, do not send a second one; judge the existing hash with `references/receipt.md` (it may note the late window, but still applies the fill rules).

Paths A/B only. Path C uses `baw market-order quote` in `references/graduated.md`.

`quoteExactInput` does **not** check Agent ID NFT ownership. On path B, finish `references/agent-id-gate.md` before this file. A quote for a non-holder is not permission to trade.

After an approve receipt, this file is not the last step. Quote here, then go to `references/swap.md` "Floor" and "Simulate". Do not confirm the swap on this quote alone.

The result is a single `uint256` in the output token's base units. Keep it as an integer; if your tool also prints a display form (for example `cast` prints `1710194868288753060294554 [1.71e24]`), use only the integer.

## Which tokens

| Order | `inputToken` | `outputToken` | `inputAmount` |
|---|---|---|---|
| Buy with BNB | zero address | the Flap token | BNB amount in wei (18 decimals) |
| Buy with the quote token | quote token | the Flap token | amount in the quote token's own decimals |
| Sell | the Flap token | quote token, or the zero address when the quote is BNB | token amount in the token's own decimals |

The zero address is `0x0000000000000000000000000000000000000000`.

Buy with BNB uses this same call whether the curve's quote asset is BNB or an ERC-20. The ERC-20 case is allowed only when `nativeToQuoteSwapEnabled` is true. Do not quote a quote-token purchase first when the user has already said pay with BNB.

A human amount is a plain decimal. Reject scientific notation. Resolve ERC-20 decimals with `decimals()` on that token. Read `decimals()` on the Flap token too. Do not assume 18 for either side when the chain says otherwise.

A result of 0 stops the order.

## Balance

Read the real balances. A state override is not a balance.

- Buy with BNB: the BNB balance must cover `inputAmount` plus the `0.002` BNB gas reserve from `references/preflight.md`. If it does not, stop.
- Buy with the quote token: the quote-token balance must cover `inputAmount`. If it does not, and `nativeToQuoteSwapEnabled` is true, ask once whether to pay with BNB. Wait for a BNB amount. Do not preview a quote-token swap, and do not buy the quote token as a first hop. If the flag is false, stop.
- Sell: the token balance must cover the amount. `all` means that balance, then the circulating-supply check below.

## Sell size

The amount must be greater than 0, no greater than the trader's balance, and no greater than `circulatingSupply`. A larger amount can revert with `Panic(0x11)`. Ask for a smaller size. The token can still be sold.

## Slippage

Default slippage is 1500 bps (15%). Use another value only when the user states one in this conversation with a unit: `5%` is 500 bps, `500 bps` is 500 bps. A bare `5` is ambiguous: ask. The allowed range is 0 through 9999 bps. Never raise it to force a fill.

```text
minOutputAmount = basis * (10000 - slippageBps) / 10000
```

Integer division rounds down. Use integer math (Python `int`, JavaScript `BigInt`), not floats and not shell `$(( ))`. If the result is 0, stop.

`basis` is chosen in `references/swap.md` "Floor": the `quoteExactInput` result, `remaining` when a buy cap is below the quote, or the simulated output when the swap ignores that cap. Do not replace the quote with a hand-computed curve, tax, or fee. Show the tax rates beside it.

When slippage is still 15%, tell the user: "Default 15%. This call has no MEV protection. A shallow pool can be sandwiched."
