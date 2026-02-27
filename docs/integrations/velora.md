# Velora Integration

## Quick Reference
- Fuse contracts: `VeloraSwapperFuse.sol`, `VeloraSwapExecutor.sol`, `VeloraSubstrateLib.sol`
- Market ID: `VELORA_SWAPPER = 43`

## Overview

Velora integrates with ParaSwap Augustus v6.2 for optimized token swapping. The architecture mirrors the Odos fuse: a dedicated executor receives tokens, performs the swap via raw calldata, and returns results to the PlasmaVault. It includes a single-pass substrate validation optimization.

## Fuse Interface

```
VeloraSwapperFuse:
  constructor(marketId)
    -> deploys VeloraSwapExecutor
    -> stores EXECUTOR address

  enter(data: {tokenIn, tokenOut, amountIn, minAmountOut, swapCallData}):
    1. Validate amountIn > 0 and tokenIn != tokenOut
    2. Single-pass substrate validation:
       - Check tokenIn granted, tokenOut granted, read slippage
    3. Record balances before swap
    4. Transfer amountIn of tokenIn to EXECUTOR
    5. Call EXECUTOR.execute(tokenIn, tokenOut, amountIn, swapCallData)
    6. Record balances after swap
    7. Validate tokenOutDelta >= minAmountOut
    8. Validate USD slippage via PriceOracleMiddleware
    9. Emit VeloraSwapperFuseEnter event
```

```
VeloraSwapExecutor:
  AUGUSTUS_V6_2 = 0x6A000F20005980200259B80c5102003040001068

  execute(tokenIn, tokenOut, amountIn, swapCallData):
    1. Approve Augustus v6.2 for amountIn
    2. Call AUGUSTUS_V6_2 with raw swapCallData
    3. Reset approval to 0
    4. Transfer remaining tokenIn back to caller
    5. Transfer all tokenOut to caller
```

## Substrate Configuration

Same typed substrate layout as Odos (first byte = type):

| Type | Value | Description |
|------|-------|-------------|
| Token | 1 | Allowed token address |
| Slippage | 2 | Slippage limit in WAD |

Default slippage fallback: 1% (1e16). A slippage value of 0 also falls back to default.

## Balance Tracking

No balance fuse -- swap-only integration with immediate token return.

## Key Invariants

- tokenIn and tokenOut must differ (same-token swap is rejected)
- Both tokens must be registered as Token substrates
- Single-pass substrate validation reads all substrates once for gas efficiency
- USD slippage protection via oracle (default 1%)
- Executor resets approvals to zero after each swap
- Fuse has no storage variables

## Related Files
- Source: `contracts/fuses/velora/VeloraSwapperFuse.sol`
- Source: `contracts/fuses/velora/VeloraSwapExecutor.sol`
- Source: `contracts/fuses/velora/VeloraSubstrateLib.sol`
