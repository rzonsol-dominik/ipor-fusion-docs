# PriceOracleMiddleware

## Quick Reference
- Source: `contracts/price_oracle/PriceOracleMiddleware.sol`
- Variant: `PriceOracleMiddlewareWithRoles.sol` (adds role-based access)
- Proxy: UUPS upgradeable, Ownable2Step
- Output: All prices in 18-decimal WAD USD

## Interface
```
getAssetPrice(address asset) → (uint256 price, uint256 decimals)
  — Returns price in WAD (18 dec), decimals always = 18.
  — Checks custom feed first, then Chainlink Feed Registry fallback.
  — Reverts: UnsupportedAsset (zero addr or no feed), UnexpectedPriceResult (price <= 0).

getAssetsPrices(address[] assets) → (uint256[] prices, uint256[] decimals)
  — Batch version. Atomic — if any asset fails, entire call reverts.

setAssetsPricesSources(address[] assets, address[] sources) — onlyOwner
  — Set custom price feed for each asset. Source must implement IPriceFeed interface.

getSourceOfAssetPrice(address asset) → address
  — Returns custom feed address (address(0) if using Chainlink fallback).
```

## Price Feed Resolution
1. Check custom feed: `PriceOracleMiddlewareStorageLib.getSourceOfAssetPrice(asset)`
2. If custom feed exists: call `IPriceFeed(source).latestRoundData()` + `decimals()`
3. If no custom feed: fall back to `CHAINLINK_FEED_REGISTRY.latestRoundData(asset, USD)`
4. If `CHAINLINK_FEED_REGISTRY == address(0)`: revert (Chainlink disabled)
5. Convert to WAD: `IporMath.convertToWad(price, feedDecimals)`

## Custom Price Feeds Available
| Feed | Source | Purpose |
|------|--------|---------|
| ERC4626PriceFeed | price_feed/ERC4626PriceFeed.sol | Price from ERC4626 vault exchange rate |
| CurveStableSwapNGPriceFeed | price_feed/CurveStableSwapNGPriceFeed.sol | Curve pool LP price |
| PtPriceFeed | price_feed/PtPriceFeed.sol | Pendle PT token price |
| CollateralTokenOnMorphoMarketPriceFeed | price_feed/CollateralTokenOnMorphoMarketPriceFeed.sol | Morpho collateral price |
| DualCrossReferencePriceFeed | price_feed/DualCrossReferencePriceFeed.sol | Cross-reference two feeds |
| BeefyVaultV7PriceFeed | price_feed/BeefyVaultV7PriceFeed.sol | Beefy vault share price |
| FixedValuePriceFeed | price_feed/FixedValuePriceFeed.sol | Hardcoded price (stablecoins) |
| OneValuePriceFeed | price_feed/OneValuePriceFeed.sol | Always returns 1 USD |
| USDPriceFeed | price_feed/USDPriceFeed.sol | USD reference |
| WETHPriceFeed | price_feed/WETHPriceFeed.sol | WETH via Chainlink |
| Chain-specific | price_feed/chains/ethereum/*.sol | SDai, WstETH, EthPlus, SrUsd |
| Chain-specific | price_feed/chains/arbitrum/*.sol | USDM |

## Constants
- `QUOTE_CURRENCY = 0x0000000000000000000000000000000000000348` (Chainlink USD standard)
- `QUOTE_CURRENCY_DECIMALS = 18`
- `CHAINLINK_FEED_REGISTRY`: immutable, set in constructor. Ethereum mainnet only for Chainlink Registry.

## Related Files
- IPriceFeed.sol — interface: `latestRoundData()`, `decimals()`
- PriceOracleMiddlewareStorageLib.sol — asset→source mapping storage
- IporMath.sol — WAD conversion utilities
