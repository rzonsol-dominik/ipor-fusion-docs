# FeeManager

## Quick Reference
- Source: `contracts/managers/fee/FeeManager.sol`
- Dependencies: FeeAccount.sol (2 instances: performance + management)
- Fee precision: 2-decimal percentage (10000 = 100%, 100 = 1%) for perf/mgmt
- Deposit fee precision: WAD (1e18 = 100%, 1e16 = 1%)

## Fee Types

### Performance Fee (High Water Mark)
- Only charged on gains above HWM
- HWM = highest exchange rate (assets per 1 share unit)
- `calculateAndUpdatePerformanceFee(uint128 exchangeRate, totalSupply, fee, assetDecimals)`
- Note: `assetDecimals` is unused (retained for ABI backward compatibility)
- If rate > HWM: `feeShares = Math.mulDiv(totalSupply * (rate - HWM), fee, rate * 10000)`
- If rate <= HWM and updateInterval elapsed: HWM lowered to current rate
- If rate <= HWM and updateInterval = 0: HWM stays (manual update only)
- Minted shares go to PERFORMANCE_FEE_ACCOUNT

### Management Fee
- Time-based, proportional to totalAssets
- Configured via `PlasmaVaultGovernance.configureManagementFee(feeAccount, totalFee)`
- Minted shares go to MANAGEMENT_FEE_ACCOUNT

### Deposit Fee
- Charged on deposits, deducted from shares
- WAD precision (1e18 = 100%)
- `calculateDepositFee(shares) → feeShares`

## Fee Distribution
```
harvestAllFees() — public, calls both harvest functions
harvestManagementFee() — public
harvestPerformanceFee() — public

Distribution order:
1. Calculate DAO portion: daoFee * balance / totalFee
2. Transfer DAO portion to iporDaoFeeRecipientAddress
3. For each recipient: recipientFee * balance / totalFee → transfer
```

## Immutable (set at construction)
- PLASMA_VAULT — vault address
- PERFORMANCE_FEE_ACCOUNT — FeeAccount contract for perf fees
- MANAGEMENT_FEE_ACCOUNT — FeeAccount contract for mgmt fees
- IPOR_DAO_MANAGEMENT_FEE — DAO management fee %
- IPOR_DAO_PERFORMANCE_FEE — DAO performance fee %

## Updatable (ATOMIST_ROLE)
```
updateManagementFee(RecipientFee[]) — harvests first, then replaces all recipients
updatePerformanceFee(RecipientFee[]) — harvests first, then replaces all recipients
setDepositFee(uint256 fee) — WAD precision
updateHighWaterMarkPerformanceFee() — reset HWM to current exchange rate
updateIntervalHighWaterMarkPerformanceFee(uint32 interval) — auto-lower HWM interval
```

## Updatable (IPOR_DAO_ROLE)
```
setIporDaoFeeRecipientAddress(address) — DAO fee recipient
```

## View Functions
```
getManagementFeeRecipients() → RecipientFee[]
getPerformanceFeeRecipients() → RecipientFee[]
getTotalManagementFee() → uint256
getTotalPerformanceFee() → uint256
getIporDaoFeeRecipientAddress() → address
getDepositFee() → uint256
getPlasmaVaultHighWaterMarkPerformanceFee() → HighWaterMarkPerformanceFeeStorage
```

## Related Files
- FeeAccount.sol — holds fee shares, approves transfers
- FeeManagerFactory.sol — RecipientFee struct
- FeeManagerStorageLib.sol — all storage
