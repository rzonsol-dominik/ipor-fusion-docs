# Velodrome Superchain Integration

## Quick Reference
- Fuse contracts (AMM): `VelodromeSuperchainLiquidityFuse.sol`, `VelodromeSuperchainGaugeFuse.sol`, `VelodromeSuperchainBalanceFuse.sol`
- Fuse contracts (Slipstream): `VelodromeSuperchainSlipstreamNewPositionFuse.sol`, `VelodromeSuperchainSlipstreamModifyPositionFuse.sol`, `VelodromeSuperchainSlipstreamCollectFuse.sol`, `VelodromeSuperchainSlipstreamLeafCLGaugeFuse.sol`, `VelodromeSuperchainSlipstreamBalanceFuse.sol`
- Market IDs: VELODROME_SUPERCHAIN=31, VELODROME_SUPERCHAIN_SLIPSTREAM=32
- External contracts: IRouter, ILeafGauge, NonfungiblePositionManager, ICLPool, ISlipstreamSugar, ILeafCLGauge

## Substrate System
Uses typed substrate encoding (same pattern as Aerodrome):
- `VelodromeSuperchainSubstrateType`: UNDEFINED, Gauge, Pool
- `VelodromeSuperchainSlipstreamSubstrateType`: UNDEFINED, Gauge, Pool

## Fuse Interface -- AMM (Market ID: 31)

### VelodromeSuperchainLiquidityFuse
```
enter(tokenA, tokenB, stable, amountADesired, amountBDesired, amountAMin, amountBMin, deadline)
  -> returns response(tokenA, tokenB, stable, amountA, amountB, liquidity)
  - Resolves pool via Router.poolFor(); pool must be Pool substrate
  - Adds liquidity via Router.addLiquidity()

exit(tokenA, tokenB, stable, liquidity, amountAMin, amountBMin, deadline)
  -> returns response(tokenA, tokenB, stable, amountA, amountB, liquidity)
```

### VelodromeSuperchainGaugeFuse
```
enter(gaugeAddress, amount, minAmount) -> (gaugeAddress, amount)
  - Deposits staking tokens into LeafGauge; gauge must be Gauge substrate
  - Validates actual deposit >= minAmount

exit(gaugeAddress, amount, minAmount) -> (gaugeAddress, amount)
  - Withdraws from LeafGauge; validates actual withdrawal >= minAmount
```

## Fuse Interface -- Slipstream (Market ID: 32)

### VelodromeSuperchainSlipstreamNewPositionFuse
```
enter(token0, token1, tickSpacing, tickLower, tickUpper, amount0Desired, amount1Desired, amount0Min, amount1Min, deadline, sqrtPriceX96)
  -> returns (tokenId, liquidity, amount0, amount1)
  - Pool computed from factory; must be Pool substrate
  - Stores tokenId in FuseStorageLib

exit(tokenIds[]) -> burns NFT positions
```

### VelodromeSuperchainSlipstreamModifyPositionFuse
Same pattern as Aerodrome Slipstream: increase/decrease liquidity on existing NFTs.

### VelodromeSuperchainSlipstreamCollectFuse
```
enter(tokenIds[]) -> (totalAmount0, totalAmount1)
  - Collects max fees from each NFT position
```

### VelodromeSuperchainSlipstreamLeafCLGaugeFuse
```
enter(gaugeAddress, tokenId) -> deposits NFT into LeafCL gauge
exit(gaugeAddress, tokenId)  -> withdraws NFT from LeafCL gauge
```

## Balance Tracking

### VelodromeSuperchainBalanceFuse (AMM)
- Pool positions: LP value (proportional reserves) + trading fees (index delta)
- Gauge positions: LP value only (VELO rewards handled by RewardsManager)
- Same fee index methodology as Aerodrome AMM balance fuse

### VelodromeSuperchainSlipstreamBalanceFuse (Slipstream)
- Uses curated storage-based position list (anti-DoS)
- Enforces maximum positions limit per substrate (reverts via TooManyPositions if exceeded)
- Pool positions: principal + fees via SlipstreamSugar
- Gauge positions: uses LeafCLGauge.stakedValues()

## Key Invariants
- Architecturally mirrors Aerodrome but uses Velodrome-specific contracts (LeafGauge, LeafCLGauge)
- AMM gauge staking: trading fees go to veVELO voters, NOT stakers
- Slipstream balance fuse has position count limits to prevent gas exhaustion
- minAmount parameters on gauge fuses provide slippage protection

## Risks & Edge Cases
- Position count exceeding max limit causes balanceOf() to revert (TooManyPositions)
- Gauge rewards (VELO) not counted in balance -- separate claim mechanism
- Pool reserves can be manipulated briefly (use TWAP-based strategies)

## Related Files
- Source: `contracts/fuses/velodrome_superchain/`, `contracts/fuses/velodrome_superchain_slipstream/`
