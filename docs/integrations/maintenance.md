# Maintenance Utilities

## Quick Reference
- Fuse contracts: `ConfigureInstantWithdrawalFuse.sol`, `UpdateWithdrawManagerMaintenanceFuse.sol`
- No specific market ID (configurable per deployment)

## Overview

Maintenance fuses provide administrative configuration capabilities that can be executed via the PlasmaVault's fuse execution framework. They modify vault-level settings rather than interacting with external protocols.

## Fuse Interface

### ConfigureInstantWithdrawalFuse

```
ConfigureInstantWithdrawalFuse:
  constructor(marketId)

  enter(data: {fuses: InstantWithdrawalFusesParamsStruct[]}):
    -> Delegates to PlasmaVaultLib.configureInstantWithdrawalFuses()
    -> Validates each fuse is supported
    -> Configures instant withdrawal fuse parameters
    -> Array of (fuse address, params bytes32[]) pairs

  exit(bytes):
    -> Always reverts with ExitNotSupported()
```

This fuse configures which fuses and parameters are used when users trigger instant withdrawals. Each entry specifies a fuse address and its associated bytes32 parameter array.

### UpdateWithdrawManagerMaintenanceFuse

```
UpdateWithdrawManagerMaintenanceFuse:
  constructor(marketId)

  enter(data: {newManager: address}):
    1. If newManager is address(0), return (no-op)
    2. Write newManager to WITHDRAW_MANAGER storage slot
       via PlasmaVaultStorageLib.getWithdrawManager().manager
    3. Emit WithdrawManagerUpdated event

  exit(bytes):
    -> No-op (returns silently)

  getWithdrawManager() -> address:
    -> Reads current withdraw manager from storage
```

This fuse updates the withdraw manager address in the PlasmaVault. The withdraw manager controls withdrawal-related operations.

## Substrate Configuration

No substrate validation is performed by these fuses. Access control relies on the PlasmaVault's alpha/guardian role system that governs which fuses can be executed.

## Balance Tracking

Not applicable -- these are configuration-only fuses that do not manage assets.

## Key Invariants

- ConfigureInstantWithdrawalFuse does not support exit operations
- UpdateWithdrawManagerMaintenanceFuse silently skips zero-address updates
- Both fuses execute via delegatecall in PlasmaVault storage context
- The WITHDRAW_MANAGER storage slot was corrected in audit IL-6952 (R4H7) to avoid collision with CALLBACK_HANDLER
- These fuses have no storage variables of their own

## Related Files
- Source: `contracts/fuses/maintenance/ConfigureInstantWithdrawalFuse.sol`
- Source: `contracts/fuses/maintenance/UpdateWithdrawManagerMaintenanceFuse.sol`
