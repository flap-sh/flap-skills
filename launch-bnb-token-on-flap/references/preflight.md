# Preflight Checks

Verify all of the following before proceeding. Block and inform the user if any check fails.

## 1. EVM tooling

This skill describes the contracts, calldata, and transaction fields; it does not require a
specific tool. You need a way to:

- make `eth_call` reads and read balances on a chain-56 RPC,
- ABI-encode calldata (`newTokenV6` / `newTokenV6WithVault`, `encodeAbiParameters` for
  `vaultData` / `extensionData`),
- compute a CREATE2 address and `keccak256` (salt mining).

Any EVM library or CLI works — `viem`, `ethers`, `web3.py`, Foundry `cast`, or raw JSON-RPC.
The examples in these reference files use `viem` for illustration. If no tool is available,
install one at a pinned version in a temporary directory outside this skill, for example:

```bash
npm install viem
```

Do not add a `scripts/` directory or a package to this skill.

## 2. EVM wallet

An EVM wallet capable of signing and sending transactions on BNB Chain (chain ID `56`) must be available. The method of signing (private key, hardware wallet, browser wallet, MPC, etc.) is determined by the caller — this skill mainly helps you construct the transaction data.

## 3. BNB mainnet RPC access

Ask for an RPC URL for BNB Chain mainnet (chain ID `56`). A public fallback is `https://bsc-dataseed.binance.org/`. 

Verify connectivity — read the chain ID (must be `56`) and the latest block. With `viem`:

```typescript
import { createPublicClient, http } from "viem";
import { bsc } from "viem/chains";

const client = createPublicClient({ chain: bsc, transport: http(RPC_URL) });
const block = await client.getBlockNumber();
console.log("Latest block:", block);
```

## 4. Sufficient BNB balance

The wallet needs BNB for:
- Gas fees (launching costs ~0.002–0.010 BNB in gas).
- Any initial buy amount (`quoteAmt`) the user wants to spend.

Check balance:

```typescript
const balance = await client.getBalance({ address: walletAddress });
// balance is in wei; 1 BNB = 1e18 wei
```

Warn if balance is below 0.01 BNB (exclusive of launch buy amount).
