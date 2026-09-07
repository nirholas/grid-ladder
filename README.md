# GridLadder

**A pool that quotes a bid and an ask, like a market maker, instead of one price both ways.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://grid-ladder.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/GridLadderHook.sol`](src/hooks/GridLadderHook.sol)
- **Licence:** Apache-2.0

## How it works

Every AMM curve has a single price at any moment, and both sides of the market trade against it. That is the defining simplification of the design, and it means a pool cannot express the one thing a quoting desk exists to express: I will buy at this, and sell at that, and the gap between them is what I am paid. The gap is not the same thing as a fee.

A fee is symmetric and proportional; a spread is a position. A desk that is long and wants to get flat quotes a keen bid and a wide ask, and it does that by moving the two sides independently. An AMM with a fee cannot say that at all: raising the fee makes it less willing to trade in both directions equally, which is exactly not what the desk wanted.

This pool holds two ladders. The bid ladder is what it pays for currency0, the ask ladder is what it charges, and each has its own base price, its own increment and its own band width. Set them symmetrically and it behaves like an ordinary stepped pool with a spread.

Set them apart and the pool leans: keen on one side, wide on the other, which is a resting position rather than a fee schedule. Both ladders step with inventory, so the pool also becomes less willing to keep going the way it is already leaning, which is the same self-correction any inventory-aware desk applies. The spread is the providers' revenue and it never leaves the reserves, so there is no separate fee parameter: a swap that crosses the spread simply hands the pool more than the mid, and the shares are a claim on reserves that grew.

Setting `askBaseX96` equal to `bidBaseX96` makes the pool free to trade and is allowed, because refusing it would be an opinion rather than a safety property.

## Prior art

Bancor's Carbon quotes independent, asymmetric bid and ask curves and is the direct ancestor of this idea, off v4 and as per-user strategies rather than a pool. Uniswap v4's own range orders express one side at a time. A single fungible v4 pool that quotes two independent inventory-stepped ladders, so providers share one book with a real spread, is the contribution here.

## Where it does not help

Two ladders mean the pool is not a conservative curve: there is no single invariant a swap preserves, so the usual arbitrage-free reasoning about constant-function market makers does not apply, and a badly configured spread can be crossed for a loss. The bands are also fixed at deployment, so a pool whose asset leaves the configured range stops quoting on that side. It is a market maker's tool and it wants a market maker's attention.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AlreadyInitialized()` | Hook was already initialized. |
| `AmountTooSmall()` | A deposit was too small to mint any shares, or a withdrawal too small to return anything. |
| `CrossedBook()` | The ask must be at or above the bid at every inventory, or the pool pays people to round-trip it. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `ExpiredPastDeadline()` | A liquidity modification order was attempted to be executed after the deadline. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InsufficientReserves()` | The pool has run out of the currency being bought. |
| `InvalidLadder()` | A ladder was configured with a zero band, a zero price, or a floor above its base. |
| `InvalidNativePayer(address)` | The native currency was settled on behalf of a `payer` other than the contract paying it. |
| `InvalidNativeValue()` | Native currency was not sent with the correct amount. |
| `LiquidityOnlyViaHook()` | Liquidity was attempted to be added or removed via the `PoolManager` instead of the hook. |
| `NoLiquidity()` | A quote was requested against an empty pool. |
| `PoolNotInitialized()` | Pool was not initialized. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |
| `SwapTooLarge()` | The swap would cross more than `MAX_STEPS` bands. |
| `TooMuchSlippage()` | Principal delta of liquidity modification resulted in too much slippage. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 0 of the fourteen:

- none

Mask: `0x0`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # GridLadder
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # curve, custom-curve, market-making, spread, inventory
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/grid-ladder
cd grid-ladder
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
