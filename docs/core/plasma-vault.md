# PlasmaVault

## Quick Reference
- Source: `contracts/vaults/PlasmaVault.sol` (~1600 lines)
- Base: `contracts/vaults/PlasmaVaultBase.sol` (ERC20/Permit/Votes via delegatecall)
- Governance: `contracts/vaults/PlasmaVaultGovernance.sol` (~1700 lines)
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

redeemFromRequest(uint256 shares, address owner, address receiver) → uint256 assets
  — Redeem shares from a scheduled withdrawal request only (no instant path).
  — Validates via WithdrawManager.canWithdrawFromRequest, charges withdraw fee.

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

```

### Governance (PlasmaVaultGovernance)
```
— FUSE_MANAGER_ROLE (300):
addFuses(address[] fuses_) / removeFuses(address[] fuses_)
addBalanceFuse(uint256 marketId, address fuse) / removeBalanceFuse(uint256 marketId, address fuse)

— ATOMIST_ROLE (100):
grantMarketSubstrates(uint256 marketId, bytes32[] substrates)
configurePerformanceFee(address feeAccount, uint256 feeInPercentage)
configureManagementFee(address feeAccount, uint256 feeInPercentage)
setPriceOracleMiddleware(address oracle)
setTotalSupplyCap(uint256 cap)
setupMarketsLimits(MarketLimit[] marketsLimits)
activateMarketsLimits() / deactivateMarketsLimits()   — no parameters
setRewardsClaimManagerAddress(address)
convertToPublicVault() / enableTransferShares()
updateDependencyBalanceGraphs(uint256[] marketIds, uint256[][] dependencies)
updateCallbackHandler(address handler, address sender, bytes4 sig)

— CONFIG_INSTANT_WITHDRAWAL_FUSES_ROLE (900):
configureInstantWithdrawalFuses(InstantWithdrawalFusesParamsStruct[])

— UPDATE_MARKETS_BALANCES_ROLE (1000):
updateMarketsBalances(uint256[] marketIds)

— PRE_HOOKS_MANAGER_ROLE (301):
setPreHookImplementations(bytes4[] selectors, address[] implementations, bytes32[][] substrates)

— View functions (no role):
getMarketSubstrates(marketId), getFuses(), getPriceOracleMiddleware(),
getPerformanceFeeData(), getManagementFeeData(), getAccessManagerAddress(),
getRewardsClaimManagerAddress(), getInstantWithdrawalFuses(), getMarketLimit(marketId),
getDependencyBalanceGraph(marketId), getTotalSupplyCap(), isMarketsLimitsActivated(),
isFuseSupported(fuse), isBalanceFuseSupported(marketId, fuse), isMarketSubstrateGranted(marketId, substrate)
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
