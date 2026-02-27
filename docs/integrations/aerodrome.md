# Aerodrome Integration

## Quick Reference
- Fuse contracts (AMM): `AerodromeLiquidityFuse.sol`, `AerodromeGaugeFuse.sol`, `AerodromeClaimFeesFuse.sol`, `AerodromeBalanceFuse.sol`
- Fuse contracts (Slipstream/CL): `AreodromeSlipstreamNewPositionFuse.sol`, `AreodromeSlipstreamModifyPositionFuse.sol`, `AreodromeSlipstreamCollectFuse.sol`, `AreodromeSlipstreamCLGaugeFuse.sol`, `AreodromeSlipstreamBalanceFuse.sol`
- Market IDs: AERODROME=30, AREODROME_SLIPSTREAM=33
- External contracts: Aerodrome Router, IGauge, ICLGauge, NonfungiblePositionManager, SlipstreamSugar

## Substrate System
Both AMM and Slipstream use typed substrate encoding:
- `AerodromeSubstrateType.Pool` / `AreodromeSlipstreamSubstrateType.Pool`
- `AerodromeSubstrateType.Gauge` / `AreodromeSlipstreamSubstrateType.Gauge`

## Fuse Interface -- AMM (Market ID: 30)

### AerodromeLiquidityFuse
```
enter(tokenA, tokenB, stable, amountADesired, amountBDesired, amountAMin, amountBMin, deadline)
  -> returns (tokenA, tokenB, stable, amountA, amountB, liquidity)
  - Pool resolved via Router.poolFor(tokenA, tokenB, stable, address(0))
  - Pool must be Pool substrate; reverts if liquidity == 0

exit(tokenA, tokenB, stable, liquidity, amountAMin, amountBMin, deadline)
  -> returns (tokenA, tokenB, stable, amountA, amountB, liquidity)
```

### AerodromeGaugeFuse
```
enter(gaugeAddress, amount) -> (gaugeAddress, amount)
  - Deposits LP tokens (stakingToken) into gauge; caps to available balance
exit(gaugeAddress, amount) -> (gaugeAddress, amount)
  - Withdraws from gauge; caps to staked balance
```

### AerodromeClaimFeesFuse
```
enter(pools[]) -> (totalClaimed0, totalClaimed1)
  - Claims accumulated trading fees from multiple pools
  - Each pool must be Pool substrate
```

## Fuse Interface -- Slipstream/CL (Market ID: 33)

### AreodromeSlipstreamNewPositionFuse
```
enter(token0, token1, tickSpacing, tickLower, tickUpper, amount0Desired, amount1Desired, amount0Min, amount1Min, deadline, sqrtPriceX96)
  -> returns (tokenId, liquidity, amount0, amount1)
  - Pool computed from factory + token0/token1/tickSpacing
  - Stores tokenId in FuseStorageLib for balance tracking

exit(tokenIds[]) -> burns NFT positions, removes from storage
```

### AreodromeSlipstreamModifyPositionFuse
```
enter(token0, token1, tokenId, amount0Desired, amount1Desired, amount0Min, amount1Min, deadline)
  -> returns (tokenId, liquidity, amount0, amount1)
  - token0/token1 fields IGNORED; actual tokens derived from position NFT
  - Validates deadline not expired, at least one amount > 0

exit(tokenId, liquidity, amount0Min, amount1Min, deadline)
  -> returns (tokenId, amount0, amount1)
```

### AreodromeSlipstreamCollectFuse
```
enter(tokenIds[]) -> (totalAmount0, totalAmount1)
  - Collects max fees from each position
```

### AreodromeSlipstreamCLGaugeFuse
```
enter(gaugeAddress, tokenId) -> deposits NFT into CL gauge
exit(gaugeAddress, tokenId)  -> withdraws NFT from CL gauge
```

## Balance Tracking

### AerodromeBalanceFuse (AMM)
- Pool positions: LP value (proportional reserves) + trading fees (index delta calculation)
- Gauge positions: LP value only (rewards via RewardsManager)
- Fee calculation: claimable + (liquidity * (globalIndex - supplyIndex)) / 1e18

### AreodromeSlipstreamBalanceFuse (CL)
- Pool positions: uses curated storage list to prevent DoS; filters by pool match
- Calculates principal + fees via SlipstreamSugar helper
- Gauge positions: uses gauge.stakedValues() for staked NFTs

## Key Invariants
- AMM gauge rewards (AERO emissions) NOT in balance; handled by RewardsManager
- CL positions tracked in FuseStorageLib to prevent DoS via arbitrary NFT transfers
- ModifyPosition derives actual tokens from NFT (ignores user-provided token0/token1)

## Risks & Edge Cases
- AMM fee index deltas can accumulate between balance checks
- CL position burn requires zero liquidity and all fees collected first
- Gauge staking stops fee accrual for AMM positions (fees go to veAERO voters instead)

## Related Files
- Source: `contracts/fuses/aerodrome/`, `contracts/fuses/aerodrome_slipstream/`
