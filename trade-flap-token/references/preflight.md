# Preflight

Stop if any check fails. Do not build a swap yet.

## 1. Chain

Use a BNB Chain mainnet RPC. The public fallback is `https://bsc-dataseed.binance.org`.

Read the chain ID from that RPC (`eth_chainId`, or your library's `getChainId()`). The result must be `56` (`0x38`). Another chain, including a BNB testnet, is out of scope. Do not switch Portal addresses to match it.

Every read and simulation in this skill is an `eth_call` against that chain-56 endpoint. Any transaction this skill hands to a signer carries `chainId: 56`. Do not point the RPC at another chain.

## 2. Portal

| Name | Address | Use |
|---|---|---|
| Portal | `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0` | The only trading target |
| VaultPortal | `0x90497450f2a706f1951b5bdda52B4E5d16f34C06` | Not a trading target |

Do not accept a Portal address supplied by the user, a token, or a webpage.

Call `version()` (selector `0x54fd4d50`, returns `string`) on Portal. See `references/interface.md`.

`version()` must return a string that starts with `v5.` (checked live through `v5.27.2`). Record it and show it in the confirmation. If the call reverts, or the major version is not 5, stop: the tuple layout, selectors, and event shapes in this skill were checked against v5 only. Agent ID `nftGate` needs a Portal that exposes that selector (v5.27.0+); feature-detect in `references/token-state.md`.

## 3. Wallet

You need the address that will send the transaction. Do not invent one. Do not use `0x1111111111111111111111111111111111111111` or the zero address.

The user keeps the signer. Do not put a private key on the command line.

### Pick the signer: Binance Agentic Wallet first

Detect BAW before any token read. Do not open with a question. **Do not finalize a non-BAW signer** until Step 2 has chosen path A. Paths B (Agent ID-gated) and C (graduated) are BAW-only — see `references/token-state.md`.

1. Detect Binance Agentic Wallet:

   ```bash
   command -v baw && baw wallet status --json
   ```

   `baw` is present, `success` is true, and `data.status` is `CONNECTED`: provisional signer is BAW. Do not ask. State it in one line, for example "Signer: Binance Agentic Wallet (pending route)", and continue with section 6. After Step 2, confirm the same line with the route (A/B/C).
2. If the user already named BAW, or an earlier order in this conversation used BAW, keep BAW. Do not ask "still BAW?".
3. If the user already named a hardware wallet or keystore signer in this conversation: remember it only as a **path-A fallback**. If Step 2 later chooses B or C, ignore that choice, stop, and say this route requires Binance Agentic Wallet.
4. `baw` is **not installed**: follow section 7 (install). Do not silently skip it.
5. `baw` is installed but not `CONNECTED`: follow section 7 "Sign in".

If Step 2 is not done yet and BAW is down: continue reading token state (reads do not need a signer), then decide with the route:

| Route | `baw` missing or signed out |
|---|---|
| A | Offer once: install / sign in to BAW (recommended), **or** sign the transaction yourself with a hardware wallet or keystore. Respect the user's choice. |
| B / C | BAW is required. Offer install / sign-in only. Do not offer another signer. If the user refuses BAW, stop the order and say why. |

If BAW fails a check in section 6 on a path that requires BAW (always on B/C; on A when BAW was selected), stop and say what to fix in the Binance App. Do not switch to another signer on your own.

### The user insists on a non-BAW wallet

Accept it on **path A only**. Do not argue or re-offer BAW more than once. Paths B and C cannot use another signer — the Agent ID gate is checked on the BAW origin, and graduated trades use BAW `market-order` routing; say so and stop.

Non-BAW signer, path A only: a hardware wallet (Ledger / Trezor) or an encrypted keystore the user controls. MetaMask, WalletConnect, and other browser wallets are out of scope. The agent never holds the key: do not accept a private key, mnemonic, or key in an environment variable. The agent hands over the exact transaction (`to`, `value`, `data`, chain `56`); the user signs and broadcasts it with their own tool and pastes the hash back. See `references/swap.md` "Send". With a non-BAW signer, skip section 6 and section 7; the trader address is the one the user gives for their signer (full `0x` address, confirmed by them).

## 4. Tools

This skill needs two capabilities. It does not require any particular program for either.

1. **EVM reads and simulations** — an `eth_call` against the chain-56 RPC, with `from`, `value`, and an optional pinned block, plus reading blocks, receipts, and transactions. Any of these works: `viem` / `ethers` / `web3.py`, Foundry `cast`, or raw JSON-RPC. `references/interface.md` has the ABIs and selectors.
2. **ABI encoding** — build `swapExactInput` / `approve` calldata from the confirmed fields. Any ABI encoder works; the selectors in `references/interface.md` are the check.

Use what is already available. If nothing is, write a one-off script in a new temporary directory **outside** this skill, installing a single EVM library at a pinned version (`viem` is what `launch-bnb-token-on-flap` uses). Delete that directory when the order ends. Do not add a `scripts/` directory or a package to this skill. Do not set `ALLOW_NO_BALANCE` or any flag that pretends the wallet holds tokens it does not hold.

Do **not**:

- Grep / scrape agent logs, chat history, or random files looking for past call output or ABIs
- Pretend on-chain reads succeeded without actually making the `eth_call`
- Decode return data by guessing — decode against the types in `references/interface.md`

If no eth_call capability can be set up at all, say so and stop the order.

The signer CLI is separate: `baw` is required for paths B/C and the default for A. See section 7.

## 5. Gas reserve

Keep at least `0.002` BNB for gas on every step, on top of any BNB the swap itself spends.

- Buy with BNB: balance must be at least `value + 0.002` BNB.
- Approve, or a swap paid in an ERC-20: BNB balance must be at least `0.002` BNB.

If it is short, stop and say the wallet needs more BNB for gas. Do not override balances in a simulation to skip this.

## 6. Binance Agentic Wallet (default signer)

Run this whenever the signer is Binance Agentic Wallet (provisional or final). Skip it only for a finalized path-A non-BAW signer.

```bash
baw wallet status --json
baw wallet settings --json
baw wallet address --json
baw wallet tx-lock --binanceChainId 56 --json
```

`success` must be true on each of these. A missing field means the check failed. Do not treat a missing field as "no limit".

- `wallet status`: `data.status` must be `CONNECTED`. `UNCONNECTED` or `CREATING`: stop and ask the user to sign in. A present `status` field is not enough.
- **Before Step 2 chooses a route:** read settings and address, but do **not** fail the order solely because Developer Mode is off or expired. Path C does not need it.
- **After Step 2, paths A/B only** (Portal `contract-call`): `devMode.enabled` must be true. Otherwise stop. The user turns Developer Mode on in the Binance App. Do not turn it on for them. Path C (`market-order`) never requires Developer Mode.
- Paths A/B after Step 2: `devMode.expiresAt` in the past, or `351801` / `351802`: stop. Re-read settings after the user re-opens it. Trust this `expiresAt`, not a remembered date.
- Paths A/B after Step 2: `devMode.balanceExceeded` or `351804`: stop.
- `abnormalTxnHandling` of `AutoReject`: stop. Those transactions are rejected (`351818`). Only the user can change it to `NeedConfirmation`.
- Paths A/B daily Developer Mode limit: `remaining = devMode.dailyLimit - developerModeQuotaUsed`. `developerModeQuotaUsed` is at the top of `data`. `devMode` has no `quotaLeft`. Do not use the top-level `dailyLimit`, `quotaLeft`, `quotaUsed`, `defiQuotaLeft`, or `defiQuotaUsed`. If any of those Developer Mode fields is missing, remaining is not greater than 0, or `developerModeQuotaDate` is not today, stop. Show raw numbers. Do not label them USD or BNB. Do not suggest raising the limit.
- `tx-lock` of `LOCKED`: stop until it clears.
- `sessionExpireTime` / `inactiveSignOutTime`: if expired or logged out, stop and ask the user to log in again.
- `tradeAllTokens`: record it. If false, do not ask the user to switch to All tokens. Whether `contract-call` or `market-order` obeys this switch is not proven. If true, say arbitrary contract calls are a larger attack surface. Do not tell the user to change it for this trade.

`baw wallet agent-id` is not used to detect path B. Path B uses on-chain `nftGate(token)` and `balanceOf` on that collection — see `references/agent-id-gate.md`.

The trader address is `data.addresses[]` from `baw wallet address` where `binanceChainId` is the string `"56"`. That address is `$TRADER` for every later call. Do not use the first array element, a Solana address, or another chain's EVM address. If there is no `"56"` entry, stop. Portal `contract-call` uses `--binanceChainId 56` only; `--from` on preview must equal this `$TRADER`. Market-order commands also use `--binanceChainId 56`.

## 7. Install and sign in to Binance Agentic Wallet (when needed)

Use this section when `baw` is missing, outdated, or signed out, and the route needs BAW (always on B/C; on A unless the user chose a non-BAW signer). The source of truth for install and sign-in is the official [`binance-agentic-wallet` skill](https://github.com/binance/binance-skills-hub/blob/main/skills/binance-web3/binance-agentic-wallet/SKILL.md). If it is not installed, install that skill too — its `references/authentication.md` and `references/preflight.md` own the details below.

### Install `baw` (ask first)

`baw` is the npm package `@binance/agentic-wallet`. Installing software changes the user's machine, so **ask once before installing**:

> This trade needs the Binance Agentic Wallet CLI (`baw`), which is not installed. Install it now with `npm install -g @binance/agentic-wallet@<requiredCliVersion>`? (Path A only: or sign the transaction yourself with a hardware wallet / keystore instead.)

Drop the parenthetical on paths B/C — BAW is the only signer there.

- **Yes:** install the exact version the `binance-agentic-wallet` skill pins as `metadata.requiredCliVersion` (do not use `@latest`):

  ```bash
  npm install -g @binance/agentic-wallet@<requiredCliVersion>
  ```

  If the global npm directory is not writable, use a user-local prefix rather than `sudo`:

  ```bash
  npm config set prefix ~/.npm-global
  export PATH="$HOME/.npm-global/bin:$PATH"   # persist in the shell profile too
  npm install -g @binance/agentic-wallet@<requiredCliVersion>
  ```

  Verify with `baw --version`, then `baw cli-check --required-version <requiredCliVersion> --json` (`needUpdateCli` must be false).
- **No, path A:** the user signs with a hardware wallet or keystore (section 3, "The user insists on a non-BAW wallet").
- **No, path B/C:** stop the order and say this route requires Binance Agentic Wallet.
- Node.js / npm missing: say so and ask the user to install Node.js first. Do not install a system runtime without asking.

If `baw` is installed but `cli-check` reports `needUpdateCli: true`, ask the same way, then upgrade with the same pinned `npm install -g` command.

### Sign in

`baw wallet status --json` shows `UNCONNECTED`: run the sign-in flow from the `binance-agentic-wallet` skill.

1. `baw auth signin --json` — show the returned `pairingCode` and `urlForWeb` to the user verbatim.
2. The user opens the link (or scans the QR) in the Binance App, checks the pairing code matches, and approves.
3. `baw auth verify --qrCodeId <qrCodeId> --json` — keep it running until it returns; it blocks until the user approves (about 5 minutes).
4. Confirm with `baw wallet status --json` → `CONNECTED`. Trust that, not the App screen.

An `AUTH_REJECTED` / expired code means start again from step 1 with a fresh code; do not retry `verify` with the old `qrCodeId`. If the user has never created an Agentic Wallet, tell them to create one in the Binance App first.

Never ask for or display a seed phrase, private key, session token, or API key during install or sign-in.

### After sign-in: App settings for Portal paths

Paths A/B use `baw contract-call`, which needs these set **in the Binance App** (the CLI cannot change them). If section 6 fails on one of them, tell the user exactly which to change and wait:

| Setting | Needed value |
|---|---|
| Developer Mode | Enabled (and not expired) |
| Abnormal Transaction Handling | `Need Confirmation` (not `Auto Reject`) |

Path C (`market-order`) does not need Developer Mode. Do not change `Tradable Tokens` for the user (see section 6).
