# Internal Libraries & Helpers

## Overview
Internal libraries and helper contracts used across the IPOR Fusion codebase. These are not deployed standalone -- they are either `library` contracts linked via `internal` calls or `abstract` contracts inherited by core system contracts. None have external entry points.

---

## Vault Libraries

### AssetDistributionProtectionLib
- Source: `contracts/libraries/AssetDistributionProtectionLib.sol`
- Purpose: Risk management system enforcing per-market exposure limits as a percentage of total vault value.
- Used by: PlasmaVault (operation validation), PlasmaVaultGovernance (configuration)

**Key Functions**
```
activateMarketsLimits() — Enables limit enforcement (sets sentinel in storage slot 0). ATOMIST_ROLE via governance.
deactivateMarketsLimits() — Disables all limit checks. Emergency control.
setupMarketsLimits(MarketLimit[] calldata) — Sets max allocation % per market. 1e18 = 100%. Reverts on marketId=0 or limit>100%.
checkLimits(DataToCheck memory) — Validates all market positions against limits. Called during vault operations. No-op if deactivated.
isMarketsLimitsActivated() → bool — Returns true if limits are enforced.
```

**Data Structures**
- `MarketLimit { uint256 marketId; uint256 limitInPercentage }` -- config input
- `MarketToCheck { uint256 marketId; uint256 balanceInMarket }` -- runtime balance check
- `DataToCheck { uint256 totalBalanceInVault; MarketToCheck[] marketsToCheck }` -- aggregated state

**Storage**: Uses `PlasmaVaultStorageLib.getMarketsLimits()`. Slot 0 is reserved for activation flag.

### PlasmaVaultFeesLib
- Source: `contracts/vaults/lib/PlasmaVaultFeesLib.sol`
- Purpose: Fee calculation helpers for performance, management, and deposit fees.
- Used by: PlasmaVault (fee realization during deposits, withdrawals, balance updates)

**Key Functions**
```
prepareForAddPerformanceFee(totalSupply, decimals, decimalsOffset, actualExchangeRate)
  → (address recipient, uint256 feeShares)
  — Delegates to FeeManager for HWM-based performance fee calculation.

prepareForRealizeDepositFee(mintedShares)
  → (address recipient, uint256 feeShares)
  — Calculates deposit fee from minted shares. Returns (0,0) if no feeAccount configured.

prepareForRealizeManagementFee(totalAssetsBefore)
  → (address recipient, uint256 unrealizedFeeInUnderlying)
  — Time-weighted management fee. Caller MUST call PlasmaVaultLib.updateManagementFeeData() only after
    successfully minting fee shares (IL-6959 fix).

getUnrealizedManagementFee(totalAssets) → uint256
  — Pure calculation: totalAssets * elapsed * feePercentage / (365 days * 1e4).
```

**Constants**: `FEE_PERCENTAGE_DECIMALS_MULTIPLIER = 1e4` (10000 = 100%, 2 decimal places).

### PlasmaVaultMarketsLib
- Source: `contracts/vaults/lib/PlasmaVaultMarketsLib.sol`
- Purpose: Market balance tracking and instant withdrawal orchestration.
- Used by: PlasmaVault (balance updates after fuse execution, withdrawal flow)

**Key Functions**
```
updateMarketsBalances(uint256[] markets, address asset, uint256 decimals, uint256 decimalsOffset)
  → DataToCheck
  — Fetches balance from each market's balance fuse via delegatecall, converts USD WAD to
    underlying decimals using price oracle, updates per-market tracking via
    PlasmaVaultLib.addToTotalAssetsInAllMarkets(). Resolves balance fuse dependency graphs
    (transitive). Note: returned DataToCheck.totalBalanceInVault is NOT populated (stays 0);
    per-market balances are in marketsToCheck[].

withdrawFromMarkets(address asset, uint256 assets, uint256 vaultCurrentBalanceUnderlying)
  → uint256[] markets
  — Iterates instant withdrawal fuses in order, delegatecalling instantWithdraw(bytes32[])
    until the required amount is covered. Returns unique market IDs touched.
```

**Internal helpers**: `_checkBalanceFusesDependencies` (transitive dependency resolution), `_filterZeroMarkets`, `_getUniqueElements`, `_checkIfExistsMarket`, `_increaseArray`, `_contains`.

### ERC20VotesUpgradeable
- Source: `contracts/vaults/ERC20VotesUpgradeable.sol`
- Purpose: Modified OpenZeppelin ERC20Votes for PlasmaVault governance voting.
- Used by: PlasmaVaultBase (inheritance)

**Key Difference from OZ**: The `_update` function is removed. PlasmaVaultBase provides its own `_update` that combines ERC20Votes checkpoint logic with total supply cap validation.

**Functions**
```
_getVotingUnits(address) → uint256 — Returns balanceOf(account). 1:1 token-to-vote mapping.
numCheckpoints(address) → uint32 — Number of checkpoints for account.
checkpoints(address, uint32 pos) → Checkpoint208 — Get specific checkpoint.
```

---

## Type & Conversion Libraries

### TypeConversionLib
- Source: `contracts/libraries/TypeConversionLib.sol`
- Purpose: Safe bidirectional conversion between `bytes32` and Solidity primitives.
- Used by: Fuse contracts, PlasmaVault configuration, substrate storage

**Conversions**
```
toBytes32(address) / toAddress(bytes32)
toBytes32(uint256) / toUint256(bytes32)
toBytes32(bool) / toBool(bytes32)
toBytes32(int256) / toInt256(bytes32)
toBytes32(bytes memory) / toBytes(bytes32)
```

**Note**: `toBytes32(bytes)` uses assembly for efficiency. Input shorter than 32 bytes is zero-padded right. Empty input returns `0x0`.

**Enum**: `DataType` -- 16 members for runtime type discrimination: UNKNOWN, UINT256, UINT128, UINT64, UINT32, UINT16, UINT8, INT256, INT128, INT64, INT32, INT16, INT8, ADDRESS, BOOL, BYTES32.

### StringConverter
- Source: `contracts/libraries/StringConverter.sol`
- Purpose: Encode/decode strings to/from `bytes32[]` arrays using length-prefixed encoding.
- Used by: Fuse metadata storage, deploy initialization

**Functions**
```
toBytes32(string memory) → bytes32[] — Length-prefixed encoding: first 2 bytes = uint16 string length
  (little-endian), remaining bytes = string data. Max length: 65535. Lossless round-trip.

fromBytes32(bytes32[] memory) → string — Decodes length-prefixed bytes32 array back to string.
  Reverts with InvalidEncodedData if declared length exceeds available bytes.
```

**Security**: IL-6957/IL-6960 fix. Length-prefix prevents collision attacks from trailing null bytes and decoding malleability.

---

## Access & Initialization Libraries

### IporFusionAccessManagerInitializerLibV1
- Source: `contracts/vaults/initializers/IporFusionAccessManagerInitializerLibV1.sol`
- Purpose: Generates complete `InitializationData` for IporFusionAccessManager during vault deployment.
- Used by: FusionFactory, deployment scripts

**Key Function**
```
generateInitializeIporPlasmaVault(DataForInitialization memory)
  → InitializationData
  — Produces three arrays:
    1. roleToFunctions — maps each role to allowed function selectors on specific targets
    2. adminRoles — sets which role administers each other role
    3. accountToRoles — assigns addresses to roles (DAO, admin, owner, atomist, alpha, etc.)
  — Validates all addresses are non-zero. Reverts with InvalidAddress on failure.
```

**Data Structures**
- `PlasmaVaultAddress` -- addresses of all vault system contracts (vault, accessManager, rewardsClaimManager, withdrawManager, feeManager, contextManager, priceOracleMiddlewareManager)
- `DataForInitialization` -- complete init input: isPublic flag, all role-holder address arrays, PlasmaVaultAddress
- `Iterator` -- simple counter for array building

**Constants**: Pre-calculated array lengths for role mappings (ADMIN_ROLES_ARRAY_LENGTH=20, ROLES_TO_FUNCTION_INITIAL_ARRAY_LENGTH=40, etc.)

### IporFusionAccessManagersStorageLib
- Source: `contracts/managers/access/IporFusionAccessManagersStorageLib.sol`
- Purpose: ERC-7201 namespaced storage for access manager state (redemption locks, execution delays, initialization flag).
- Used by: IporFusionAccessManager, RedemptionDelayLib

**Storage Slots (ERC-7201)**
| Slot Constant | Namespace | Purpose |
|--------------|-----------|---------|
| `REDEMPTION_LOCKS` | `io.ipor.managers.access.RedemptionLocks` | Per-account unlock timestamps |
| `MINIMAL_EXECUTION_DELAY_FOR_ROLE` | `io.ipor.managers.access.MinimalExecutionDelayForRole` | Role-specific execution delays |
| `INITIALIZATION_FLAG` | `io.ipor.managers.access.InitializationFlag` | One-time init guard |

**Key Functions**
```
getInitializationFlag() → InitializationFlag storage
getMinimalExecutionDelayForRole() → MinimalExecutionDelayForRole storage
getRedemptionLocks() → RedemptionLocks storage
setRedemptionLocks(address account) — Sets lock = block.timestamp + REDEMPTION_DELAY_IN_SECONDS.
  No-op if delay is 0. Emits RedemptionDelayForAccountUpdated.
```

### IporFusionAccessControl
- Source: `contracts/price_oracle/IporFusionAccessControl.sol`
- Purpose: Role-based access control base for price oracle contracts. Extends OZ AccessControlEnumerableUpgradeable.
- Used by: PriceOracleMiddlewareWithRoles

**Roles Defined**
```
ADD_PT_TOKEN_PRICE = keccak256("ADD_PT_TOKEN_PRICE")
  — Allows adding Pendle PT token price feeds.

SET_ASSETS_PRICES_SOURCES = keccak256("SET_ASSETS_PRICES_SOURCES")
  — Allows setting asset price feed sources.
```

**Pattern**: Constructor calls `_disableInitializers()`. Must be initialized through proxy via `__IporFusionAccessControl_init()` which calls `__AccessControlEnumerable_init()`. Note: this init does NOT grant the deployer any roles — the concrete inheriting contract must grant DEFAULT_ADMIN_ROLE separately.

---

## Price Manager Library

### PriceOracleMiddlewareManagerLib
- Source: `contracts/managers/price/PriceOracleMiddlewareManagerLib.sol`
- Purpose: Storage and logic for managing asset price sources and price change validation within the PriceOracleMiddlewareManager.
- Used by: PriceOracleMiddlewareManager

**Storage Slots (ERC-7201)**
| Slot Constant | Namespace | Purpose |
|--------------|-----------|---------|
| `ASSETS_PRICES_SOURCES` | `io.ipor.priceOracleManager.AssetsPricesSources` | Asset-to-price-feed mapping |
| `PRICES_ORACLE_MIDDLEWARE` | `io.ipor.priceOracle.PriceOracleMiddleware` | Active oracle middleware address |
| `PRICE_VALIDATION_SLOT` | `io.ipor.priceOracleManager.PriceValidation` | Per-asset price delta validation |

**Key Functions**
```
addAssetPriceSource(address asset, address source) — Maps asset to price feed. Tracks in assets array.
removeAssetPriceSource(address asset) — Removes mapping. Swap-and-pop on assets array.
getSourceOfAssetPrice(address asset) → address
getConfiguredAssets() → address[]

setPriceOracleMiddleware(address) — Sets active oracle. Reverts on zero address.
getPriceOracleMiddleware() → address

updatePriceValidation(address asset, uint256 maxPriceDelta) — Configures max allowed price change (1e18=100%).
removePriceValidation(address asset) — Clears validation config and stored baseline.
validatePriceChange(address asset, uint256 price) → bool baselineUpdated
  — First call: stores baseline, returns true.
  — Subsequent: reverts PriceChangeExceeded if delta > maxPriceDelta.
  — Updates baseline when delta > maxPriceDelta/2 (rolling baseline strategy).
isPriceValidationSupported(address asset) → bool
getPriceValidationInfo(address asset) → (maxPriceDelta, lastValidatedPrice, lastValidatedTimestamp)
getConfiguredPriceValidationAssets() → address[] — Returns all assets with price validation configured.
```

---

## Deploy Metadata Libraries

### FuseMetadataTypes
- Source: `contracts/deploy/initialization/FuseMetadataTypes.sol`
- Purpose: Constants and helpers for fuse metadata classification in the deployment registry.
- Used by: Deployment scripts, FusionFactory initialization

**Metadata Type IDs**
| ID | Name | Values |
|----|------|--------|
| 0 | AUDIT_STATUS | UNAUDITED, REVIEWED, TESTED, AUDITED |
| 1 | SUBSTRATE_INFO | (protocol-specific) |
| 2 | CATEGORY_INFO | SUPPLY, BALANCE, DEX, PERPETUAL, BORROW, REWARDS, COLLATERAL, FLASH_LOAN, OTHER |
| 3 | ABI_VERSION | V1, V2 |
| 4 | PROTOCOL_INFO | AAVE, COMPOUND, CURVE, EULER, FLUID, GEARBOX, HARVEST, MOONWELL, MORPHO, RAMSES, ERC4626, ERC20, FUSION, META_MORPHO, UNISWAP, PENDLE |

**Functions**: `getAllFuseMetadataTypeIds()`, `getAllFuseMetadataTypeNames()`, `stringToBytes32Array(string)`.

### FuseStatus
- Source: `contracts/deploy/initialization/FuseStatus.sol`
- Purpose: Lifecycle status constants for fuse contracts.
- Used by: Deployment registry

**Statuses**
| ID | Name | Meaning |
|----|------|---------|
| 0 | DEFAULT | After deployment |
| 1 | ACTIVE | Usable |
| 2 | DEPRECATED | Should not be used |
| 3 | REMOVED | Must not be used |

**Functions**: `getAllFuseStatuIds()`, `getAllFuseStatusNames()`.

### FuseTypes
- Source: `contracts/deploy/initialization/FuseTypes.sol`
- Purpose: Canonical ID and name registry for all known fuse implementations (105 fuse types).
- Used by: Deployment scripts, fuse metadata registration

**Coverage**: Aave V2/V3, Compound V2/V3, Curve, ERC20, ERC4626 (14 market variants), Euler V2, Fluid Instadapp, Gearbox V3, Harvest, Moonwell, Morpho, Pendle, Ramses V2, Spark, Uniswap V2/V3, Universal Token Swapper, Meta Morpho, Burn Request Fee, Universal Reader, and Plasma Vault internal fuses.

**Functions**: `getAllFuseIds() → uint16[]`, `getAllFuseNames() → string[]`.

---

## Related Files
- `contracts/libraries/PlasmaVaultStorageLib.sol` -- ERC-7201 storage slots for vault state
- `contracts/libraries/PlasmaVaultLib.sol` -- vault state accessors (fee data, oracle, markets)
- `contracts/libraries/FusesLib.sol` -- fuse registration and balance fuse lookup
- `contracts/libraries/Roles.sol` -- role ID constants
- `contracts/libraries/math/IporMath.sol` -- WAD math, decimal conversion
- `contracts/managers/access/IporFusionAccessManagerInitializationLib.sol` -- InitializationData, RoleToFunction, AccountToRole structs
- `contracts/managers/access/RedemptionDelayLib.sol` -- redemption lock checking
- `contracts/managers/access/RoleExecutionTimelockLib.sol` -- execution delay enforcement
