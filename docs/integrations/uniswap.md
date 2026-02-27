# Uniswap Integration

## Quick Reference
- Fuse contracts: `UniswapV2SwapFuse.sol`, `UniswapV3SwapFuse.sol`, `UniswapV3NewPositionFuse.sol`, `UniswapV3ModifyPositionFuse.sol`, `UniswapV3CollectFuse.sol`, `UniswapV3Balance.sol`
- Market IDs: UNISWAP_SWAP_V3_POSITIONS=8, UNISWAP_SWAP_V2=9, UNISWAP_SWAP_V3=10
- External contracts: UniversalRouter, NonfungiblePositionManager, UniswapV3Factory

## Fuse Interface

### UniswapV2SwapFuse (Market ID: 9)
```
enter(tokenInAmount, path[], minOutAmount)
  - path: array of token addresses [tokenIn, ..., tokenOut]
  - Transfers tokenIn to UniversalRouter, executes V2_SWAP_EXACT_IN (0x08)
  - Caps inputAmount to vault's balance of path[0]
  - All tokens in path must be granted substrates
  - No exit function (swap-only)
```

### UniswapV3SwapFuse (Market ID: 10)
```
enter(tokenInAmount, minOutAmount, path)
  - path: packed bytes [address, uint24 fee, address, uint24 fee, ..., address]
  - Executes V3_SWAP_EXACT_IN (0x00) via UniversalRouter
  - Caps inputAmount to vault's balance of first token
  - All tokens in path must be granted substrates
  - No exit function (swap-only)
```

### UniswapV3NewPositionFuse (Market ID: 8)
```
enter(token0, token1, fee, tickLower, tickUpper, amount0Desired, amount1Desired, amount0Min, amount1Min, deadline)
  -> returns (tokenId, liquidity, amount0, amount1, token0, token1, fee, tickLower, tickUpper)
  - Mints new NFT via NonfungiblePositionManager.mint()
  - Stores tokenId in FuseStorageLib for balance tracking
  - Both tokens must be granted substrates

exit(tokenIds[])
  -> burns NFT positions
  - Only burns if position value < half of token decimals (dust check)
  - Removes tokenId from storage using swap-and-pop pattern
```

### UniswapV3ModifyPositionFuse (Market ID: 8)
```
enter(token0, token1, tokenId, amount0Desired, amount1Desired, amount0Min, amount1Min, deadline)
  -> returns (tokenId, liquidity, amount0, amount1)
  - Increases liquidity on existing position

exit(tokenId, liquidity, amount0Min, amount1Min, deadline)
  -> returns (tokenId, amount0, amount1)
  - Decreases liquidity (does NOT transfer tokens to caller, must collect separately)
```

### UniswapV3CollectFuse (Market ID: 8)
```
enter(tokenIds[])
  -> returns (totalAmount0, totalAmount1)
  - Collects accumulated fees from positions
  - Sets amount0Max/amount1Max to uint128.max (collect all)
```

## Balance Tracking
`UniswapV3Balance.balanceOf()`:
- Iterates stored NFT tokenIds from FuseStorageLib
- For each position: extracts token0, token1, fee via assembly
- Gets sqrtPriceX96 from pool's slot0
- Calculates principal + fees via PositionValue.total()
- Converts both token amounts to USD via PriceOracleMiddleware
- Returns sum in WAD (18 decimals)

## Key Invariants
- All tokens in swap paths and LP positions must be granted as substrates for the market
- Swap fuses cap input to vault balance (no revert on insufficient balance, just uses available)
- Position NFTs are tracked in FuseStorageLib to prevent unbounded enumeration
- Exit (burn) only succeeds if position has dust-level remaining value

## Risks & Edge Cases
- V3 decreaseLiquidity does NOT transfer tokens; must call collect separately
- Swap fuses have no exit -- swaps are one-directional operations
- Position burn fails if any non-dust liquidity or uncollected fees remain
- Path encoding differs between V2 (address[]) and V3 (packed bytes with fees)

## Related Files
- Source: `contracts/fuses/uniswap/`
- Ext: `ext/INonfungiblePositionManager.sol`, `ext/IUniversalRouter.sol`, `ext/PositionValue.sol`
