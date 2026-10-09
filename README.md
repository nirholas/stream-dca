# StreamDCA

**A standing order that buys more when the price falls, settled inside other people's swaps.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://stream-dca.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/StreamDCAHook.sol`](src/hooks/StreamDCAHook.sol)
- **Licence:** Apache-2.0

## How it works

Every on-chain recurring buy built so far spends a fixed amount per period. That is dollar-cost averaging, and it is popular because it is easy to explain rather than because it is good: it spends the same money whether the asset just halved or just doubled, which is to say it treats the two identically and buys the most units in exactly the periods that matter least. The strategy that fixes this is older than crypto.

Value averaging, described by Michael Edleson in 1988, commits to growing the *position* by a fixed amount each period rather than spending a fixed amount. If the price fell, the position is behind its target and the period buys more; if it rose, the position is ahead and the period buys less or nothing. It beats dollar-cost averaging on essentially every historical series, and it has not been built on-chain because it needs a price at every period and a recomputation per participant, which is exactly the shape of thing a keeper bot is bad at and a hook is good at.

A stream here names a budget, a period, and how much value the position should gain each period. Nothing runs on a schedule: due streams are settled inside the next swap anybody makes on the pool, in one batched trade per direction, priced by the pool itself. That batching is what makes it affordable, and it is also what makes it fair, since every stream due in the same block gets the same price rather than a queue position.

The reference price is the pool's own. That is deliberate and it is the sharp edge: a manipulated pool price makes a stream buy the wrong amount, so this belongs on pools deep enough that moving them costs more than the streams behind them are worth.

## Prior art

TWAMM, both the original paper and the v4 hooks implementing it, splits a large order over time at a constant rate. Mean Finance and the DCA protocols run keeper bots on fixed-size periodic buys. Sablier and Superfluid stream tokens without trading them. Value averaging itself is a 1988 equity strategy with no on-chain implementation. Making the per-period amount a function of how far the position is from its target value, priced by the pool and settled in batches inside unrelated swaps, is the contribution here.

## Where it does not help

The pool's own price is the reference, so a pool shallow enough to move cheaply is a pool where streams can be made to buy at the wrong size; this belongs on deep pools. Streams settle when somebody swaps, so a pool with no flow leaves them late until anybody calls `poke`, and a late period is executed at the price it is finally seen at rather than the price it was due at. Work per swap is capped, so a pool with more due streams than the cap catches up over several swaps rather than all at once. A stream can also finish under budget, since a position that keeps running ahead of target never spends the rest.

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
| `CallbackNotPoolManager()` | Only the `PoolManager` may drive the unlock callback. |
| `InvalidStream()` | A stream with no budget, no periods, or a zero-length period is not a stream. |
| `NoSuchStream()` | There is no stream at that index on this pool. |
| `NotYourStream()` | Only the account that opened a stream may change or close it. |
| `NothingToWithdraw()` | The stream has nothing left to withdraw. |
| `SafeCastOverflowedIntToUint(int256)` | An int value doesn't fit in a uint of `bits` size. |
| `SafeCastOverflowedUintDowncast(uint8,uint256)` | Value doesn't fit in a uint of `bits` size. |
| `SafeCastOverflowedUintToInt(uint256)` | A uint value doesn't fit in an int of `bits` size. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 1 of the fourteen:

- `beforeSwap`

Mask: `0x80`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # StreamDCA
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # order-types, dca, value-averaging, keeper-free, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/stream-dca
cd stream-dca
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

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/stream-dca&type=Date)](https://www.star-history.com/#nirholas/stream-dca&Date)
