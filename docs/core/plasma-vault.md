# PlasmaVault

## Quick Reference
- Source: `contracts/vaults/PlasmaVault.sol` (~700 lines)
- Base: `contracts/vaults/PlasmaVaultBase.sol` (ERC20/Permit/Votes via delegatecall)
- Governance: `contracts/vaults/PlasmaVaultGovernance.sol` (~800 lines)
- Inherits: ERC4626Upgradeable, ReentrancyGuardUpgradeable, AccessManagedUpgradeable
- Proxy: UUPS upgradeable

## Public Interface

### User-facing (WHITELIST_ROLE or PUBLIC_ROLE)
```
deposit(uint256 assets, address receiver) → uint256 shares
  — Standard ERC4626. Validates cap. Charges deposit fee if configured.
  — Calls FeeManager.calculateDepositFee() if feeManager set.

mint(uint256 shares, address receiver) → uint256 assets
  — Mint exact shares, pull required assets.

withdraw(uint256 assets, address owner, address receiver) → uint256 shares
  — Two paths: (1) instant from unallocated balance, (2) from WithdrawManager request.
  — Instant: checks unallocated >= amount, charges withdraw fee.
  — Request: checks canWithdrawFromRequest, decreases sharesToRelease.

redeem(uint256 shares, address owner, address receiver) → uint256 assets
  — Same dual-path as withdraw, but burns exact shares.

depositWithPermit(uint256 assets, address receiver, ...) → uint256 shares
  — ERC2612 permit + deposit in one tx.

transfer/transferFrom — gated by TECH_VAULT_TRANSFER_SHARES_ROLE (disabled by default)
```

### Alpha-only (ALPHA_ROLE)
```
execute(FuseAction[] calldata actions_)
  — Batch-execute fuse actions via delegatecall.
  — For each action: validates fuse is registered, delegatecalls fuse with data.
  — Post-execution: calls _updateMarketsBalances() to recompute positions.
  — Checks asset distribution limits, reverts on violation.
  — NonReentrant.

executeWithContext(FuseAction[] calldata actions_, address contextTarget_, bytes calldata contextData_)
  — Like execute() but first calls contextTarget with contextData (for setting up transient storage etc.)
```

### Governance (ATOMIST_ROLE via PlasmaVaultGovernance)
```
addFuses(address[] fuses_) / removeFuses(address[] fuses_)
addBalanceFuse(uint256 marketId, address fuse) / removeBalanceFuse(uint256 marketId, address fuse)
grantMarketSubstrates(uint256 marketId, bytes32[] substrates)
configurePerformanceFee(address feeAccount, uint256 feeInPercentage)
configureManagementFee(address feeAccount, uint256 feeInPercentage)
setPriceOracleMiddleware(address oracle)
configureInstantWithdrawalFuses(InstantWithdrawalFusesParamsStruct[])
setTotalSupplyCap(uint256 cap)
updateMarketsBalances(uint256[] marketIds)
activateMarketsLimits(MarketLimit[] limits) / deactivateMarketsLimits(uint256[] marketIds)
setWithdrawManager(address) / setRewardsClaimManager(address) / setFeeManager(address)
convertToPublicVault() / enableTransferShares()
addPreHook(bytes4 selector, address hook) / removePreHook(bytes4 selector)
```

## State
- `totalAssets()` = liquid underlying balance + sum of all market positions (via balance fuses, in USD → converted)
- Share price = totalAssets() / totalSupply()
- Market balances updated via delegatecall to balance fuses on each execute()
- Asset distribution limits: per-market caps as percentage of totalAssets

## Key Storage
- Uses PlasmaVaultStorageLib (diamond-style slot-based storage)
- Market balances: `mapping(marketId => int256)` in WAD USD
- Fuse registry: FuseStorageLib (mapping + array)
- Balance fuses: `mapping(marketId => address)`
- Market substrates: `mapping(marketId => mapping(bytes32 => bool))`

## Related Files
- Libraries: PlasmaVaultLib.sol, PlasmaVaultStorageLib.sol, PlasmaVaultConfigLib.sol, FusesLib.sol
- Interfaces: IPlasmaVault.sol, IPlasmaVaultBase.sol, IPlasmaVaultGovernance.sol
- Errors: libraries/errors/Errors.sol
