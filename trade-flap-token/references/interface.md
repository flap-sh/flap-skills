# Contract interface

Everything this skill reads or sends is a plain EVM call on BNB Chain mainnet (chain ID `56`). This file lists the contracts, function signatures, selectors, return layouts, errors, and events. The other reference files say **what** to call and **when**; this file says **how the call is shaped**.

Use whatever EVM tooling the agent already has: `viem`, `ethers`, `web3.py`, Foundry `cast`, or raw JSON-RPC (`eth_call`, `eth_getBlockByNumber`, `eth_getTransactionReceipt`). No specific tool is required. The examples below use `viem`, the same library as the `launch-bnb-token-on-flap` skill. They are illustrations of the call shape, not scripts shipped with this skill.

## Contracts (BNB mainnet)

| Name | Address | Use |
|---|---|---|
| Portal | `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0` | The only trading target |
| VaultPortal | `0x90497450f2a706f1951b5bdda52B4E5d16f34C06` | Not a trading target |

Do not accept a Portal address supplied by the user, a token, or a webpage. The layouts below were checked against Portal `v5.x` (live through `v5.27.2`).

## Portal functions

| Function | Selector | Mutability | Returns |
|---|---|---|---|
| `version()` | `0x54fd4d50` | view | `string` |
| `getTokenV8Safe(address token)` | `0x62fafcca` | view | `TokenStateV8Safe` tuple (below) |
| `nftGate(address token)` | `0x9b95f8dc` | view | `address` (v5.27.0+; older builds revert) |
| `maxBuyPerOrigin(address token)` | `0x4b02c8e9` | view | `(uint16 bps, uint256 maxBuyAmount)` |
| `buyQuotaOf(address token, address origin)` | `0x929b098c` | view | `(uint256 bought, uint256 remaining)` |
| `quoteExactInput((address inputToken, address outputToken, uint256 inputAmount))` | `0xfc847c2b` | **nonpayable** — read with `eth_call` only, never send | `uint256 outputAmount` |
| `swapExactInput((address inputToken, address outputToken, uint256 inputAmount, uint256 minOutputAmount, bytes permitData))` | `0xef7ec2e7` | payable | `uint256 outputAmount` |

Do not use `getTokenV8`, `getTokenV9Safe`, or `swapExactInputV3`.

### `getTokenV8Safe` return tuple

| Index | Field | Type |
|---|---|---|
| 0 | `status` | `uint8` (enum, see `token-state.md`) |
| 1 | `reserve` | `uint256` |
| 2 | `circulatingSupply` | `uint256` |
| 3 | `price` | `uint256` |
| 4 | `tokenVersion` | `uint8` |
| 5 | `r` | `uint256` |
| 6 | `h` | `uint256` |
| 7 | `k` | `uint256` |
| 8 | `dexSupplyThresh` | `uint256` |
| 9 | `quoteTokenAddress` | `address` (zero = native BNB) |
| 10 | `nativeToQuoteSwapEnabled` | `bool` |
| 11 | `extensionID` | `bytes32` |
| 12 | `buyTaxRate` | `uint256` (bps) |
| 13 | `sellTaxRate` | `uint256` (bps) |
| 14 | `pool` | `address` |
| 15 | `progress` | `uint256` |
| 16 | `lpFeeProfile` | `uint8` |
| 17 | `dexId` | `uint8` |

## ERC-20 / ERC-721 functions

| Function | Selector | Called on |
|---|---|---|
| `decimals()` | `0x313ce567` | quote token, Flap token |
| `symbol()` | `0x95d89b41` | quote token, Flap token, NFT collection |
| `name()` | `0x06fdde03` | NFT collection |
| `balanceOf(address)` | `0x70a08231` | ERC-20 balance, or ERC-721 holding count on `nftGate` |
| `allowance(address owner, address spender)` | `0xdd62ed3e` | input token, spender = Portal |
| `approve(address spender, uint256 amount)` | `0x095ea7b3` | input token, spender = Portal |

## Errors

| Error | Selector | Meaning |
|---|---|---|
| `TokenNotFound(address)` | `0xde6137d1` | Not a Flap token on this Portal |
| `TokenNotTradable(address)` | `0x6e8698f2` | Not on the curve (often graduated) |
| `SlippageTooHigh(uint256,uint256)` | `0x76baadda` | Fill below `minOutputAmount` |
| `NFTGatedOnly(address)` | `0xf38fb4a7` | `tx.origin` holds no NFT from the token's gate |
| `BuyCapExceeded(address,uint256,uint256)` | `0xc866db66` | Origin buy cap reached |
| `Panic(uint256)` with code `0x11` | `0x4e487b71` | Arithmetic underflow (sell above circulating supply) |

Match a revert by its first 4 bytes. Do not match on the decoded message alone.

## Events (emitted by Portal)

Neither event has indexed fields: a matching log has exactly one topic (`topic0`), and every field is ABI-encoded in `data`.

```text
TokenBought(uint256 ts, address token, address buyer, uint256 amount, uint256 eth, uint256 fee, uint256 postPrice)
  topic0 0xa800a2038683844fac66747f771bfdfae862eb28b16bcfa387afa9fbacce8ff7
TokenSold(uint256 ts, address token, address seller, uint256 amount, uint256 eth, uint256 fee, uint256 postPrice)
  topic0 0x03a4693e592f5e75dc7c136acb39b146d2b4966c0e509c34f362dee02b3b861a
```

## ABI (human-readable)

```typescript
import { parseAbi } from "viem";

export const PORTAL = "0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0";

export const PORTAL_ABI = parseAbi([
  "function version() view returns (string)",
  "function getTokenV8Safe(address token) view returns ((uint8 status, uint256 reserve, uint256 circulatingSupply, uint256 price, uint8 tokenVersion, uint256 r, uint256 h, uint256 k, uint256 dexSupplyThresh, address quoteTokenAddress, bool nativeToQuoteSwapEnabled, bytes32 extensionID, uint256 buyTaxRate, uint256 sellTaxRate, address pool, uint256 progress, uint8 lpFeeProfile, uint8 dexId))",
  "function nftGate(address token) view returns (address)",
  "function maxBuyPerOrigin(address token) view returns (uint16 bps, uint256 maxBuyAmount)",
  "function buyQuotaOf(address token, address origin) view returns (uint256 bought, uint256 remaining)",
  "function quoteExactInput((address inputToken, address outputToken, uint256 inputAmount) params) returns (uint256)",
  "function swapExactInput((address inputToken, address outputToken, uint256 inputAmount, uint256 minOutputAmount, bytes permitData) params) payable returns (uint256)",
  "event TokenBought(uint256 ts, address token, address buyer, uint256 amount, uint256 eth, uint256 fee, uint256 postPrice)",
  "event TokenSold(uint256 ts, address token, address seller, uint256 amount, uint256 eth, uint256 fee, uint256 postPrice)",
  "error TokenNotFound(address token)",
  "error TokenNotTradable(address token)",
  "error SlippageTooHigh(uint256 actualAmount, uint256 minAmount)",
  "error NFTGatedOnly(address)",
  "error BuyCapExceeded(address, uint256, uint256)",
]);

export const ERC20_ABI = parseAbi([
  "function decimals() view returns (uint8)",
  "function symbol() view returns (string)",
  "function name() view returns (string)",
  "function balanceOf(address) view returns (uint256)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "function approve(address spender, uint256 amount) returns (bool)",
]);
```

## One way to make a call

Every read, quote, and simulation in this skill is the same operation: an `eth_call` with `to`, `data`, an optional `from` and `value`, and an optional pinned block. Encode the call, send `eth_call`, decode the result. A generic example:

```typescript
import { createPublicClient, http, encodeFunctionData, decodeFunctionResult } from "viem";
import { bsc } from "viem/chains";

const client = createPublicClient({ chain: bsc, transport: http(RPC) });

async function ethCall({ to, abi, functionName, args = [], from, value = 0n, blockNumber }) {
  const data = encodeFunctionData({ abi, functionName, args });
  const { data: ret } = await client.call({ to, data, account: from, value, blockNumber });
  return decodeFunctionResult({ abi, functionName, data: ret });
}
```

`quoteExactInput` and `swapExactInput` are not `view`, so use a raw `eth_call` like this one (or `simulateContract`), not a helper that only accepts view functions. A revert surfaces as an error whose data starts with one of the selectors above.

The same shape as raw JSON-RPC:

```json
{"jsonrpc":"2.0","id":1,"method":"eth_call","params":[{"from":"<trader>","to":"<contract>","value":"0x0","data":"<encoded calldata>"},"<block tag or hex number>"]}
```

## Numbers

Every return value is an integer in the contract's base units. Keep them as integers (`bigint` in JavaScript, `int` in Python) end to end. Never convert to floats, and never use shell `$(( ))`, which overflows. If your tool prints a display form next to the integer (for example `cast` prints `1710194868288753060294554 [1.71e24]`), use only the integer.

## Transaction envelope

A transaction this skill asks a signer to send is always exactly:

| Field | Value |
|---|---|
| `to` | Portal (swap) or the input token (approve) |
| `value` | wei integer — the BNB amount on a buy with BNB, otherwise `0` |
| `data` | the calldata built in `swap.md` "Calldata" |
| `chainId` | `56` |

No gas limit and no gas price. The signer estimates gas.
