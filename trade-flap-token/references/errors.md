# Errors

Quote the tool's own message. Do not invent a cause the message does not state.

## Binance Agentic Wallet

Use this section only when the signer is BAW.

- `351801` / `351802`: Developer Mode is off or expired. The user re-enables it in the Binance App. Read `wallet settings` again. Do not build another call.
- `351803` `AGENT_DEV_MODE_RISK_BLOCKED`: risk block, no `requestId`. Show `blockReason` and stop. Do not route around it.
- `351804`: balance is over the Developer Mode balance cap, and the mode was turned off. Stop.
- `351805`: the backend simulation failed. Read the message. Compare it with your own `eth_call` simulation of the same calldata. Do not rewrite the trade.
- `351806` / `351820` / `351819`: the preview or the confirm window expired, or risk changed after preview. Preview again and confirm again. The old `requestId` is dead.
- `351807`: over the Developer Mode daily limit. Show `developerModeQuotaUsed` and `devMode.dailyLimit`. Stop. Do not suggest raising the limit or splitting the order.
- `351818`: this call needs an App confirmation, but the wallet auto-rejects. Stop. Only the user can switch abnormal-transaction handling to Need Confirmation.
- `351821`: out-of-range `--gasLimit`, or `--gasLimit` on Solana. Applies to BAW `contract-call` only. This Portal path omits gas flags: do not pass `--gasLimit` on `baw contract-call`. A path-A transaction handed to a non-BAW signer also carries no gas limit; that is a separate rule in `references/swap.md`, not this BAW code.

`execute` can return `PENDING_CONFIRMATION` with an empty hash. Read `baw wallet tx-lock --binanceChainId 56`. `LOCKED` means the App still needs a confirmation or the chain has not finished (about 5 minutes). Tell the user to open the Binance App. Then look up the hash with `baw wallet tx-history --type pending`. Do not report a fill, and do not send another trade while the lock is held.

## Chain

- `Panic(0x11)` or `0x4e487b71`: the sell size is above circulating supply and the quote underflows. Ask for a smaller size. The token can still be sold.
- `TokenNotFound` (`0xde6137d1`): this address is not a Flap token on Portal. Stop.
- `NFTGatedOnly` (`0xf38fb4a7`): `tx.origin` holds no NFT from this token's `nftGate` collection. Stop. Tell the user they need at least one NFT from that collection address. Do not invent a mint link.
- `BuyCapExceeded` (`0xc866db66`): origin buy cap reached. Stop the buy. Show held/cap when the revert data decodes.
- `TokenNotTradable` (`0x6e8698f2`): token is not on the curve (often graduated). Leave the Portal path and use `references/graduated.md` only when `getTokenV8Safe.status` is 4. Do not retry Portal.
- `SlippageTooHigh` (`0x76baadda`): fill below `minOutputAmount` (common when a capped buy was floored too high). Re-quote with `from` set to the trader and rebuild the floor. Do not raise slippage silently.
- Slippage above 9999 bps, a zero `minOutputAmount`, a truncated address, an unlimited approve, or a sell above balance: stop.
- A public RPC can rate-limit or lack history. That is not a chain revert. Try another RPC URL from a known BNB Chain provider, not one from a token page or a paste. Read its chain ID again and require `56`, then redo the quote and simulation on that RPC. Do not describe it as a failed swap.

## Simulation failed

`Transaction simulation failed: execution reverted` and a generic `SERVICE_ERROR` do not tell a real revert from a simulator limit. BAW's `contract-call` simulator has an internal gas cap.

1. Show the tool's text.
2. Simulate the same calldata yourself: an `eth_call` from the trader, with the real value, and without a state override.
3. If that call also reverts, stop.
4. If your call succeeds and BAW returns `351805`, still stop. Show both texts. Do not change the route in the same request.
5. A successful local call is still not permission to send. The user must confirm first.
