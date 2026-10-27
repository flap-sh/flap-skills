# Agent ID gate (path B only)

Use this file only when `references/token-state.md` chose path B: inner-market token with a non-zero `nftGate(token)`.

This gate is **per token**. Each gated token points at its own ERC-721 collection. There is no single site-wide NFT. After graduation (`status == 4`), the gate does not apply — use `references/graduated.md` instead.

## Signer

Binance Agentic Wallet only. `$TRADER` is the BAW address for chain `"56"`. The on-chain check uses `tx.origin`; BAW `contract-call` is sent from that address. Do not use a hardware wallet, a keystore, or another EOA for path B.

If BAW is not installed or not `CONNECTED`, offer to install / sign in per `references/preflight.md` §7, with this context:

> This token is Agent ID-gated on the inner market and can only be traded with Binance Agentic Wallet. Install / sign in now?

If the user refuses or insists on another wallet, stop: the gate is checked on the BAW origin. Do not list another signer as an alternative. Do not offer a multi-signer picker on this path.

## Eligibility

`$NFT_GATE` is the address from `nftGate(token)`.

Call these on the `$NFT_GATE` collection (ERC-721 / ERC-20 shared selectors, see `references/interface.md`):

| Call | Selector | Returns |
|---|---|---|
| `balanceOf($TRADER)` | `0x70a08231` | `uint256` — number of NFTs held from this collection |
| `name()` | `0x06fdde03` | `string` |
| `symbol()` | `0x95d89b41` | `string` |

`name` / `symbol` may revert; then show the collection address alone.

- Collection-wide: any NFT from `$NFT_GATE` counts. Do not require a specific `tokenId`.
- Buy **and** sell on the curve require `balanceOf($TRADER) >= 1`.
- If balance is 0: stop. Say the trader must hold at least one NFT from **this token's** collection (`$NFT_GATE`, plus name/symbol when known) before an inner-market trade. Do **not** name a fixed test collection, Binance Agent ID, or a mint command unless the user asks how to get one for this exact address.
- Do not mint, buy, or transfer an NFT for the user.

`quoteExactInput` does **not** check the NFT. A non-holder can still get a quote. Never treat a successful quote as eligibility. Run this check before Step 4.

## Buy quota on gated tokens

Agent ID-gated launches usually ship with a per-origin buy cap (often 200 bps = 2% of max supply). Always read `maxBuyPerOrigin` and `buyQuotaOf` from `references/tax-and-quota.md`. Never hardcode 2%.

- Buy: `remaining == 0` (when a real cap exists) stops the buy.
- Sell: not limited by that buy cap. Only the NFT holder check applies.
- On gated tokens, `buyQuotaOf.bought` is `max(netBought, balanceOf(origin))`. Prefer the Portal views over hand-computed balances.

## Confirmation lines

On every path-B summary, include:

- `route: Agent ID-gated inner market`
- `nftGate: <full address>` and name/symbol when known
- `nft balance: <N> (need ≥ 1)`
- buy quota lines when buying

## Errors

- `NFTGatedOnly` (`0xf38fb4a7`): wallet holds no NFT. Stop. Same message as a failed eligibility check.
- `BuyCapExceeded` (`0xc866db66`): buy quota used up. Stop the buy. See `references/errors.md`.
