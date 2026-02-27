# Aave V4 Integration

## Quick Reference
- Fuse contracts: `AaveV4SupplyFuse.sol`, `AaveV4BorrowFuse.sol`, `AaveV4EModeFuse.sol`, `AaveV4BalanceFuse.sol`, `AaveV4SubstrateLib.sol`
- Market ID: `AAVE_V4 = 45`
- Chains: TBD (Aave V4 deployment chains)
- External contracts: Aave V4 Spoke contracts (passed per-call)

## Substrate Encoding
Aave V4 uses typed substrates encoded in bytes32. The high byte is a type flag:
- `0x01` = Asset (ERC20 token address)
- `0x02` = Spoke (Aave V4 Spoke contract address)

Both asset AND spoke must be granted substrates for any operation.

## Fuse Interface

### AaveV4SupplyFuse
Supplies and withdraws assets via Aave V4 Spoke contracts. Supports slippage protection via min shares/amounts.

```
enter(spoke, asset, reserveId, amount, minShares):
  -- Validates asset substrate AND spoke substrate are granted
  -- Validates spoke.getReserve(reserveId).underlying == asset
  -- finalAmount = min(balance(asset), amount)
  -- approve(spoke, finalAmount)
  -- shares = spoke.supply(reserveId, finalAmount, self)
  -- Reverts if shares < minShares
  => returns (asset, finalAmount)

exit(spoke, asset, reserveId, amount, minAmount):
  -- Validates asset + spoke substrates
  -- Validates reserve underlying matches asset
  -- supplyAssets = spoke.getUserSuppliedAssets(reserveId, self)
  -- finalAmount = min(supplyAssets, amount)
  -- withdrawnAmount = spoke.withdraw(reserveId, finalAmount, self)
  -- Reverts if withdrawnAmount < minAmount
  => returns (asset, withdrawnAmount)

instantWithdraw(params[amount, asset, spoke, reserveId, minAmount]):
  -- Same as exit but catches exceptions
```

### AaveV4BorrowFuse
Borrows and repays assets via Spoke contracts with slippage protection.

```
enter(spoke, asset, reserveId, amount, minShares):
  -- Validates asset + spoke substrates, validates reserve asset
  -- shares = spoke.borrow(reserveId, amount, self)
  -- Reverts if shares < minShares
  => returns (asset, amount)

exit(spoke, asset, reserveId, amount, minSharesRepaid):
  -- Validates asset + spoke substrates, validates reserve asset
  -- repayAmount = min(balance(asset), amount)
  -- approve(spoke, repayAmount)
  -- (sharesRepaid, repaid) = spoke.repay(reserveId, repayAmount, self)
  -- Reverts if sharesRepaid < minSharesRepaid
  => returns (asset, repaid)
```

### AaveV4EModeFuse
Sets E-Mode category on a Spoke. Enter-only (set eModeCategory=0 to disable).

```
enter(spoke, eModeCategory):
  -- Validates spoke substrate is granted
  -- spoke.setUserEMode(eModeCategory)
```

## Balance Tracking

```
balanceOf():
  for each Spoke substrate:
    for each reserve in spoke (0..reserveCount):
      skip if reserve.underlying is not a granted Asset substrate
      supply = spoke.getUserSuppliedAssets(reserveId, self)
      debt = spoke.getUserTotalDebt(reserveId, self)
      net = supply - debt
      balance += convertToWad(net * oraclePrice, assetDecimals + priceDecimals)
  Reverts if total balance < 0
  return balance  // USD in 18 decimals
```

Uses PriceOracleMiddleware from PlasmaVault storage.

## Key Invariants
- Both Asset and Spoke substrates must be granted (dual validation)
- Reserve underlying must match provided asset (protects against reserve index shifts)
- Slippage protection on supply/borrow via minShares, on withdraw via minAmount
- Negative total balance causes revert (not silent zero)

## Risks & Edge Cases
- Reserve index shifts from Aave governance could cause mismatched assets (mitigated by validation)
- Spoke contract upgrades could change behavior
- E-Mode affects all positions on the Spoke

## Related Files
- Source: `contracts/fuses/aave_v4/`
- Tests: `test/fuses/aave_v4/`
- External interfaces: `ext/IAaveV4Spoke.sol`, `ext/IPriceOracleGetter.sol`
