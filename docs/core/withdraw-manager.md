# WithdrawManager

## Quick Reference
- Source: `contracts/managers/withdraw/WithdrawManager.sol`
- Proxy: Initializable, AccessManagedUpgradeable
- Two withdrawal paths: scheduled (request→release→withdraw) and instant (unallocated)

## Interface

### Public
```
requestShares(uint256 shares)
  — Create withdrawal request. Deducts request fee if configured.
  — Fee transferred to WithdrawManager via PlasmaVaultBase.transferRequestSharesFee().

requestInfo(address account) → WithdrawRequestInfo
  — Returns: { shares, endWithdrawWindowTimestamp, canWithdraw, withdrawWindowInSeconds }

getLastReleaseFundsTimestamp() → uint256
getSharesToRelease() → uint256
getWithdrawWindow() → uint256
getWithdrawFee() → uint256
getRequestFee() → uint256
```

### ALPHA_ROLE
```
releaseFunds(uint256 timestamp, uint256 sharesToRelease)
  — Sets release timestamp (must be in past) and amount of shares released.
  — Opens withdraw window for all requests made before this timestamp.
```

### ATOMIST_ROLE
```
updateWithdrawWindow(uint256 windowInSeconds) — Set withdraw window duration.
```

### Fee Roles
```
updateWithdrawFee(uint256 fee) — WITHDRAW_MANAGER_WITHDRAW_FEE_ROLE. Max 1e18 (100%).
updateRequestFee(uint256 fee) — WITHDRAW_MANAGER_REQUEST_FEE_ROLE. Max 1e18 (100%).
```

### TECH_PLASMA_VAULT_ROLE (called by PlasmaVault only)
```
canWithdrawFromRequest(address account, uint256 shares) → bool
  — Validates: within window, has enough shares, decreases counters.

canWithdrawFromUnallocated(uint256 shares) → uint256 feeSharesToBurn
  — Validates: unallocated balance >= shares + sharesToRelease.
  — Returns fee to burn (0 if no fee).
```

## Withdrawal Flow

### Scheduled Path
```
1. User calls requestShares(shares) → request stored with endWindowTimestamp
2. Alpha calls releaseFunds(timestamp, sharesToRelease) → opens window
3. User calls PlasmaVault.withdraw/redeem → vault calls canWithdrawFromRequest
4. If within window AND requestTimestamp < releaseFundsTimestamp → approved
5. Shares decreased, sharesToRelease decreased
```

### Instant Path (Unallocated)
```
1. User calls PlasmaVault.withdraw/redeem
2. Vault calls canWithdrawFromUnallocated(shares)
3. Check: vault's underlying balance - sharesToRelease >= shares
4. Apply withdraw fee if configured
5. Withdraw proceeds
```

## Key Invariants
- Pending requests NOT released by governance do NOT reduce unallocated balance
- Only sharesToRelease is reserved from unallocated
- Request fee deducted at request time (transferred to WithdrawManager)
- Withdraw fee deducted at withdrawal time (burned via BurnRequestFeeFuse)
- Both fees capped at 1e18 (100%), WAD precision

## Related Files
- WithdrawManagerStorageLib.sol — all storage operations
- BurnRequestFeeFuse.sol — burns fee shares
