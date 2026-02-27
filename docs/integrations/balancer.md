# Balancer Integration

## Quick Reference
- Fuse contracts: `BalancerBalanceFuse.sol`, `BalancerGaugeFuse.sol`, `BalancerLiquidityProportionalFuse.sol`, `BalancerLiquidityUnbalancedFuse.sol`, `BalancerSingleTokenFuse.sol`, `BalancerSubstrateLib.sol`
- Market ID: BALANCER=36
- External contracts: Balancer Router (V3), Permit2, IPool, ILiquidityGauge

## Substrate System
Balancer uses a typed substrate encoding:
- `BalancerSubstrateType.POOL` = pool address (for liquidity operations)
- `BalancerSubstrateType.GAUGE` = gauge address (for staking)
- `BalancerSubstrateType.TOKEN` = token address (for exit token validation)

Encoded as: `bytes32 = address | (substrateType << 160)`

## Fuse Interface

### BalancerLiquidityProportionalFuse
```
enter(pool, tokens[], maxAmountsIn[], exactBptAmountOut)
  -> returns amountsIn[]
  - Adds liquidity proportionally via Router.addLiquidityProportional()
  - Uses Permit2 for token approvals (approve -> Permit2 -> Router)
  - Pool must be POOL substrate; tokens validated against pool.getTokenInfo()

exit(pool, exactBptAmountIn, minAmountsOut[])
  -> returns amountsOut[]
  - Removes liquidity proportionally; validates all pool tokens are TOKEN substrates
```

### BalancerLiquidityUnbalancedFuse
```
enter(pool, tokens[], exactAmountsIn[], minBptAmountOut)
  -> returns bptAmountOut
  - Unbalanced deposit; each token with amount > 0 must be TOKEN substrate

exit(pool, maxBptAmountIn, minAmountsOut[])
  -> returns (bptAmountIn, amountsOut[])
  - Custom removal via Router.removeLiquidityCustom()
  - All pool tokens must be TOKEN substrates
```

### BalancerSingleTokenFuse
```
enter(pool, tokenIn, maxAmountIn, exactBptAmountOut)
  -> returns amountIn
  - Single-token add via Router.addLiquiditySingleTokenExactOut()

exit(pool, tokenOut, maxBptAmountIn, exactAmountOut)
  -> returns bptAmountIn
  - Single-token remove; tokenOut must be TOKEN substrate
```

### BalancerGaugeFuse
```
enter(gaugeAddress, bptAmount, minBptAmount) -> depositAmount
  - Deposits BPT into gauge; gauge must be GAUGE substrate
  - Reverts if actual deposit < minBptAmount

exit(gaugeAddress, bptAmount, minBptAmount) -> withdrawAmount
  - Withdraws BPT from gauge
```

## Balance Tracking
`BalancerBalanceFuse.balanceOf()`:
- For POOL substrates: uses pool address directly, gets LP balance
- For GAUGE substrates: gets LP token from gauge.lp_token(), gets staked balance
- Proportional valuation: `amountOut[i] = balance[i] * lpBalance / totalSupply`
- Uses pool.getTokenInfo() for lastBalancesLiveScaled18 (18 decimal precision)
- Converts each token amount to USD via price oracle

## Key Invariants
- Approval flow: Token -> Permit2 -> Router (approvals cleaned up after each operation)
- Exit operations validate all output tokens are granted as TOKEN substrates
- minBptAmount on gauge operations acts as slippage protection
- balanceOf() is NOT view (calls external non-view functions)

## Risks & Edge Cases
- Permit2 approvals use `uint48(block.timestamp + 1)` expiration (1 second window)
- Exit token validation prevents withdrawing non-whitelisted tokens into vault
- Pool totalSupply of 0 would cause division by zero in balance calculation

## Related Files
- Source: `contracts/fuses/balancer/`
- Ext: `ext/IRouter.sol`, `ext/IPool.sol`, `ext/ILiquidityGauge.sol`, `ext/IPermit2.sol`
