# Utility / Internal Fuses

## Quick Reference
- Source directory: `contracts/fuses/`
- These fuses are **not** protocol integrations -- they serve the vault's internal infrastructure
- Market IDs: configurable per deployment (cross-vault, validation, fee fuses) or `ZERO_BALANCE_MARKET` (update balances)

## Overview

Utility fuses are internal-use fuses that operate on PlasmaVault infrastructure rather than external DeFi protocols. They handle cross-vault share management, balance bookkeeping, asset validation, and fee burning. All execute via `delegatecall` from `PlasmaVault.execute()`, meaning `address(this)` is the PlasmaVault during execution.

---

## Cross-Vault Fuses

Manage share lifecycle between PlasmaVaults -- requesting withdrawals from a target vault and redeeming the resulting assets.

### PlasmaVaultRequestSharesFuse

Requests a scheduled withdrawal of shares from a target PlasmaVault.

```
PlasmaVaultRequestSharesFuse:
  constructor(marketId)

  enter(data: {sharesAmount: uint256, plasmaVault: address}):
    -> Returns early if sharesAmount == 0
    -> Validates plasmaVault is a granted substrate for MARKET_ID
    -> Clamps sharesAmount to vault's actual balance of target vault tokens
    -> Reads target vault's WithdrawManager via UniversalReader.read() + getWithdrawManager()
    -> Reverts if WithdrawManager is address(0)
    -> Calls WithdrawManager.requestShares(finalSharesAmount)
    -> Returns (plasmaVault, finalSharesAmount)
    -> Emits PlasmaVaultRequestSharesFuseEnter

  enterTransient():
    -> Reads inputs[0]=sharesAmount, inputs[1]=plasmaVault from transient storage
    -> Calls enter(), writes outputs to transient storage

  getWithdrawManager() -> address:
    -> Must be called via delegatecall (reverts if called directly on fuse)
    -> Reads WITHDRAW_MANAGER storage slot from PlasmaVaultStorageLib

  exit: not implemented
```

**Substrate requirement**: Target `plasmaVault` address must be granted as a substrate asset for this fuse's `MARKET_ID`.

### PlasmaVaultRedeemFromRequestFuse

Redeems assets from a previously submitted withdrawal request on a target PlasmaVault.

```
PlasmaVaultRedeemFromRequestFuse:
  constructor(marketId)

  enter(data: {sharesAmount: uint256, plasmaVault: address}):
    -> Returns early if sharesAmount == 0
    -> Validates plasmaVault is a granted substrate for MARKET_ID
    -> Clamps sharesAmount to vault's actual balance of target vault tokens
    -> Calls IPlasmaVault(plasmaVault).redeemFromRequest(finalSharesAmount, self, self)
    -> Returns (plasmaVault, finalSharesAmount)
    -> Emits PlasmaVaultRedeemFromRequestFuseEnter

  enterTransient():
    -> Reads inputs[0]=sharesAmount, inputs[1]=plasmaVault from transient storage
    -> Calls enter(), writes outputs to transient storage

  exit: not implemented
```

**Substrate requirement**: Same as PlasmaVaultRequestSharesFuse -- target vault must be a granted substrate.

**Typical workflow**: `PlasmaVaultRequestSharesFuse.enter()` (initiate withdrawal) -> wait for withdrawal window -> `PlasmaVaultRedeemFromRequestFuse.enter()` (claim assets).

---

## Balance Fuses

### ZeroBalanceFuse

A no-op balance fuse that always returns zero. Used when a market needs a registered balance fuse but has no meaningful position value to track (e.g., utility/configuration-only markets).

```
ZeroBalanceFuse:
  constructor(marketId)

  balanceOf() -> uint256:
    -> Always returns 0
```

Implements `IMarketBalanceFuse`. Does not implement `enter()`/`exit()` -- it is a pure balance reporter.

### UpdateMarketsBalancesFuse

Triggers on-demand balance recalculation for one or more markets. Useful when external price changes or position updates require the vault's internal accounting to be refreshed outside the normal execution flow.

```
UpdateMarketsBalancesFuse:
  constructor()  // no parameters

  MARKET_ID = IporFusionMarkets.ZERO_BALANCE_MARKET  (type(uint256).max)

  enter(data: {marketIds: uint256[]}):
    -> Reverts if marketIds array is empty
    -> Reads vault's asset address and decimals via IERC4626/IERC20Metadata (delegatecall context)
    -> Calls PlasmaVaultMarketsLib.updateMarketsBalances(marketIds, asset, decimals, DECIMALS_OFFSET)
    -> Emits UpdateMarketsBalancesEnter

  exit(bytes):
    -> Always reverts with UpdateMarketsBalancesFuseExitNotSupported
```

**Note**: Uses `ZERO_BALANCE_MARKET` (`type(uint256).max`) as its own market ID since it is not tied to any specific market -- it updates balances for arbitrary markets passed as input.

### PlasmaVaultBalanceAssetsValidationFuse

An assertion fuse that validates the vault's ERC20 token balances fall within specified min/max ranges. Typically used as a guard step within a multi-fuse execution batch to enforce invariants.

```
PlasmaVaultBalanceAssetsValidationFuse:
  constructor(marketId)  // reverts if marketId == 0

  enter(data: {assets: address[], minBalanceValues: uint256[], maxBalanceValues: uint256[]}):
    -> Reverts if array lengths do not match
    -> For each asset:
       - Skips address(0) entries
       - Validates asset is a granted substrate for MARKET_ID
       - Reads ERC20.balanceOf(address(this)) (vault's balance in delegatecall context)
       - Reverts with PlasmaVaultBalanceAssetsValidationFuseInvalidBalance
         if balance < min or balance > max

  exit: not implemented
```

**Substrate requirement**: Each validated asset must be granted as a substrate for this fuse's `MARKET_ID`. Balance values use the asset's native decimals (not WAD-normalized).

---

## Fee Fuses

### BurnRequestFeeFuse

Burns request fee shares that were collected by the WithdrawManager. Reduces total supply and maintains governance voting checkpoint consistency by routing the burn through the vault's `_update` pipeline.

```
BurnRequestFeeFuse:
  constructor(marketId)

  enter(data: {amount: uint256}):
    -> Reads WithdrawManager address from WITHDRAW_MANAGER storage slot
    -> Reverts with BurnRequestFeeWithdrawManagerNotSet if not configured
    -> Returns early if amount == 0
    -> Calls PlasmaVaultBase.updateInternal(withdrawManager, address(0), amount)
       via nested delegatecall to burn shares from withdrawManager to zero address
    -> Emits BurnRequestFeeEnter

  enterTransient():
    -> Reads inputs[0]=amount from transient storage
    -> Calls enter()

  exit():
    -> Always reverts with BurnRequestFeeExitNotImplemented

  exitTransient():
    -> Always reverts with BurnRequestFeeExitNotImplemented
```

**Burn routing**: The burn deliberately uses `PlasmaVaultBase.updateInternal` (nested delegatecall) rather than a direct token burn. This ensures `_transferVotingUnits` and other ERC20Votes hooks execute, keeping governance state consistent.

---

## Key Invariants

- All utility fuses execute via `delegatecall` from PlasmaVault -- they have no storage of their own
- Cross-vault fuses clamp share amounts to actual token balances, preventing over-request/over-redemption
- Cross-vault fuses require target vault addresses to be granted as substrates (access control via `PlasmaVaultConfigLib.isSubstrateAsAssetGranted`)
- `UpdateMarketsBalancesFuse` uses `ZERO_BALANCE_MARKET` as its market ID since it operates across arbitrary markets
- `PlasmaVaultBalanceAssetsValidationFuse` is a pure assertion -- it reverts on failure rather than modifying state
- `BurnRequestFeeFuse` routes burns through the vault's `_update` pipeline to preserve voting checkpoints
- The `WITHDRAW_MANAGER` storage slot was corrected in audit IL-6952 (R4H7) to avoid collision with `CALLBACK_HANDLER` -- fuses reading this slot (`PlasmaVaultRequestSharesFuse`, `BurnRequestFeeFuse`) must use the corrected `PlasmaVaultStorageLib`
- `ZeroBalanceFuse` constructor requires only `marketId`; `PlasmaVaultBalanceAssetsValidationFuse` rejects `marketId == 0`

## Related Files

- Source: `contracts/fuses/plasma_vault/PlasmaVaultRequestSharesFuse.sol`
- Source: `contracts/fuses/plasma_vault/PlasmaVaultRedeemFromRequestFuse.sol`
- Source: `contracts/fuses/update_balances/UpdateMarketsBalancesFuse.sol`
- Source: `contracts/fuses/update_balances/IUpdateMarketsBalancesFuse.sol`
- Source: `contracts/fuses/ZeroBalanceFuse.sol`
- Source: `contracts/fuses/PlasmaVaultBalanceAssetsValidationFuse.sol`
- Source: `contracts/fuses/burn_request_fee/BurnRequestFeeFuse.sol`
- Interface: `contracts/fuses/IMarketBalanceFuse.sol`
- Interface: `contracts/fuses/IFuseCommon.sol`
- Constants: `contracts/libraries/IporFusionMarkets.sol` (`ZERO_BALANCE_MARKET`)
- Storage: `contracts/libraries/PlasmaVaultStorageLib.sol` (WITHDRAW_MANAGER slot)
- Related doc: `docs/integrations/maintenance.md` (ConfigureInstantWithdrawalFuse, UpdateWithdrawManagerMaintenanceFuse)
