# Ramses Integration

## Quick Reference
- Fuse contracts: `RamsesV2NewPositionFuse.sol`, `RamsesV2ModifyPositionFuse.sol`, `RamsesV2CollectFuse.sol`, `RamsesV2Balance.sol`
- Market ID: RAMSES_V2_POSITIONS=18
- External contracts: INonfungiblePositionManagerRamses, IRamsesV2Pool, Ramses Factory

## Fuse Interface

### RamsesV2NewPositionFuse
```
enter(token0, token1, fee, tickLower, tickUpper, amount0Desired, amount1Desired, amount0Min, amount1Min, deadline, veRamTokenId)
  -> returns (tokenId, liquidity, amount0, amount1)
  - Mints new NFT liquidity position via RamsesPositionManager.mint()
  - veRamTokenId: Ramses-specific boosting parameter for veRAM token ID
  - Both tokens must be granted substrates
  - Stores tokenId in FuseStorageLib

exit(tokenIds[])
  -> burns NFT positions
  - Removes from storage using swap-and-pop pattern
```

### RamsesV2ModifyPositionFuse
```
enter(token0, token1, tokenId, amount0Desired, amount1Desired, amount0Min, amount1Min, deadline)
  -> returns (tokenId, liquidity, amount0, amount1)
  - Increases liquidity on existing Ramses position

exit(tokenId, liquidity, amount0Min, amount1Min, deadline)
  -> returns (tokenId, amount0, amount1)
  - Decreases liquidity (does NOT transfer tokens; must collect separately)
```

### RamsesV2CollectFuse
```
enter(tokenIds[])
  -> returns (totalAmount0, totalAmount1)
  - Collects accumulated fees from multiple positions
  - Sets max collect amounts to uint128.max
```

## Balance Tracking
`RamsesV2Balance.balanceOf()`:
- Iterates stored NFT tokenIds from FuseStorageLib
- For each position: gets token0, token1, fee from position manager
- Gets sqrtPriceX96 from pool slot0
- Calculates position value using LiquidityAmounts and TickMath libraries
- Includes principal liquidity value only (fees via separate mechanism)
- Converts to USD via PriceOracleMiddleware, returns WAD

## Key Invariants
- Architecture mirrors UniswapV3 with Ramses-specific extensions (veRamTokenId)
- Position NFTs tracked in FuseStorageLib to prevent DoS
- Both tokens must be granted substrates for new positions
- decreaseLiquidity does NOT transfer tokens -- must collect afterward

## Risks & Edge Cases
- veRamTokenId boosting: incorrect token ID may reduce rewards
- Position burn requires zero liquidity and all fees collected
- Ramses uses its own PositionManager interface (slightly different from Uniswap's)
- Fee calculation may differ from Uniswap V3 due to Ramses-specific modifications

## Related Files
- Source: `contracts/fuses/ramses/`
- Ext: `ext/INonfungiblePositionManagerRamses.sol`, `ext/LiquidityAmounts.sol`, `ext/TickMath.sol`, `ext/PoolAddress.sol`
