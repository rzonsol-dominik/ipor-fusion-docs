# Vault Extensions

## Overview

Vault extensions wrap an underlying PlasmaVault with additional functionality -- fee management, access control, or referral tracking -- while presenting a standard ERC4626 interface to depositors. The wrapper holds shares of the underlying vault and issues its own shares to users.

## Architecture

```
User <-> WrappedPlasmaVault (ERC4626) <-> PlasmaVault (ERC4626) <-> DeFi protocols
```

The wrapper deposits user assets into the underlying PlasmaVault and mints its own shares. On withdrawal, it redeems from the underlying vault and returns assets to the user.

## WrappedPlasmaVaultBase (Abstract)

**Source:** `contracts/vaults/extensions/WrappedPlasmaVaultBase.sol`

Core ERC4626 wrapper with two fee mechanisms:

| Fee Type | Calculation | Trigger |
|----------|------------|---------|
| **Management** | `totalAssets * timeElapsed * feeRate / (365 days * 1e4)` | Time-based, accrues continuously |
| **Performance** | `(currentAssets - lastTotalAssets) * feeRate / 1e4` | Value increase since last operation |

Key design decisions:
- `totalAssets()` = `PLASMA_VAULT.maxWithdraw(this) - unrealizedManagementFee`
- Fee percentages use 2 decimal places (10000 = 100%)
- Decimals offset of 2 (`_decimalsOffset() = 2`) for share scaling
- Fees are realized (minted as shares to fee accounts) on every deposit/withdraw/mint/redeem
- `lastTotalAssets` tracks the high-water mark for performance fees
- All mutating operations are `nonReentrant`

## WrappedPlasmaVault

**Source:** `contracts/vaults/extensions/WrappedPlasmaVault.sol`

Extends `WrappedPlasmaVaultBase` + `Ownable2StepUpgradeable`. Fee configuration restricted to `onlyOwner`.

```solidity
configureManagementFee(feeAccount, feeInPercentage)  // onlyOwner
configurePerformanceFee(feeAccount, feeInPercentage)  // onlyOwner
```

## WhitelistWrappedPlasmaVault

**Source:** `contracts/vaults/extensions/WhitelistWrappedPlasmaVault.sol`

Extends `WrappedPlasmaVaultBase` + `WhitelistAccessControl`. All deposit/withdraw/mint/redeem operations require `WHITELISTED` role.

**Role hierarchy** (from `WhitelistAccessControl`):
```
DEFAULT_ADMIN_ROLE -> WHITELIST_MANAGER -> WHITELISTED
```
- `DEFAULT_ADMIN_ROLE`: Configures fees, manages WHITELIST_MANAGER role
- `WHITELIST_MANAGER`: Grants/revokes WHITELISTED role
- `WHITELISTED`: Can deposit/withdraw/mint/redeem

## ReferralPlasmaVault

**Source:** `contracts/vaults/extensions/ReferralPlasmaVault.sol`

Standalone contract (not a wrapped vault) that adds referral tracking to any ERC4626 vault.

```solidity
deposit(vault, assets, receiver, referralCode)  // anyone -- transfers assets, deposits, emits ReferralEvent
emitReferralForZapIn(referrer, referralCode)     // onlyZapIn
setZapInAddress(address)                          // onlyOwner, then renounces ownership permanently
```

The owner can set the zap-in address exactly once, after which ownership is permanently renounced.

## Key Invariants

- Wrapped vaults always hold underlying PlasmaVault shares, never raw protocol positions.
- Fee realization order: management fee first, then performance fee.
- Management fee timestamp only updates when fee shares are actually minted (prevents loss of small accruals).
- `ReferralPlasmaVault.setZapInAddress` is a one-shot operation -- ownership is renounced after the call.
- All ERC4626 wrapper operations validate non-zero amounts and non-zero receiver addresses.

## Related Files

- `contracts/vaults/extensions/` -- all extension contracts
- `contracts/libraries/PlasmaVaultLib.sol` -- fee configuration storage
- `contracts/libraries/PlasmaVaultStorageLib.sol` -- fee data structs
