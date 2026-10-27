# Agent ID launch (optional)

An Agent ID launch lets only holders of the configured Agent ID collection buy or sell on the
inner market. The launch also enables a 2% per-origin buy cap automatically. This is optional; a
normal launch leaves `extensionID` / `extensionData` empty.

The launch still uses `newTokenV6` (Portal) or `newTokenV6WithVault` (VaultPortal) with
`tokenVersion = TOKEN_TAXED_V3`. Only the two extension fields change.

## 1. Preflight (BNB mainnet)

Use these values:

| Name | Value |
|---|---|
| Portal | `0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0` |
| RPC | `https://bsc-dataseed.binance.org` (or any trusted chain-56 RPC) |
| Agent ID collection | the ERC-721 collection address the user chose |

Read two Portal views with an `eth_call` (any EVM tool works — `viem`, `ethers`, `web3.py`,
Foundry `cast`, or raw JSON-RPC):

| Function | Selector | Returns |
|---|---|---|
| `newFeatureSwitch()` | `0xb1aad1e1` | `bool` |
| `nftWhitelisted(address agentId)` | `0x0bfdb80d` | `bool` |

```typescript
import { createPublicClient, http, parseAbi } from "viem";
import { bsc } from "viem/chains";

const PORTAL = "0xe2cE6ab80874Fa9Fa2aAE65D277Dd6B8e65C9De0";
const AGENT_ID_ABI = parseAbi([
  "function newFeatureSwitch() view returns (bool)",
  "function nftWhitelisted(address agentId) view returns (bool)",
  "function nftGate(address token) view returns (address)",
  "function maxBuyPerOrigin(address token) view returns (uint16 bps, uint256 maxBuyAmount)",
]);

const client = createPublicClient({ chain: bsc, transport: http(RPC_URL) });
const featureOn = await client.readContract({ address: PORTAL, abi: AGENT_ID_ABI, functionName: "newFeatureSwitch" });
const whitelisted = await client.readContract({ address: PORTAL, abi: AGENT_ID_ABI, functionName: "nftWhitelisted", args: [agentId] });
```

`nftWhitelisted` is the on-chain getter name; it returns whether an Agent ID collection is
whitelisted.

- `newFeatureSwitch() == false`: only tester addresses can start an Agent ID launch
  (`NFTGateLaunchDisabled` otherwise). Re-read it for every launch; do not cache an earlier
  value.
- `nftWhitelisted(agentId) == false`: only tester addresses can use that Agent ID collection
  (`NFTNotWhitelisted(agentId)` otherwise). Re-read it for every launch.
- Tester status is not exposed by a getter. If the user is not sure, ask Flap before building
  the transaction.

Both checks apply at creation time only. Removing an Agent ID collection from the whitelist
later does not change tokens that already launched.

## 2. Build the extension fields

```typescript
import { encodeAbiParameters, keccak256, toBytes } from "viem";

const EXTENSION_ARRAY_ID = keccak256(toBytes("flap.extensions.v1"));
const AGENT_ID_EXTENSION_ID = keccak256(toBytes("flap.extension.nft_gated.v1"));

const extensionID = EXTENSION_ARRAY_ID;
const extensionData = encodeAbiParameters(
  [
    {
      type: "tuple[]",
      components: [
        { name: "extensionId", type: "bytes32" },
        { name: "extensionData", type: "bytes" },
      ],
    },
  ],
  [
    [
      {
        extensionId: AGENT_ID_EXTENSION_ID,
        extensionData: encodeAbiParameters([{ type: "address" }], [agentId]),
      },
    ],
  ],
);
```

Use these values in the `newTokenV6` / `newTokenV6WithVault` params from
`references/construct-tx.md`. For a normal launch use:

```typescript
extensionID: "0x0000000000000000000000000000000000000000000000000000000000000000",
extensionData: "0x",
```

The on-chain extension id stays `flap.extension.nft_gated.v1`; it refers to the Agent ID
collection.

## 3. What the launch sets

After creation:

- `nftGate(token)` returns the configured Agent ID collection (or `address(0)` for a normal
  token).
- `maxBuyPerOrigin(token)` returns `(200, 20000000e18)`: 2% of the 1B max supply.
- `buyQuotaOf(token, origin)` uses `max(netBought, balanceOf(origin))` for Agent ID tokens.

Trading rules:

- Buys and sells on the inner market require `agentId.balanceOf(tx.origin) >= 1`.
  Otherwise the swap reverts with `NFTGatedOnly(tx.origin)`.
- The gate is checked on `tx.origin`, not `msg.sender`.
- The 2% cap applies to every buy, including the launch buy. At/over the cap the buy reverts
  with `BuyCapExceeded(origin, balance, cap)`; a buy from below the cap fills to the cap and
  refunds any excess input.
- DEX trades after graduation are not gated.
- `quoteExactInput` does not check the gate. A non-holder can get a quote, but the swap will
  revert. Check the gate before quoting or simulating.

Dev buy (the launch `quoteAmt`):

- The dev must hold the Agent ID collection too. Otherwise the launch buy reverts with
  `NFTGatedOnly(tx.origin)` and the whole transaction is reverted.
- The 2% cap applies to the dev buy with no exemption.
- A dev buy above the remaining allowance is **not** an error: it fills to the cap and the
  excess `quoteAmt` is refunded automatically in the same transaction. The launch still
  succeeds. Do not lower or skip the dev buy for this reason.
- The only cap revert is when the origin already holds at/over the cap before the dev buy:
  `BuyCapExceeded(origin, balance, cap)`.
- If the dev does not hold an Agent ID, set `quoteAmt = 0` and buy after acquiring one.

The gate reads the collection's ERC-721 `balanceOf(address)`. ERC-1155 and other standards are
not supported.

## 4. Examples on BNB mainnet

| Example          | Token                                        | Agent ID collection                          | `nftWhitelisted` at check | Note                                                                                                                                        |
|------------------|----------------------------------------------|----------------------------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| Agent ID case    | `0x3c6eA125F1dDA90215176075b777945aE9217777` | `0xB29Ff3126A9794df8AC02c6610e36942a14f308F` | No                        | Collection is open source and has a public `mint()`; new launches with it currently need a tester, or the collection must be re-whitelisted |
| Binance Agent ID | `0x4131Cae06908e7D874cBA44004d8803045367777` | `0x60B9262456dE011F7d15a54CdD8c48bAaa8B022F` | Yes                       | `0x60B9...022F` is the Binance Agent ID collection                                                                                          |

Both examples return `200` bps from `maxBuyPerOrigin`. The first token keeps working even
though its collection is not currently whitelisted, because the whitelist is only checked at
creation. Re-read `nftWhitelisted(agentId)` before launching; the values above are a snapshot,
not constants.

## 5. Admin operations

Admin operations are out of scope for this skill. Agent ID collections and existing gates are
managed through Portal roles.

## 6. Verify after launch

Read two Portal views for the new token (`AGENT_ID_ABI` from section 1):

| Function | Selector | Expected |
|---|---|---|
| `nftGate(address token)` | `0x9b95f8dc` | the Agent ID collection used in the extension |
| `maxBuyPerOrigin(address token)` | `0x4b02c8e9` | `(200, 20000000e18)` |

```typescript
const gate = await client.readContract({ address: PORTAL, abi: AGENT_ID_ABI, functionName: "nftGate", args: [token] });
const [bps, maxBuyAmount] = await client.readContract({ address: PORTAL, abi: AGENT_ID_ABI, functionName: "maxBuyPerOrigin", args: [token] });
```

`nftGate(token)` must equal the Agent ID collection used in the extension. If it is zero, the
extension was not decoded and the launch is not gated.
