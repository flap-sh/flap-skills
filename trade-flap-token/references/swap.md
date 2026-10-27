# Swap

Paths A and B only (inner market). Path C is `references/graduated.md`.

Every buy and sell here is Portal `swapExactInput`. The selector is `0xef7ec2e7`. `permitData` is `0x`. Do not build `swapExactInputV3`. Do not pass a fixed gas limit. The website's old `600000` gas figure is not an argument to copy. Function signatures, selectors, and errors are in `references/interface.md`.

Path B: signer is BAW only. Do not hand the transaction to any other signer. Approves and swaps use `baw contract-call` preview/execute.

Sections run in this order: approve check, simulation, calldata, confirmation, send. Every transaction is sent in the last section, "Send". Do not send one before its own confirmation.

## Fields

```text
swapExactInput((address inputToken, address outputToken, uint256 inputAmount, uint256 minOutputAmount, bytes permitData) params) payable returns (uint256 outputAmount)
```

| Field | Buy with BNB | Buy with quote token | Sell |
|---|---|---|---|
| `to` | Portal | Portal | Portal |
| `inputToken` | zero address | quote token | Flap token |
| `outputToken` | Flap token | Flap token | quote token, or zero address if the quote is BNB |
| `inputAmount` | BNB wei | quote-token base units | token base units |
| `minOutputAmount` | see "Floor" below | see "Floor" below | see "Floor" below |
| `permitData` | `0x` | `0x` | `0x` |
| `value` | same as `inputAmount` | `0` | `0` |

The zero address is `0x0000000000000000000000000000000000000000`.

Buy with BNB is this one call for both a native-quoted token and an ERC-20-quoted token whose `nativeToQuoteSwapEnabled` is true. Portal converts BNB internally. Do not buy the quote token first, and do not approve it. A later sell still returns that quote token, not BNB. Say that in the confirmation.

## Approve check, only for an ERC-20 input

BNB buys skip this section.

Read `allowance($TRADER, Portal)` on `$INPUT_TOKEN` (selector `0xdd62ed3e`, returns `uint256`).

- At least `inputAmount`: no approve. Go to "Simulate".
- Already `type(uint256).max`: say the approval is already unlimited. Do not increase it. Go to "Simulate".
- Below `inputAmount`: the next transaction is only an approve of exactly `inputAmount`. Do not simulate the swap yet, and do not attach it to the same confirmation.

Simulate the approve before asking: an `eth_call` to `$INPUT_TOKEN` of `approve(Portal, $INPUT_AMOUNT)` (selector `0x095ea7b3`, returns `bool`) with `from` = `$TRADER`.

If it reverts while the current allowance is above 0, the token may require a reset to 0 first (USDT-style). Then the next transaction is `approve(Portal, 0)` alone, with its own confirmation and its own receipt. After that, simulate the exact approve again. Do not reset an allowance that is already 0, and do not reset without that revert. Any other revert stops the order.

The approve confirmation shows: the token being approved (full address and symbol), the spender (Portal, full address), the amount in base units and in human units, and the sentence "Exact amount, not unlimited." For Binance Agentic Wallet, run the approve preview from "Send" first and show its fields too. Then go to "Confirm". After the approve is sent, wait for a receipt with status 1, read `allowance` again, and confirm it is at least `inputAmount`. Then leave this file: read `references/quote.md`, quote again, come back to "Floor" and "Simulate", and simulate the swap on that new quote. Do not open "Confirm" for the swap until that second simulation succeeds. A quote is not a simulation. On a sell, reuse the approved amount. Do not pass "all" again: a reflection token's balance can grow past the approval.

## Floor

Whether a buy cap exists is defined only in `references/tax-and-quota.md` ("No cap wins", including `remaining == 2^256 - 1`). Decide that first. The table below uses that result; do not treat a max-uint `remaining` as a real cap.

Do all math on integers (Python `int` or JavaScript `BigInt`), never floats, and never shell `$(( ))`, which overflows. If your tool prints a display form next to an integer, use only the integer.

```text
minOutputAmount = basis * (10000 - slippageBps) / 10000   (round down)
```

Pick the first `basis` this way:

| Case | First `basis` |
|---|---|
| No buy cap (per `tax-and-quota.md`), or a sell | the `quoteExactInput` result |
| A buy cap exists and the quote is no larger than `remaining` | the `quoteExactInput` result |
| A buy cap exists and the quote is larger than `remaining` | `remaining` |

"Buy cap" is defined in `references/tax-and-quota.md`. If `minOutputAmount` is 0, stop.

## Simulate

Run the exact swap as an `eth_call`: `to` Portal, `data` the `swapExactInput` calldata built from the fields above, `from` = `$TRADER`, `value` = the real `value`. Do not override balance or allowance. The trader must already hold the input and, for an ERC-20 input, the allowance.

Illustration (any equivalent eth_call works — see `references/interface.md`):

```typescript
const simulatedOut = await ethCall({
  to: PORTAL, abi: PORTAL_ABI, functionName: "swapExactInput",
  args: [{ inputToken: INPUT_TOKEN, outputToken: OUTPUT_TOKEN, inputAmount: INPUT_AMOUNT,
           minOutputAmount: MIN_OUT, permitData: "0x" }],
  from: TRADER, value: VALUE,
});
```

A revert stops the order. Do not widen the floor, and do not show a send summary. Show the revert reason (decoded against the errors in `references/interface.md` when the selector matches), and wait for the user to change the order or the slippage.

The returned `uint256` is the simulated output. Then:

- First `basis` was `remaining`, and the simulated output is greater than `remaining`: the swap did not apply the quota cap. Set `basis` to the simulated output, recompute `minOutputAmount`, and simulate once more with that floor. A revert stops the order. Tell the user the quota view did not reduce the swap.
- First `basis` was `remaining`, and the simulated output is no greater than `remaining`: keep the floor. Tell the user the buy is capped by the quota, that they get at most `remaining`, and that unused input may be refunded.
- First `basis` was the quote: keep the floor. If the simulated output is below the quote, show both numbers.

A successful simulation is not permission to send.

## Calldata

Build the calldata from the final fields with any ABI encoder. The same hex is what the user confirms and what is sent.

**Swap** — `swapExactInput` with a single tuple argument:

```text
selector 0xef7ec2e7
args     (inputToken, outputToken, inputAmount, minOutputAmount, permitData = 0x)
```

**Approve** — ERC-20 `approve` on the input token:

```text
selector 0x095ea7b3
args     (spender = Portal 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0, amount = inputAmount)
```

Illustration:

```typescript
import { encodeFunctionData } from "viem";

const swapData = encodeFunctionData({
  abi: PORTAL_ABI, functionName: "swapExactInput",
  args: [{ inputToken: INPUT_TOKEN, outputToken: OUTPUT_TOKEN, inputAmount: INPUT_AMOUNT,
           minOutputAmount: MIN_OUT, permitData: "0x" }],
});

const approveData = encodeFunctionData({
  abi: ERC20_ABI, functionName: "approve", args: [PORTAL, INPUT_AMOUNT],
});
```

Checks before using it:

- The swap calldata starts with `0xef7ec2e7`.
- The approve calldata starts with `0x095ea7b3`, and its amount is not `type(uint256).max`.
- Decoding the calldata back gives exactly the fields in the summary.

## Confirm

Do not open this section for a swap until "Simulate" has succeeded on the current quote. After an approve, that means the post-approve quote and the second simulation, not the pre-approve numbers.

Each transaction gets its own summary and its own confirmation: one for the approve, one for the swap. No other gates (no audit `继续`).

Emit **one** user-facing confirmation message per transaction. Do not split "here is the quote" and "now please confirm" across turns once preview is ready.

The swap summary shows `to`, `value`, every `swapExactInput` argument, the calldata, the quote, the simulated output, `minOutputAmount` and its basis, slippage, both tax rates, the buy quota, `version()`, the quote block, the audit status (or `audit: unavailable`), and a **confirm within ~50s** line. Path B also includes the Agent ID lines from `references/agent-id-gate.md`. For a 15% default, include the MEV sentence from `references/quote.md`.

For Binance Agentic Wallet, run the preview from "Send" before asking. Refresh `$QUOTE_TS` at preview success. From a successful preview, add all of these to the **same** summary:

- `data.requestId`
- `data.parsedTx` — `parsedTx.amount` is the smallest unit (wei for BNB). Divide by `decimals` from `data.tokenInfos` before showing a human amount.
- `data.simulationResult.balanceChanges`
- `data.simulationResult.allowanceChanges`
- `data.simulationResult.authorityChanges`
- `data.risks`
- `data.requireConfirmation`
- `data.expiresAt`

Those three change lists live under `simulationResult`, not at the top of `data`. If `simulationResult` is missing, the preview is unusable: stop. If the preview disagrees with your summary (another `to`, another amount, an unexpected allowance or authority change), stop and show both. A preview without `requestId`, or `351803`, stops the order.

Only a reply sent after that summary (and, for BAW, after that preview) counts. The reply must be just `yes`, `confirm`, `go ahead`, `确认`, or `同意`. When the host supports choice chips or buttons, present those exact confirm strings plus `取消` / `cancel` (preferred over typing). A chip that submits one of the confirm strings counts. A question, a condition, or a longer sentence that contains one of those words is not a confirmation. Anything earlier does not count.

When the reply arrives, read the latest block's `timestamp` before acting on it.

If that is more than 60 seconds after `$QUOTE_TS` from `references/quote.md`, or the BAW preview expired: the confirmation is void. Quote again, set the floor again, simulate again, preview again, and show **one** new summary + chips (do not ask "retry?"). Never send a call that differs from the one the user confirmed.

## Send, only after that confirmation

Every transaction is exactly this envelope:

| Field | Swap | Approve |
|---|---|---|
| `to` | Portal `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0` | the input token |
| `value` | wei integer (BNB buy) or `0` | `0` |
| `data` | swap calldata from "Calldata" | approve calldata from "Calldata" |
| `chainId` | `56` | `56` |

No gas limit and no gas-price fields.

### Path B, or path A with BAW

Do not hand the transaction to any other signer. The preview runs before the confirmation. Only `execute` runs after it.

```bash
# Before the confirmation:
baw contract-call preview --binanceChainId 56 --from $TRADER \
  --to <Portal for the swap, the input token for the approve> \
  --value <wei integer> --inputData <calldata from "Calldata"> --json

# After the confirmation:
baw contract-call execute --requestId <requestId from that preview> --json
```

No `--gasLimit` and no gas-price flags. A `requestId` from another chat, another preview, or an older preview is dead. See `references/errors.md`.

Do not change `to`, `value`, or the calldata after confirmation. Then go to `references/receipt.md`.

### Path A with a non-BAW signer

The user signs. The agent does not hold the key and does not run the signing step: a hardware-wallet or keystore confirm is interactive. Do not accept a private key, a mnemonic, or a password in chat or on a command line.

Give the user the envelope above with every value written out literally: `to` in full, `value` as a base-unit integer, `data` as the full hex, and chain ID `56`. They broadcast it with whatever tool drives their signer and paste the transaction hash back. The values must match the confirmed summary character for character. Do not give a keyed RPC URL that holds a secret; use the public fallback from `references/preflight.md`.

If the user asks for a ready-to-run command for their tool, print one that carries exactly those values and no gas-limit flag. For example, with Foundry `cast` and a hardware wallet or keystore:

```bash
# Swap (literal values, no shell variables):
cast send 0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0 <swap calldata hex> \
  --value <value> --from <trader> --chain 56 --rpc-url <rpc> <--ledger | --trezor | --account <name>>

# Approve:
cast send <input token> <approve calldata hex> \
  --from <trader> --chain 56 --rpc-url <rpc> <--ledger | --trezor | --account <name>>
```

For a Ledger or Trezor account that is not the default path, the user's derivation path is also needed; ask for it, do not guess.

Tell the user to send it now. If more than 60 seconds of chain time have passed since `$QUOTE_TS`, do **not** send: go back to Step 4 and quote again. A confirmation that is still unused after that window is void. If the user already broadcast a late transaction, do not send another; judge that hash with `references/receipt.md` (it may note the late window). The swap has no deadline argument; `minOutputAmount` is the only on-chain guard after broadcast.

Do not change `to`, `value`, or the calldata after confirmation. Then go to `references/receipt.md`.
