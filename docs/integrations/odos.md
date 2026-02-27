# Odos Integration

## Quick Reference
- Fuse contracts: `OdosSwapperFuse.sol`, `OdosSwapExecutor.sol`, `OdosSubstrateLib.sol`
- Market ID: `ODOS_SWAPPER = 42`

## Overview

Odos is a DEX aggregator integration that routes token swaps through the Odos Router V3. The fuse transfers tokens to a dedicated executor contract, which performs the swap via raw calldata from the Odos API, then returns all resulting tokens back to the PlasmaVault.

## Fuse Interface

```
OdosSwapperFuse:
  constructor(marketId)
    -> deploys OdosSwapExecutor
    -> stores EXECUTOR address

  enter(data: {tokenIn, tokenOut, amountIn, minAmountOut, swapCallData}):
    1. Validate tokenIn and tokenOut are non-zero and in granted substrates
    2. Validate amountIn > 0
    3. Record balances before swap
    4. Transfer amountIn of tokenIn to EXECUTOR
    5. Call EXECUTOR.execute(tokenIn, tokenOut, amountIn, swapCallData)
    6. Record balances after swap
    7. Validate tokenOutDelta >= minAmountOut (alpha slippage check)
    8. Validate USD slippage via PriceOracleMiddleware
    9. Emit OdosSwapperFuseEnter event
```

```
OdosSwapExecutor:
  ODOS_ROUTER = 0x0D05a7D3448512B78fa8A9e46c4872C88C4a0D05

  execute(tokenIn, tokenOut, amountIn, swapCallData):
    1. Approve Odos Router for amountIn
    2. Call ODOS_ROUTER with raw swapCallData
    3. Reset approval to 0
    4. Transfer remaining tokenIn back to caller
    5. Transfer all tokenOut to caller
```

## Substrate Configuration

Substrates are encoded as typed bytes32 values (first byte = type):

| Type | Value | Description |
|------|-------|-------------|
| Token | 1 | Allowed token address (tokenIn or tokenOut) |
| Slippage | 2 | Slippage limit in WAD (1e18 = 100%) |

If no slippage substrate is configured, the default of 1% (1e16) is used.

## Balance Tracking

No balance fuse -- Odos is a swap-only integration. All tokens return to the PlasmaVault after the swap.

## Key Invariants

- Both tokenIn and tokenOut must be registered as Token substrates for the market
- amountIn must be greater than zero
- The output amount must meet the alpha-specified minAmountOut
- USD slippage (based on oracle prices) must not exceed the configured limit (default 1%)
- The executor resets token approvals to zero after every swap
- The fuse has no storage variables (runs via delegatecall)

## Related Files
- Source: `contracts/fuses/odos/OdosSwapperFuse.sol`
- Source: `contracts/fuses/odos/OdosSwapExecutor.sol`
- Source: `contracts/fuses/odos/OdosSubstrateLib.sol`
