# Euler V2 Integration

## Quick Reference
- Fuse contracts: `EulerV2SupplyFuse.sol`, `EulerV2BorrowFuse.sol`, `EulerV2CollateralFuse.sol`, `EulerV2ControllerFuse.sol`, `EulerV2BatchFuse.sol`, `EulerV2BalanceFuse.sol`, `EulerFuseLib.sol`
- Market ID: `EULER_V2 = 11`
- Chains: Ethereum, others where Euler V2 is deployed
- External contracts: Euler vaults (ERC4626), Ethereum Vault Connector (EVC)

## Substrate Model
Substrates are packed `EulerSubstrate` structs encoded as bytes32:
```
| bits 255..96 | 95..88      | 87..80    | 79..72      |
| eulerVault   | isCollateral| canBorrow | subAccounts |
```

Permissions are encoded per-substrate: `isCollateral` and `canBorrow` flags control which operations are allowed. The `subAccounts` byte identifies which sub-account to use.

Sub-accounts: `subAccountAddress = plasmaVault XOR uint8(subAccountByte)`

## Fuse Interface

### EulerV2SupplyFuse
Deposits and withdraws from Euler V2 vaults (ERC4626). Uses EVC for secure interaction.

```
enter(eulerVault, maxAmount, subAccount):
  -- Validates canSupply(MARKET_ID, vault, subAccount) via EulerFuseLib
  -- transferAmount = min(maxAmount, vaultAsset.balanceOf(self))
  -- approve(eulerVault, transferAmount)
  -- EVC.call(eulerVault, self, 0, deposit(transferAmount, subAccountAddr))
  -- Reset approval to 0
  => returns mintedShares

exit(eulerVault, maxAmount, subAccount):
  -- No substrate validation (allows withdrawal from any existing position)
  -- finalAmount = min(maxAmount, vault.convertToAssets(vault.balanceOf(subAccount)))
  -- EVC.call(eulerVault, subAccount, 0, withdraw(finalAmount, self, subAccount))
  => returns withdrawnAssets

instantWithdraw(params[amount, vault, subAccount]):
  -- Validates canInstantWithdraw (isCollateral=false AND canBorrow=false)
  -- Same as exit but catches exceptions
```

### EulerV2BorrowFuse
Borrows and repays from Euler V2 vaults.

```
enter(eulerVault, assetAmount, subAccount):
  -- Validates canBorrow(MARKET_ID, vault, subAccount)
  -- EVC.call(vault, subAccount, 0, borrow(assetAmount, self))
  => returns (vault, borrowAmount, subAccount)

exit(eulerVault, maxAssetAmount, subAccount):
  -- Validates canBorrow
  -- approve(asset, max) to vault
  -- EVC.call(vault, self, 0, repay(maxAssetAmount, subAccount))
  -- Reset approval to 0
  => returns (vault, repayAmount, subAccount)
```

### EulerV2CollateralFuse
Enables/disables vaults as collateral via EVC.

```
enter(eulerVault, subAccount):
  -- Validates canCollateral
  -- EVC.enableCollateral(subAccountAddr, vault)

exit(eulerVault, subAccount):
  -- EVC.disableCollateral(subAccountAddr, vault)
```

### EulerV2ControllerFuse
Enables/disables vaults as controllers via EVC.

```
enter(eulerVault, subAccount):
  -- Validates canBorrow
  -- EVC.enableController(subAccountAddr, vault)

exit(eulerVault, subAccount):
  -- Validates canSupply
  -- EVC.call(vault, subAccount, 0, disableController())
```

### EulerV2BatchFuse
Executes multiple Euler V2 operations atomically via EVC batch. Enter-only (exit reverts).

```
enter(batchItems[], assetsForApprovals[], eulerVaultsForApprovals[]):
  -- Validates each batch item (deposit, withdraw, borrow, repay, controller ops)
  -- Sets max approvals for all asset/vault pairs
  -- EVC.batch(batchItems)
  -- Resets all approvals to 0
  => returns (batchSize, assets, vaults)
```

## Balance Tracking

```
balanceOf():
  for each substrate (EulerSubstrate):
    vaultAsset = ERC4626(vault).asset()
    (price, priceDecimals) = priceOracleMiddleware.getAssetPrice(vaultAsset)
    shares = vault.balanceOf(subAccountAddr)
    collateralAssets = vault.convertToAssets(shares)
    debtAssets = IBorrowing(vault).debtOf(subAccountAddr)
    balance += convertToWad(collateralAssets * price) - convertToWad(debtAssets * price)
  return balance  // USD in 18 decimals, must be non-negative
```

## Key Invariants
- All vault interactions go through EVC (Ethereum Vault Connector)
- Sub-account isolation: each (vault, subAccount) pair has independent permissions
- Exit on supply fuse has NO substrate validation (always allow withdrawal)
- Instant withdraw only allowed when isCollateral=false AND canBorrow=false
- Batch fuse validates every operation type against substrate permissions

## Risks & Edge Cases
- Sub-account XOR computation means sub-account 0x00 uses the PlasmaVault address directly
- Controller must be enabled before borrowing, collateral must be enabled before it counts
- Batch operations are atomic -- one failure reverts all
- Flash loans via batch are explicitly NOT supported (reverts with UnsupportedOperation)

## Related Files
- Source: `contracts/fuses/euler/`
- Tests: `test/fuses/euler/`
- External interfaces: `ext/IBorrowing.sol`, EVC from `ethereum-vault-connector`
