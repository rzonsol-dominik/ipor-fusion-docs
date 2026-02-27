# Universal Token Swapper Integration

## Quick Reference
- Fuse contracts: `UniversalTokenSwapperFuse.sol`, `UniversalTokenSwapperEthFuse.sol`, `UniversalTokenSwapperWithVerificationFuse.sol`
- Executors: `SwapExecutor.sol`, `SwapExecutorEth.sol`, `SwapExecutorRestricted.sol`
- Library: `UniversalTokenSwapperSubstrateLib.sol`
- Market ID: `UNIVERSAL_TOKEN_SWAPPER = 12`

## Overview

A generic, DEX-agnostic swap framework that executes arbitrary swap calldata across one or more target DEX contracts. Three fuse variants exist: basic (ERC20 only), ETH-capable (handles native ETH and WETH wrapping), and a legacy verification variant with immutable slippage.

## Fuse Interface

### UniversalTokenSwapperFuse (basic)

```
constructor(marketId) -> deploys SwapExecutor

enter(data: {tokenIn, tokenOut, amountIn, minAmountOut,
             data: {targets[], data[]}}):
  1. Validate amountIn > 0, non-empty targets, array lengths match
  2. Single-pass substrate validation:
     - Check tokenIn, tokenOut granted (Token substrates)
     - Check all targets granted (Target substrates)
     - Read slippage (Slippage substrate, default 1%)
  3. Transfer tokenIn to executor
  4. Call executor.execute({tokenIn, tokenOut, dexs, dexsData})
  5. Validate minAmountOut and USD slippage
  6. Emit event, return deltas

enterTransient():
  -> Same logic but reads/writes via transient storage
```

### UniversalTokenSwapperEthFuse (ETH support)

```
constructor(marketId, wEth) -> deploys SwapExecutorEth

enter(data: {tokenIn, tokenOut, amountIn, minAmountOut,
             data: {targets[], callDatas[], ethAmounts[], tokensDustToCheck[]}}):
  1. Same validation as basic + array length check for ethAmounts
  2. Also validates tokensDustToCheck against Token substrates
  3. Executor handles ETH calls and wraps remaining ETH to WETH
```

### UniversalTokenSwapperWithVerificationFuse (legacy)

```
constructor(marketId, executor, slippageReverse)
  -> Uses externally-provided executor (SwapExecutorEth)
  -> Immutable slippage set at deploy time

enter(data):
  -> Validates substrates using (functionSelector, target) encoding
  -> Checks function selectors from calldata against substrates
```

## Substrate Configuration

Three substrate types (first byte = type):

| Type | Value | Description |
|------|-------|-------------|
| Token | 1 | Allowed tokenIn/tokenOut address |
| Target | 2 | Allowed DEX/router address |
| Slippage | 3 | Slippage limit in WAD |

Default slippage: 1% (1e16). Slippage > 100% reverts.

The legacy `WithVerification` variant uses a different encoding: `[selector (4B) | address (20B)]` packed into bytes32.

## Balance Tracking

No balance fuse -- swap-only integration. All tokens return to PlasmaVault after execution.

## Key Invariants

- All target DEX addresses must be registered as Target substrates
- Both tokenIn and tokenOut must be registered as Token substrates
- Dust check tokens (Eth variant) must also be Token substrates
- targets, callDatas, and ethAmounts arrays must have matching lengths
- SwapExecutor uses ReentrancyGuard for external DEX calls
- SwapExecutorRestricted limits callers to a single authorized address
- SwapExecutorEth wraps any remaining native ETH to WETH after execution

## Related Files
- Source: `contracts/fuses/universal_token_swapper/UniversalTokenSwapperFuse.sol`
- Source: `contracts/fuses/universal_token_swapper/UniversalTokenSwapperEthFuse.sol`
- Source: `contracts/fuses/universal_token_swapper/UniversalTokenSwapperWithVerificationFuse.sol`
- Source: `contracts/fuses/universal_token_swapper/SwapExecutor.sol`
- Source: `contracts/fuses/universal_token_swapper/SwapExecutorEth.sol`
- Source: `contracts/fuses/universal_token_swapper/SwapExecutorRestricted.sol`
- Source: `contracts/fuses/universal_token_swapper/UniversalTokenSwapperSubstrateLib.sol`
