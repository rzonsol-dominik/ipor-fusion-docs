# Async Action Integration

## Quick Reference
- Fuse contracts: `AsyncActionFuse.sol`, `AsyncActionBalanceFuse.sol`
- Executor: `AsyncExecutor.sol`
- Library: `AsyncActionFuseLib.sol`
- Market ID: `ASYNC_ACTION = 40`

## Overview

The Async Action fuse provides a framework for executing operations where the result is not immediately available (e.g., cross-chain bridges, pending swaps). It transfers tokens to a persistent AsyncExecutor, executes a batch of arbitrary calls, tracks expected balance for NAV reporting, and later fetches returned assets with slippage validation.

## Fuse Interface

### AsyncActionFuse

```
constructor(marketId, wEth)

enter(data: {tokenOut, amountOut, targets[], callDatas[],
             ethAmounts[], tokensDustToCheck[]}):
  1. Validate tokenOut != address(0) and array lengths match
  2. Decode substrates into:
     - AllowedAmountToOutside[] (token + max amount pairs)
     - AllowedTargets[] (target + selector pairs)
     - AllowedSlippage (WAD-based exit slippage)
  3. Validate tokenOut is in allowed amounts and amountOut <= limit
  4. Validate each target/selector pair is in allowed targets
  5. Deploy executor if not exists (ERC-7201 storage)
  6. If amountOut > 0 and executor balance == 0: transfer tokens
  7. If amountOut > 0 and executor balance > 0: revert
  8. Call executor.execute({tokenIn, targets, callDatas, ethAmounts, priceOracle})
  9. Executor caches balance in underlying asset units

exit(data: {assets[], fetchCallDatas[]}):
  1. Decode substrates for allowedTargets and allowedSlippage
  2. Validate targets/selectors
  3. Call executor.fetchAssets(assets, priceOracle, slippage)
     -> Sums USD value of all held assets
     -> Converts to underlying units
     -> Validates >= (cachedBalance - slippage%)
     -> Transfers all assets to PlasmaVault
     -> Resets cached balance to 0

enterTransient() / exitTransient():
  -> Same logic using transient storage
```

### AsyncExecutor

```
execute(data: {tokenIn, targets[], callDatas[], ethAmounts[], priceOracle}):
  1. If cached balance == 0: compute and cache balance from token deposits
  2. Execute each call (with or without ETH value)
  3. Tokens remain in executor until fetchAssets is called

fetchAssets(assets[], priceOracle, slippage):
  1. Calculate total USD value of all held assets
  2. Convert to underlying amount
  3. Validate: actual >= cached - (cached * slippage / 1e18)
  4. Transfer all assets to PlasmaVault
  5. Reset cached balance to 0
```

## Substrate Configuration

Three substrate types (first byte = type discriminator):

| Type | Value | Description |
|------|-------|-------------|
| ALLOWED_AMOUNT_TO_OUTSIDE | 0 | Asset address (20B) + max amount (uint88, 11B) |
| ALLOWED_TARGETS | 1 | Target address (20B) + selector (4B) |
| ALLOWED_EXIT_SLIPPAGE | 2 | Slippage in WAD (uint248) |

## Balance Tracking

`AsyncActionBalanceFuse.balanceOf()`:
- Reads executor address from ERC-7201 storage
- Reads `executor.balance()` (cached value in underlying asset units)
- Converts to USD using PriceOracleMiddleware
- Returns 0 if executor not deployed or balance is zero

## Key Invariants

- Cannot enter with amountOut > 0 if executor already has a cached balance
- Token amount limits are enforced per-asset via substrates (uint88 max)
- Every target/selector pair must be explicitly whitelisted
- Exit slippage validation ensures returned assets meet minimum threshold
- Executor is only callable by the PlasmaVault
- Balance tracks expected value for NAV even when assets are in transit

## Related Files
- Source: `contracts/fuses/async_action/AsyncActionFuse.sol`
- Source: `contracts/fuses/async_action/AsyncActionBalanceFuse.sol`
- Source: `contracts/fuses/async_action/AsyncActionFuseLib.sol`
- Source: `contracts/fuses/async_action/AsyncExecutor.sol`
