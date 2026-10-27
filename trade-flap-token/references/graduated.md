# Graduated token (path C)

Use this file when `getTokenV8Safe` `status` is 4 (`DEX`). The token has left the inner market.

## What changes

- Portal `swapExactInput` / `quoteExactInput` revert with `TokenNotTradable`. Do not build a Portal call.
- Any Agent ID / NFT gate from the curve **no longer applies**. Do not read `nftGate` for eligibility. Do not ask the user to hold an NFT for this trade.
- Do not build Pancake, Universal Router, or other DEX calldata in this skill.

## Signer and route

Binance Agentic Wallet only. The default buy/sell path is the wallet's own market swap (`baw market-order`), which uses Binance Web3 Wallet routing. This skill does not pick pools or routers.

If BAW is not installed or not `CONNECTED`, offer to install / sign in per `references/preflight.md` §7. Do not offer another signer; if the user refuses BAW, stop.

Developer Mode is **not** required for `market-order` (it is required for Portal `contract-call` on paths A/B). Still run `baw wallet status` / `address` so `$TRADER` is the chain-`"56"` entry.

## Flow

Follow the installed `binance-agentic-wallet` skill's `references/market-order.md` and its security pre-check. In short:

1. Resolve `$TRADER` on chain 56.
2. On a buy, try the BAW market-order security pre-check from the `binance-agentic-wallet` skill (`references/market-order.md` / `security.md`), typically via `query-token-audit`. Follow `references/safety.md`: silent one-shot install/load if missing; if still no usable audit, put `audit: unavailable` in the trade summary and continue — **no** separate `继续` gate. A sell does not need the audit.
3. `baw market-order quote` with `--binanceChainId 56`, human-readable amounts, native BNB as `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` when paying or receiving BNB. Do this before opening the confirmation message.
4. Show **one** summary: full token address, side, spend, expected output, slippage (default from that skill unless the user set one), audit line, confirm-within-~50s (or quote freshness from that skill), and that the route is **BAW market-order (Binance wallet routing)**, not Flap Portal. Say NFT gate does not apply after graduation.
5. Wait for a standalone `yes` / `confirm` / `go ahead` / `确认` / `同意` after that summary. Prefer chips: those labels plus `取消` / `cancel`. An earlier yes does not count.
6. Only then `baw market-order swap` with the same parameters.
7. Poll `baw market-order list --orderId …` until `FINISHED` or `FAILED`. An `orderId` alone is not a fill. Report `txHash` and amounts only on `FINISHED`.

## Failures

If quote or swap fails because the token is outside Binance routing, liquidity, or allowlist: show the tool message and stop. Do not fall back to Portal. Do not hand-build a DEX swap.

If swap fails for **slippage** (e.g. `ORDER_API_ERROR` / insufficient slip): show the error, **do not** raise slippage yourself. Offer chips for a user-chosen new slip (e.g. `8%`, `10%`, `取消`). After they pick a value with a unit, re-quote and show a **new** confirmation summary.

## Out of scope here

Portal approve / revoke, Portal receipt event matching, inner-market tax lines as trading requirements, and Agent ID minting.
