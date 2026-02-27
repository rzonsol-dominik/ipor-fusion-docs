# Factory System

## Overview

The Factory System is the deployment infrastructure for IPOR Fusion vaults. `FusionFactory` is a UUPS-upgradeable orchestrator that coordinates six component factories to atomically deploy a complete vault instance (PlasmaVault + AccessManager + WithdrawManager + RewardsManager + PriceManager + ContextManager + FeeManager) in a single transaction using the EIP-1167 minimal proxy (Clones) pattern. Separate extension factories and price feed factories exist for wrapped vaults and oracle integrations.

## FusionFactory

The main entry point. Inherits `UUPSUpgradeable` and `FusionFactoryAccessControl`. Stores all configuration in ERC-7201 namespaced storage via `FusionFactoryStorageLib`.

**Key functions:**

```
clone(assetName, assetSymbol, underlyingToken, redemptionDelayInSeconds, owner, daoFeePackageIndex)
    -> FusionInstance  // permissionless vault creation

cloneSupervised(...)
    -> FusionInstance  // same as clone, restricted to MAINTENANCE_MANAGER_ROLE
                       // passes withAdmin=true, granting PlasmaVaultAdmin roles

updateFactoryAddresses(version, newFactoryAddresses)   // MAINTENANCE_MANAGER_ROLE
updateBaseAddresses(version, newBases...)               // MAINTENANCE_MANAGER_ROLE
updatePlasmaVaultBase(newBase)                          // MAINTENANCE_MANAGER_ROLE
updatePriceOracleMiddleware(newMiddleware)               // MAINTENANCE_MANAGER_ROLE
updateWithdrawWindowInSeconds(seconds)                   // MAINTENANCE_MANAGER_ROLE
updateVestingPeriodInSeconds(seconds)                    // MAINTENANCE_MANAGER_ROLE
setDaoFeePackages(FeePackage[])                          // DAO_FEE_MANAGER_ROLE
updatePlasmaVaultAdminArray(addresses)                   // DEFAULT_ADMIN_ROLE
```

**Stored configuration:** factory version, factory index (auto-incremented per vault), factory addresses (7 component factories), base addresses (6 implementation contracts), PlasmaVault base, price oracle middleware, burn request fee fuse + balance fuse, withdraw window (default 24h), vesting period (default 1 week), DAO fee packages, admin array.

## Component Factories

Each component factory exposes both `create(...)` (fresh deploy) and `clone(baseAddress, ...)` (EIP-1167 minimal proxy + `proxyInitialize`). The clone path is used by `FusionFactory` for gas efficiency.

### PlasmaVaultFactory
Clones a `PlasmaVault` base implementation and calls `proxyInitialize(PlasmaVaultInitData)`. The init data includes asset name/symbol, underlying token, price oracle, fee config, access manager, withdraw manager, and the PlasmaVault base (delegate) address.

### AccessManagerFactory
Deploys or clones `IporFusionAccessManager`. Initialized with an `initialAuthority` (the FusionFactory itself during creation) and `redemptionDelayInSeconds`. The factory temporarily holds authority, then transfers it during final configuration.

### WithdrawManagerFactory
Clones `WithdrawManager`, initialized with an `accessManager` address. The withdraw window and plasma vault address are configured post-clone by the orchestrator.

### RewardsManagerFactory
Clones `RewardsClaimManager`, initialized with `accessManager` and `plasmaVault` addresses. Vesting time is configured post-clone.

### PriceManagerFactory
Clones `PriceOracleMiddlewareManager`, initialized with `accessManager` and `priceOracleMiddleware` addresses. Acts as the per-vault price oracle proxy.

### ContextManagerFactory
Clones `ContextManager`, initialized with `accessManager` and an array of `approvedTargets` (PlasmaVault, WithdrawManager, PriceManager, RewardsManager, FeeManager). Created last because it needs all other component addresses.

## Deployment Flow

When `clone()` or `cloneSupervised()` is called:

1. **Validate inputs** -- underlying token and owner must be non-zero.
2. **Increment factory index** -- each vault gets a unique sequential index.
3. **Validate base addresses** -- all 6 base implementations must be non-zero.
4. **Validate DAO fee package** -- selected package index must exist and be valid.
5. **Clone AccessManager** -- initial authority set to FusionFactory itself.
6. **Clone PriceManager** -- linked to the new AccessManager and global price oracle middleware.
7. **Clone WithdrawManager** -- linked to the new AccessManager.
8. **Clone PlasmaVault** -- receives fee config (from DAO fee package), underlying token, AccessManager, PriceManager, WithdrawManager, and PlasmaVault base (delegate).
9. **Clone RewardsManager** -- linked to AccessManager and the new PlasmaVault.
10. **Retrieve FeeManager address** -- extracted from the PlasmaVault's performance fee data (created during PlasmaVault init).
11. **Clone ContextManager** -- approved targets = [PlasmaVault, WithdrawManager, PriceManager, RewardsManager, FeeManager].
12. **Post-clone configuration:**
    - Set rewards vesting period on RewardsManager.
    - Set rewards claim manager address on PlasmaVault.
    - Set withdraw window and plasma vault address on WithdrawManager.
    - Add BurnRequestFeeFuse to PlasmaVault.
    - Add BurnRequestFeeBalanceFuse to PlasmaVault's zero-balance market.
    - Initialize FeeManager.
13. **Initialize AccessManager** -- configure roles: set owner, optionally set admin array (if supervised), set DAO fee recipient as IPOR DAO. FusionFactory relinquishes authority.
14. **Emit `FusionInstanceCreated` event** with all addresses and metadata.
15. **Return `FusionInstance` struct** containing all deployed addresses.

## Factory Extensions

### WrappedPlasmaVaultFactory
UUPS-upgradeable, `Ownable2Step`. Creates `WrappedPlasmaVault` instances that wrap an existing PlasmaVault with additional fee layers. Permissionless `create()` function.

```
create(name, symbol, plasmaVault, owner,
       managementFeeAccount, managementFeePercentage,    // basis points, max 10000
       performanceFeeAccount, performanceFeePercentage)
    -> address wrappedPlasmaVault
```

### WhitelistWrappedPlasmaVaultFactory
Same pattern as `WrappedPlasmaVaultFactory` but creates `WhitelistWrappedPlasmaVault` instances that add access-controlled (whitelist-gated) deposits on top of a PlasmaVault. Takes an `initialAdmin` who manages the whitelist instead of a simple `owner`.

## Price Feed Factories

All price feed factories are UUPS-upgradeable with `Ownable2Step` access control. They deploy standalone Chainlink-compatible price feed contracts (`latestRoundData` interface).

| Factory | Deploys | Purpose |
|---------|---------|---------|
| `ERC4626PriceFeedFactory` | `ERC4626PriceFeed` | USD price for any ERC-4626 vault share via `convertToAssets` + underlying price. Validates vault decimals, share price, and underlying asset price on creation. |
| `PtPriceFeedFactory` | `PtPriceFeed` | USD price for Pendle Principal Tokens. Uses Pendle TWAP oracle (`getPtToSyRate` or `getPtToAssetRate`). Validates price within 1% of expected value. Includes a `calculatePrice()` view for previewing. |
| `BeefyVaultV7PriceFeedFactory` | `BeefyVaultV7PriceFeed` | USD price for Beefy Vault V7 LP tokens. Validates non-zero price on creation. |
| `CurveStableSwapNGPriceFeedFactory` | `CurveStableSwapNGPriceFeed` | USD price for Curve StableSwap NG LP tokens. Validates non-zero price on creation. |
| `DualCrossReferencePriceFeedFactory` | `DualCrossReferencePriceFeed` | USD price derived by cross-referencing two oracle feeds (assetX/assetY and assetY/USD). |
| `CollateralTokenOnMorphoMarketPriceFeedFactory` | `CollateralTokenOnMorphoMarketPriceFeed` | USD price for Morpho market collateral tokens. Tracks created feeds in a mapping keyed by `(creator, morphoOracle, collateralToken, loanToken, middleware)`. Prevents duplicate creation. |

## Access Control

`FusionFactoryAccessControl` extends OpenZeppelin `AccessControlEnumerableUpgradeable` and defines three roles:

| Role | Hash Prefix | Protects |
|------|-------------|----------|
| `DEFAULT_ADMIN_ROLE` | `0x00` | `updatePlasmaVaultAdminArray`, UUPS upgrade authorization, role management |
| `MAINTENANCE_MANAGER_ROLE` | `0xc927...` | All `update*` functions: factory addresses, base addresses, PlasmaVault base, price oracle, fee fuses, withdraw window, vesting period. Also `cloneSupervised`. |
| `DAO_FEE_MANAGER_ROLE` | `0x12ca...` | `setDaoFeePackages` |
| `PAUSE_MANAGER_ROLE` | `0x356a...` | pause/unpause (declared but not yet implemented in factory) |

Extension and price feed factories use `Ownable2Step` (two-step ownership transfer) instead of role-based access.

## Key Invariants

- All base addresses and factory addresses must be non-zero before cloning.
- The factory index is strictly monotonically increasing (never reused).
- DAO fee packages: management and performance fees capped at 10000 (100%), fee recipient cannot be zero address.
- FusionFactory holds temporary authority over the AccessManager only during deployment; authority is transferred to the configured owner/admin in the final initialization step.
- Price feed factories validate that newly created feeds return valid (non-zero/positive) prices before completing deployment (except `DualCrossReferencePriceFeedFactory`).
- `CollateralTokenOnMorphoMarketPriceFeedFactory` enforces uniqueness per `(creator, oracle, collateral, loan, middleware)` tuple.
- Storage uses ERC-7201 namespaced pattern to prevent slot collisions across upgrades.

## Related Files

- `contracts/factory/FusionFactory.sol` -- main orchestrator
- `contracts/factory/FusionFactoryAccessControl.sol` -- role definitions
- `contracts/factory/PlasmaVaultFactory.sol` -- vault cloning
- `contracts/factory/AccessManagerFactory.sol` -- access manager creation
- `contracts/factory/WithdrawManagerFactory.sol` -- withdraw manager creation
- `contracts/factory/RewardsManagerFactory.sol` -- rewards manager creation
- `contracts/factory/PriceManagerFactory.sol` -- price manager creation
- `contracts/factory/ContextManagerFactory.sol` -- context manager creation
- `contracts/factory/lib/FusionFactoryLib.sol` -- initialization and clone orchestration
- `contracts/factory/lib/FusionFactoryLogicLib.sol` -- clone sequencing and final configuration
- `contracts/factory/lib/FusionFactoryStorageLib.sol` -- ERC-7201 storage layout
- `contracts/factory/extensions/WrappedPlasmaVaultFactory.sol` -- wrapped vault factory
- `contracts/factory/extensions/WhitelistWrappedPlasmaVaultFactory.sol` -- whitelist wrapped vault factory
- `contracts/factory/price_feed/ERC4626PriceFeedFactory.sol`
- `contracts/factory/price_feed/PtPriceFeedFactory.sol`
- `contracts/factory/price_feed/BeefyVaultV7PriceFeedFactory.sol`
- `contracts/factory/price_feed/CurveStableSwapNGPriceFeedFactory.sol`
- `contracts/factory/price_feed/DualCrossReferencePriceFeedFactory.sol`
- `contracts/factory/price_feed/CollateralTokenOnMorphoMarketPriceFeedFactory.sol`
