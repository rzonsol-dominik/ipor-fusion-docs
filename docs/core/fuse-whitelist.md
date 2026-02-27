# Fuse Whitelist System

## Overview

The Fuse Whitelist is a UUPS-upgradeable on-chain registry that catalogs all known fuse contracts in the IPOR Fusion ecosystem. It provides a canonical source of truth for fuse addresses, types, states, metadata, and market associations. The registry is governance-managed through granular role-based access control, allowing independent management of fuse types, states, metadata schemas, and fuse entries.

The contract also inherits `UniversalReader`, enabling off-chain `delegatecall`-based queries through the universal reader pattern.

## Architecture and Key Contracts

| Contract | Source | Role |
|----------|--------|------|
| `FuseWhitelist` | `contracts/fuses/whitelist/FuseWhitelist.sol` | Main registry contract (UUPS proxy entry point) |
| `FuseWhitelistAccessControl` | `contracts/fuses/whitelist/FuseWhitelistAccessControl.sol` | Abstract base defining all access control roles |
| `FuseWhitelistLib` | `contracts/fuses/whitelist/FuseWhitelistLib.sol` | Library implementing all storage operations, validation, and queries |

Inheritance chain: `FuseWhitelist` -> `UUPSUpgradeable`, `FuseWhitelistAccessControl` -> `AccessControlEnumerableUpgradeable`, `UniversalReader`.

## Data Model

The registry manages five independent data dimensions:

1. **Fuse Types** -- Named categories for fuses (e.g., "enter", "exit", "balance"). Identified by `uint16` ID. ID `0` is reserved and cannot be used.
2. **Fuse States** -- Lifecycle states (e.g., "active", "deprecated"). Identified by `uint16` ID.
3. **Metadata Types** -- Extensible key schemas for attaching arbitrary `bytes32[]` data to fuses. Identified by `uint16` ID.
4. **Fuse Entries** -- Individual fuse registrations linking an address to a type, state, deployment timestamp, and optional metadata.
5. **Market ID Associations** -- Automatic grouping of fuses by their `MARKET_ID()` return value (read from the fuse contract via `IFuseCommon`). Silently skipped if the fuse does not implement `MARKET_ID()`.

### FuseInfo Struct

```solidity
struct FuseInfo {
    uint16 fuseState;
    uint16 fuseType;
    address fuseAddress;
    uint32 timestamp;                                       // deployment timestamp
    mapping(uint256 metadataId => bytes32[] metadata) metadata;
    uint256[] metadataIds;
}
```

## Public/External Function Signatures

### Initialization

```
initialize(address initialAdmin_)                           // initializer, grants DEFAULT_ADMIN_ROLE
```

### Write Functions (role-gated)

```
addFuseTypes(uint16[] fuseTypeIds_, string[] fuseTypeNames_)
    -> bool                                                 // FUSE_TYPE_MANAGER_ROLE

addFuseStates(uint16[] fuseStateIds_, string[] fuseStateNames_)
    -> bool                                                 // FUSE_STATE_MANAGER_ROLE

addMetadataTypes(uint16[] metadataIds_, string[] metadataTypes_)
    -> bool                                                 // FUSE_METADATA_MANAGER_ROLE

addFuses(address[] fuses_, uint16[] types_, uint16[] states_, uint32[] deploymentTimestamps_)
    -> bool                                                 // ADD_FUSE_MANAGER_ROLE
    // Reverts if any fuse address already registered (no duplicates).
    // Auto-indexes by MARKET_ID if fuse implements IFuseCommon.

updateFuseType(address[] fuseAddresses_, uint16[] fuseTypes_)
    -> bool                                                 // UPDATE_FUSE_TYPE_MANAGER_ROLE
    // Moves fuse from old type list to new type list (swap-and-pop removal).

updateFuseState(address fuseAddress_, uint16 fuseState_)
    -> bool                                                 // UPDATE_FUSE_STATE_MANAGER_ROLE

updateFuseMetadata(address fuseAddress_, uint16 metadataId_, bytes32[] metadata_)
    -> bool                                                 // UPDATE_FUSE_METADATA_MANAGER_ROLE

updateFusesMetadata(address[] fuseAddresses_, uint16[] metadataIds_, bytes32[][] metadatas_)
    -> bool                                                 // UPDATE_FUSE_METADATA_MANAGER_ROLE

updateFuseDeploymentTimestamp(address fuseAddress_, uint32 deploymentTimestamp_)
    -> bool                                                 // UPDATE_FUSE_DEPLOYMENT_TIMESTAMP_MANAGER_ROLE

updateFusesDeploymentTimestamps(address[] fuseAddresses_, uint32[] deploymentTimestamps_)
    -> bool                                                 // UPDATE_FUSE_DEPLOYMENT_TIMESTAMP_MANAGER_ROLE
```

### View Functions (permissionless)

```
getFuseTypes()           -> (uint16[] ids, string[] names)
getFuseTypeDescription(uint16 fuseTypeId_) -> string
getFuseStates()          -> (uint16[] ids, string[] names)
getFuseStateName(uint16 fuseStateId_) -> string
getMetadataTypes()       -> (uint16[] ids, string[] names)
getMetadataType(uint16 metadataId_) -> string
getFusesByType(uint16 fuseTypeId_) -> address[]
getFuseByAddress(address fuseAddress_) -> (uint16 fuseState, uint16 fuseType, address fuseAddress, uint32 timestamp)
getFuseMetadataInfo(address fuseAddress_) -> (uint256[] metadataIds, bytes32[][] metadata)
getFusesByMarketId(uint256 marketId_) -> address[]
getFusesByTypeAndMarketIdAndStatus(uint16 type_, uint256 marketId_, uint16 status_) -> address[]
```

## Storage Layout

All storage uses ERC-7201 namespaced slots (diamond storage pattern) to prevent slot collisions across upgrades.

| Slot Constant | Namespace | Struct | Purpose |
|---------------|-----------|--------|---------|
| `FUSES_TYPES` | `io.ipor.whitelists.fuseTypes` | `FusesTypes` | Type ID -> name mapping + ID list |
| `FUSES_STATES` | `io.ipor.whitelists.fuseStates` | `FusesStates` | State ID -> name mapping + ID list |
| `METADATA_TYPES` | `io.ipor.whitelists.metadataTypes` | `MetadataTypes` | Metadata ID -> name mapping + ID list |
| `FUSE_INFO` | `io.ipor.whitelists.fuseInfo` | `FuseListByAddress` | Address -> `FuseInfo` mapping |
| `FUSE_LISTS_BY_TYPE` | `io.ipor.whitelists.fuseListsByType` | `FuseListsByType` | Type ID -> address[] mapping |
| `FUSE_INFO_BY_MARKET_ID` | `io.ipor.whitelists.fuseInfoByMarketId` | `FuseInfoByMarketId` | Market ID -> address[] mapping |

Slot computation: `keccak256(abi.encode(uint256(keccak256(namespace)) - 1)) & ~bytes32(uint256(0xff))`.

## Access Control Roles

Defined in `FuseWhitelistAccessControl`. All roles use OpenZeppelin `AccessControlEnumerableUpgradeable` (enumerable role members). `DEFAULT_ADMIN_ROLE` is the admin for all custom roles.

| Role | Hash | Protects |
|------|------|----------|
| `DEFAULT_ADMIN_ROLE` | `0x00` | `initialize`, `_authorizeUpgrade`, role management |
| `FUSE_TYPE_MANAGER_ROLE` | `keccak256("FUSE_TYPE_MANAGER_ROLE")` | `addFuseTypes` |
| `FUSE_STATE_MANAGER_ROLE` | `keccak256("FUSE_STATE_MANAGER_ROLE")` | `addFuseStates` |
| `FUSE_METADATA_MANAGER_ROLE` | `keccak256("FUSE_METADATA_MANAGER_ROLE")` | `addMetadataTypes` |
| `ADD_FUSE_MANAGER_ROLE` | `keccak256("ADD_FUSE_MANAGER_ROLE")` | `addFuses` |
| `UPDATE_FUSE_STATE_MANAGER_ROLE` | `keccak256("UPDATE_FUSE_STATE_MANAGER_ROLE")` | `updateFuseState` |
| `UPDATE_FUSE_TYPE_MANAGER_ROLE` | `keccak256("UPDATE_FUSE_TYPE_MANAGER_ROLE")` | `updateFuseType` |
| `UPDATE_FUSE_METADATA_MANAGER_ROLE` | `keccak256("UPDATE_FUSE_METADATA_MANAGER_ROLE")` | `updateFuseMetadata`, `updateFusesMetadata` |
| `UPDATE_FUSE_DEPLOYMENT_TIMESTAMP_MANAGER_ROLE` | `keccak256("UPDATE_FUSE_DEPLOYMENT_TIMESTAMP_MANAGER_ROLE")` | `updateFuseDeploymentTimestamp`, `updateFusesDeploymentTimestamps` |

## Key Invariants

- Fuse type ID `0` is reserved and cannot be used (`ZeroFuseTypeIdNotAllowed`).
- Fuse types, fuse states, and metadata types are append-only -- IDs cannot be removed or overwritten once registered.
- A fuse address can only be registered once (`FuseWhitelistFuseAlreadyExists`). No mechanism exists to remove a fuse entry.
- All batch operations require equal-length input arrays (`FuseWhitelistInvalidInputLength`).
- A fuse's type and state must reference previously registered type/state IDs at write time.
- Deployment timestamps must be non-zero.
- `updateFuseType` atomically removes the fuse from the old type list (swap-and-pop) and appends to the new type list.
- Market ID association is best-effort: silently skipped if the fuse contract does not implement `IFuseCommon.MARKET_ID()`.
- UUPS upgrade authorization is restricted to `DEFAULT_ADMIN_ROLE`.
- Constructor calls `_disableInitializers()` to prevent initialization of the implementation contract directly.

## Events

```
FuseTypeAdded(uint16 fuseId, string fuseType)
FuseStateAdded(uint16 stateId, string fuseState)
MetadataTypeAdded(uint16 metadataId, string metadataType)
FuseAddedToListByType(uint16 fuseTypeId, address fuseAddress)
FuseInfoAdded(address fuseAddress, uint16 fuseType, uint32 timestamp)
FuseStateUpdated(address fuseAddress, uint16 fuseState, uint16 fuseType)
FuseTypeUpdated(address fuseAddress, uint16 oldFuseType, uint16 newFuseType)
FuseMetadataUpdated(address fuseAddress, uint256 metadataId, bytes32[] metadata)
FuseAddedToMarketId(address fuseAddress, uint256 marketId)
FuseDeploymentTimestampUpdated(address fuseAddress, uint32 oldTimestamp, uint32 newTimestamp)
```

## Related Files

- `contracts/fuses/whitelist/FuseWhitelist.sol` -- main contract
- `contracts/fuses/whitelist/FuseWhitelistAccessControl.sol` -- role definitions
- `contracts/fuses/whitelist/FuseWhitelistLib.sol` -- storage library with all data operations
- `contracts/fuses/IFuseCommon.sol` -- `IFuseCommon` interface (`MARKET_ID()`)
- `contracts/universal_reader/UniversalReader.sol` -- inherited off-chain delegatecall reader
- `test/fuses/whitelist/FuseWhitelistTest.t.sol` -- Foundry test suite
