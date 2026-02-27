# Yield Basis Integration

## Quick Reference
- Fuse contracts: `YieldBasisLtSupplyFuse.sol`, `YieldBasisLtBalanceFuse.sol`
- Market ID: YIELD_BASIS_LT=37
- External contracts: IYieldBasisLT (Leveraged Liquidity Token)

## Architecture
Yield Basis LT (Leveraged Liquidity Token) is NOT ERC4626 compatible. It uses a custom vault interface with:
- `deposit(assets, debt, minShares, receiver)` -- takes a debt parameter for leveraged positions
- `withdraw(shares, minAssets, receiver)` -- standard share-based withdrawal
- `emergency_withdraw(shares)` -- bypasses normal withdrawal logic
- `pricePerShare()` -- returns 18-decimal price per share

## Fuse Interface

### YieldBasisLtSupplyFuse
```
enter(ltAddress, ltAssetAmount, debt, minSharesToReceive)
  -> returns (ltAddress, ltAssetToken, ltAssetAmount, debt, ltSharesAmountReceived)
  - ltAddress must be granted substrate
  - Gets ASSET_TOKEN() from LT contract
  - Approves asset token to LT, calls LT.deposit(amount, debt, minShares, address(this))
  - debt: USD amount (18 decimals) for AMM debt position (approximately assetAmount * assetPrice)
  - Reverts if received shares < minSharesToReceive

exit(ltAddress, ltSharesAmount, minLtAssetAmountToReceive)
  -> returns (ltAddress, ltSharesAmount, ltAssetAmountReceived)
  - Calls LT.withdraw(shares, minAssets, address(this))
  - Reverts if received assets < minLtAssetAmountToReceive

instantWithdraw(params[amount, ltAddress])
  - Converts underlying PlasmaVault asset amount to LT shares:
    1. Convert to WAD: underlyingAmountInWad = convertToWad(amount, decimals)
    2. Calculate shares: ltSharesAmount = underlyingAmountInWad * 1e18 / lt.pricePerShare()
  - Caps to available LT balance
  - Calls _exit with minAssets=0 (no slippage protection in instant mode)
```

## Balance Tracking
`YieldBasisLtBalanceFuse.balanceOf()`:
- For each LT substrate:
  - `ltSharesInWad = convertToWad(lt.balanceOf(plasmaVault), lt.decimals())`
  - `ltAssetsInWad = ltSharesInWad * lt.pricePerShare() / 1e18`
  - Gets ASSET_TOKEN price from PriceOracleMiddleware
  - `balance += FullMath.mulDiv(ltAssetsInWad, assetPrice, 10 ** priceDecimals)`
- Uses FullMath.mulDiv for overflow-safe multiplication

## Key Invariants
- NOT ERC4626 compatible -- uses custom IYieldBasisLT interface
- Substrates are LT token addresses
- Price feed required for LT's ASSET_TOKEN (e.g., WBTC)
- debt parameter controls leverage level on deposit
- pricePerShare() used for share-to-asset conversion (18 decimal precision)
- instantWithdraw has no slippage protection (minAssets=0)

## Risks & Edge Cases
- Leveraged positions: debt parameter means the vault takes on AMM debt; incorrect debt can lead to bad positions
- pricePerShare() may be manipulatable in certain conditions
- emergency_withdraw available but not used in normal fuse operations
- instantWithdraw converts PlasmaVault underlying amount to LT shares using pricePerShare -- rounding may cause slight undershoot
- FullMath.mulDiv used to prevent overflow in large balance calculations

## Related Files
- Source: `contracts/fuses/yield_basis/`
- Ext: `ext/IYieldBasisLT.sol`
- Shared: `contracts/fuses/ramses/ext/FullMath.sol` (imported for overflow-safe math)
