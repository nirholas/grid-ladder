---
name: use-grid-ladder
description: Build, test, integrate and deploy the GridLadderHook Uniswap v4 hook from the grid-ladder repo. Use when you need to work with this hook's contract, wire it into a pool, or deploy it with a mined CREATE2 salt.
---

# GridLadderHook

A pool that quotes a bid and an ask, like a market maker, instead of one price both ways.

Family: Curves. Tags: curve, custom-curve, market-making, spread, inventory. License: Apache-2.0. Source: `src/hooks/GridLadderHook.sol`. The machine-readable manifest is `hook.json` (ABI, errors, permissions, properties).

## Build and test

Foundry project (`foundry.toml`, solc 0.8.26, evm `cancun`). From a clone of the repo:

```bash
git clone https://github.com/nirholas/grid-ladder && cd grid-ladder
git submodule update --init --recursive   # dependencies live in lib/
forge build
forge test
```

Tests are in `test/GridLadderHook.t.sol`. Use `FOUNDRY_PROFILE=ci forge test` for the larger fuzz run (5000 runs).

## Hook permissions

Uniswap v4 reads a hook's permissions from the low bits of its address. This hook claims: `beforeSwapReturnsDelta`, `beforeSwap`, `beforeRemoveLiquidity`, `beforeAddLiquidity`, `beforeInitialize`. It is not upgradeable and has no custom swap data requirement (`hook.json` `properties`).

## Configuring

This hook has no `configure` call. Its terms are constructor arguments, fixed at deployment:

```solidity
constructor(IPoolManager, Ladder bid, Ladder ask, string shareName, string shareSymbol)
```

Edit the arguments in `script/Deploy.s.sol` before deploying, since the deployed address depends on them.

## Deploying

A v4 hook must be deployed at an address whose bits match its permissions, so `script/Deploy.s.sol` mines a CREATE2 salt first. The dry run sends nothing:

```bash
export RPC_URL=https://...        # a chain with a Uniswap v4 PoolManager
forge script script/Deploy.s.sol --rpc-url $RPC_URL
```

Broadcasting spends gas from the signer (`PRIVATE_KEY`, or `--account` / `--ledger`) and is irreversible, so show the user the chain, the predicted hook address and the signer, and get an explicit yes before running:

```bash
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Re-running is safe: a hook already at its mined address is reported and skipped. If the miner fails with `EvmError: OutOfGas`, add `--gas-limit 30000000000`. `--verify` needs `ETHERSCAN_API_KEY`. The chain must be in the address book `src/libraries/Chains.sol`. `script/Deploy.s.sol` fails with "no Uniswap v4 PoolManager known for this chain" otherwise.

## Identifying a deployed hook

```solidity
IHookMetadata(hookAddress).hookName();
IHookMetadata(hookAddress).specURI();   // points at hook.json
```

## Errors you may meet

`AlreadyInitialized()`, `AmountTooSmall()`, `CrossedBook()`, `ExpiredPastDeadline()`, `InsufficientInitialLiquidity()`, `InsufficientReserves()`, `InvalidLadder()`, `InvalidNativePayer(address)`, `InvalidNativeValue()`, `InvalidPool()`, `LiquidityOnlyViaHook()`, `NoLiquidity()`, `PoolNotInitialized()`, `SafeERC20FailedOperation(address)`, `SwapTooLarge()`, `TooMuchSlippage()`.

## Limitation

Two ladders mean the pool is not a conservative curve: there is no single invariant a swap preserves, so the usual arbitrage-free reasoning about constant-function market makers does not apply, and a badly configured spread can be crossed for a loss. The bands are also fixed at deployment, so a pool whose asset leaves the configured range stops quoting on that side. It is a market maker's tool and it wants a market maker's attention.

More: `docs/integrating.md`, `docs/deploying.md`, and None.
