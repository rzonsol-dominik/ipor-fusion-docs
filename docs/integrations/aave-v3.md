# Aave V3 Integration

## Quick Reference
- Fuse contracts: `AaveV3SupplyFuse.sol`, `AaveV3BorrowFuse.sol`, `AaveV3CollateralFuse.sol`, `AaveV3BalanceFuse.sol`, `AaveV3WithPriceOracleMiddlewareBalanceFuse.sol`
- Market ID: `AAVE_V3 = 1`
- Chains: Ethereum, Arbitrum, others where Aave V3 is deployed
- External contracts: Aave V3 Pool (via PoolAddressesProvider)

## Fuse Interface

### AaveV3SupplyFuse
Supplies and withdraws assets to/from Aave V3 lending pools. Supports instant withdraw with exception handling.

```
enter(asset, amount, userEModeCategoryId):
  -- Validates asset is granted substrate for MARKET_ID
  -- finalAmount = min(balance(asset), amount)
  -- approve(aavePool, finalAmount)
  -- aavePool.supply(asset, finalAmount, self, 0)
  -- if userEModeCategoryId < 256: aavePool.setUserEMode(categoryId)
  => returns (asset, finalAmount)

exit(asset, amount):
  -- Validates asset is granted substrate
  -- Gets aToken address via PoolDataProvider
  -- finalAmount = min(aToken.balanceOf(self), amount)
  -- aavePool.withdraw(asset, finalAmount, self)
  => returns (asset, withdrawnAmount)

instantWithdraw(params[amount, asset]):
  -- Same as exit but catches exceptions (emits ExitFailed on error)
```

### AaveV3BorrowFuse
Borrows and repays assets using variable interest rate (mode=2).

```
enter(asset, amount):
  -- Validates asset is granted substrate
  -- aavePool.borrow(asset, amount, INTEREST_RATE_MODE=2, 0, self)
  => returns (asset, amount)

exit(asset, amount):
  -- Validates asset is granted substrate
  -- approve(aavePool, amount)
  -- aavePool.repay(asset, amount, INTEREST_RATE_MODE=2, self)
  => returns (asset, repaidAmount)
```

### AaveV3CollateralFuse
Enables/disables assets as isolated collateral.

```
enter(assetAddress):
  -- Validates asset is granted substrate
  -- pool.setUserUseReserveAsCollateral(asset, true)

exit(assetAddress):
  -- Validates asset is granted substrate
  -- pool.setUserUseReserveAsCollateral(asset, false)
```

## Balance Tracking

Two balance fuse variants exist:

**AaveV3BalanceFuse** (uses Aave's own price oracle, 8 decimal prices):
```
balanceOf():
  for each substrate asset:
    net = aToken.balanceOf - stableDebt.balanceOf - variableDebt.balanceOf
    balance += convertToWad(net * aaveOraclePrice, decimals + 8)
  return balance  // USD in 18 decimals
```

**AaveV3WithPriceOracleMiddlewareBalanceFuse** (uses IPOR PriceOracleMiddleware):
Same logic but fetches price from `PlasmaVaultLib.getPriceOracleMiddleware()` with dynamic price decimals.

## Key Invariants
- Assets must be registered as substrates (via `PlasmaVaultConfigLib.isSubstrateAsAssetGranted`)
- Supply amount is capped at vault's token balance
- Withdrawal amount is capped at aToken balance
- Borrow uses variable rate only (mode=2)
- eMode category only set when value < 256

## Risks & Edge Cases
- Aave liquidity crunch: withdrawals may fail if utilization is too high
- eMode changes affect collateral parameters for all positions
- Instant withdraw catches exceptions silently -- position may not fully unwind
- Price oracle failure (zero price) causes balance calculation to revert

## Related Files
- Source: `contracts/fuses/aave_v3/`
- Tests: `test/fuses/aave_v3/`
- External interfaces: `ext/IPool.sol`, `ext/IPoolAddressesProvider.sol`, `ext/IAavePoolDataProvider.sol`
