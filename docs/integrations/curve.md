# Curve Integration

## Quick Reference
- Fuse contracts: `CurveStableswapNGSingleSideSupplyFuse.sol`, `CurveStableswapNGSingleSideBalanceFuse.sol`, `CurveChildLiquidityGaugeSupplyFuse.sol`, `CurveChildLiquidityGaugeBalanceFuse.sol`, `CurveChildLiquidityGaugeErc4626BalanceFuse.sol`
- Market IDs: CURVE_POOL=16, CURVE_LP_GAUGE=17, CURVE_GAUGE_ERC4626=25
- External contracts: ICurveStableswapNG pool contracts, IChildLiquidityGauge gauge contracts

## Fuse Interface

### CurveStableswapNGSingleSideSupplyFuse (Market ID: 16)
```
enter(curveStableswapNG, asset, assetAmount, minLpTokenAmountReceived)
  -> returns (curveStableswapNG, lpTokenAmountReceived)
  - Single-sided deposit into Curve StableswapNG pool
  - Pool address (LP token) must be granted substrate
  - Asset must be one of the pool's coins (validated via N_COINS + coins(i))
  - Calls pool.add_liquidity(coinsAmounts, minLpTokenAmountReceived, address(this))

exit(curveStableswapNG, lpTokenAmount, asset, minCoinAmountReceived)
  -> returns (curveStableswapNG, coinAmountReceived)
  - Calls pool.remove_liquidity_one_coin()
  - Wrapped in try/catch: on failure emits ExitFailed event, returns 0
```

### CurveChildLiquidityGaugeSupplyFuse (Market ID: 17)
```
enter(childLiquidityGauge, lpTokenAmount)
  -> returns (childLiquidityGauge, lpTokenAmount)
  - Stakes LP tokens into Curve gauge
  - Gauge address must be granted substrate
  - Caps deposit to available LP token balance

exit(childLiquidityGauge, lpTokenAmount)
  -> returns (childLiquidityGauge, lpTokenAmount)
  - Unstakes LP tokens from gauge
  - Caps withdrawal to staked balance

instantWithdraw(params[amount, gaugeAddress])
  - ERC4626-only: converts underlying amount to LP shares via previewWithdraw()
  - Validates sufficient staked balance before withdrawal
```

## Balance Tracking

### CurveStableswapNGSingleSideBalanceFuse (Market ID: 16)
- Pro-rata share valuation: for each pool substrate
  - Gets LP balance, totalSupply, N_COINS
  - For each coin: coinAmount = pool.balances(j) * lpBalance / totalSupply
  - Converts each coin amount to USD via price oracle
  - Returns sum in WAD

### CurveChildLiquidityGaugeBalanceFuse (Market ID: 17)
- For each gauge substrate: gets LP token from gauge.lp_token()
- Calculates withdrawable amount via pool.calc_withdraw_one_coin(stakedBalance, coinIndex)
- Converts to USD using vault's underlying asset price

### CurveChildLiquidityGaugeErc4626BalanceFuse (Market ID: 25)
- For ERC4626-compatible LP tokens staked in gauges
- Uses ERC4626Upgradeable(lpToken).convertToAssets(stakedBalance) instead of calc_withdraw_one_coin
- Gauge balance is 1:1 with LP tokens (staking does not change ratio)

## Key Invariants
- Substrates for pool fuse are Curve pool LP token addresses
- Substrates for gauge fuse are gauge contract addresses
- Rewards (CRV emissions) are NOT included in balance -- handled separately
- Single-side operations only (no multi-asset balanced deposits)

## Risks & Edge Cases
- Exit wrapped in try/catch: if remove_liquidity_one_coin fails, emits failure event and returns 0
- calc_withdraw_one_coin may revert for illiquid pools (balance fuse handles gracefully)
- Gauge deposit/withdraw does not claim rewards by default (claim_rewards=false)

## Related Files
- Source: `contracts/fuses/curve_stableswap_ng/`, `contracts/fuses/curve_gauge/`
- Ext: `ext/ICurveStableswapNG.sol`, `ext/IChildLiquidityGauge.sol`
