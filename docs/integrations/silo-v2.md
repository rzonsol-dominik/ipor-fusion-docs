# Silo V2 Integration

## Quick Reference
- Fuse contracts: `SiloV2SupplyBorrowableCollateralFuse.sol`, `SiloV2SupplyNonBorrowableCollateralFuse.sol`, `SiloV2BorrowFuse.sol`, `SiloV2BalanceFuse.sol`, `SiloV2SupplyCollateralFuseAbstract.sol`, `SiloIndex.sol`
- Market ID: `SILO_V2 = 35`
- Chains: Ethereum, Arbitrum
- External contracts: SiloConfig, Silo (per-pair), ShareTokens

## Substrate Model
Substrates are **SiloConfig addresses** (validated as assets via `isSubstrateAsAssetGranted`). Each SiloConfig manages a pair of Silos (Silo0 and Silo1), selected via the `SiloIndex` enum.

## Collateral Types
Silo V2 has two collateral types:
- **Collateral** (borrowable): deposited via `SiloV2SupplyBorrowableCollateralFuse`, can be borrowed against
- **Protected** (non-borrowable): deposited via `SiloV2SupplyNonBorrowableCollateralFuse`, cannot be borrowed by others

## Fuse Interface

### SiloV2SupplyBorrowableCollateralFuse / SiloV2SupplyNonBorrowableCollateralFuse
Both extend `SiloV2SupplyCollateralFuseAbstract` with different `CollateralType`.

```
enter(siloConfig, siloIndex, siloAssetAmount, minSiloAssetAmount):
  -- Validates siloConfig is granted substrate
  -- (silo0, silo1) = siloConfig.getSilos()
  -- silo = siloIndex == SILO0 ? silo0 : silo1
  -- asset = silo.asset()
  -- finalAmount = min(asset.balanceOf(self), siloAssetAmount)
  -- Reverts if finalAmount < minSiloAssetAmount
  -- approve(silo, finalAmount)
  -- shares = silo.deposit(finalAmount, self, collateralType)
  -- Reset approval to 0
  => returns (collateralType, siloConfig, silo, shares, finalAmount)

exit(siloConfig, siloIndex, siloShares, minSiloShares):
  -- Validates siloConfig is granted substrate
  -- silo = select silo by index
  -- finalShares = min(silo.maxRedeem(self, collateralType), siloShares)
  -- Reverts if finalShares < minSiloShares
  -- assetAmount = silo.redeem(finalShares, self, self, collateralType)
  => returns (collateralType, siloConfig, silo, finalShares, assetAmount)
```

### SiloV2BorrowFuse
Borrows and repays assets from Silo V2.

```
enter(siloConfig, siloIndex, siloAssetAmount):
  -- Validates siloConfig is granted substrate
  -- silo = select silo by index
  -- shares = silo.borrow(siloAssetAmount, self, self)
  => returns (siloConfig, silo, siloAssetAmount, shares)

exit(siloConfig, siloIndex, siloAssetAmount):
  -- Validates siloConfig is granted substrate
  -- silo = select silo by index
  -- approve(silo.asset(), siloAssetAmount)
  -- shares = silo.repay(siloAssetAmount, self)
  -- Reset approval to 0
  => returns (siloConfig, silo, siloAssetAmount, shares)
```

## Balance Tracking

```
balanceOf():
  for each siloConfig substrate:
    (silo0, silo1) = siloConfig.getSilos()
    for each silo in [silo0, silo1]:
      (protectedShareToken, collateralShareToken, debtShareToken) = siloConfig.getShareTokens(silo)
      grossAssets = silo.convertToAssets(protectedShares, Protected)
                  + silo.convertToAssets(collateralShares, Collateral)
      debtAssets = silo.convertToAssets(debtShares, Debt)
      siloAssets = grossAssets - debtAssets
      balance += convertToWad(siloAssets * price, decimals + priceDecimals)
  return max(0, balance)  // clamps to zero if negative
```

## Key Invariants
- Substrates are SiloConfig addresses (one config per lending pair)
- Each SiloConfig has two Silos (Silo0, Silo1), each for a different asset
- Collateral vs Protected determines whether deposited funds can be borrowed by others
- Exit uses shares (not assets) -- requires converting asset amounts to shares externally
- Slippage protection via minSiloAssetAmount (enter) and minSiloShares (exit)

## Risks & Edge Cases
- Silo pairs are isolated -- no cross-pair collateralization
- Debt can exceed collateral in extreme market conditions (balance goes negative, clamped to 0)
- Share token addresses must be fetched from SiloConfig, not hardcoded
- Protected collateral earns lower yield but cannot be borrowed

## Related Files
- Source: `contracts/fuses/silo_v2/`
- Tests: `test/fuses/silo_v2/`
- External interfaces: `ext/ISilo.sol`, `ext/ISiloConfig.sol`, `ext/IShareToken.sol`
