# Transient Storage Pattern

## Overview

IPOR Fusion uses EVM transient storage (EIP-1153 `TSTORE`/`TLOAD`) to pass parameters between fuses within a single transaction. When a PlasmaVault executes a fuse chain via `enterTransient`/`exitTransient`, fuses can write outputs and read inputs through `TransientStorageLib`, enabling data flow between steps without persistent storage costs.

The pattern: **SetInputs -> Fuse A (writes outputs) -> Mapper (reads A's outputs, writes B's inputs) -> Fuse B (reads inputs)**.

All transient storage is automatically cleared at the end of the transaction by the EVM.

## TransientStorageLib

**Source:** `contracts/transient_storage/TransientStorageLib.sol`

Library providing typed access to per-address input/output arrays in transient storage.

| Function | Description |
|----------|-------------|
| `setInputs(account, bytes32[])` | Clears then writes input array for account |
| `setInput(account, index, value)` | Overwrites single input at index (must be pre-initialized) |
| `getInputs(account)` / `getInput(account, index)` | Read inputs |
| `setOutputs(account, bytes32[])` | Clears then writes output array for account |
| `getOutputs(account)` / `getOutput(account, index)` | Read outputs |
| `clearInputs(account)` / `clearOutputs(account)` | Zero out arrays |

Storage layout per account (ERC-7201 namespaced):
```
slot = keccak256(account . TRANSIENT_STORAGE_SLOT)
  [slot]     = inputs.length
  [keccak256(slot) + i] = inputs[i]
  [slot + 1] = outputs.length
  [keccak256(slot+1) + i] = outputs[i]
```

Accounts are identified by fuse `VERSION` address (the fuse's own deployment address).

## Transient Storage Fuses

### TransientStorageSetInputsFuse

**Source:** `contracts/fuses/transient_storage/TransientStorageSetInputsFuse.sol`

Initializes input arrays for downstream fuses. Called at the start of a chain.

```solidity
enter(TransientStorageSetInputsFuseEnterData{fuse[], inputsByFuse[][]})
```

Sets `TransientStorageLib.setInputs(fuse[i], inputsByFuse[i])` for each fuse. Validates non-zero addresses and non-empty inputs.

### TransientStorageChainReaderFuse

**Source:** `contracts/fuses/transient_storage/TransientStorageChainReaderFuse.sol`

Reads on-chain data via static calls to external contracts and stores results in its own outputs.

```solidity
enter(bytes calldata data)
// data is abi-decoded as ExternalCalls{calls[], responseLength}
```

The `data` parameter is raw `bytes calldata` that gets ABI-decoded into an `ExternalCalls` struct internally. Each `ExternalCall` contains a target, calldata, and `ReadDataFromResponse[]` readers that extract typed values (uint256, int128, address, bool, etc.) from specific byte ranges of the response. Results are collected into a `bytes32[]` array and stored via `setOutputs(VERSION, results)`.

### TransientStorageMapperFuse

**Source:** `contracts/fuses/transient_storage/TransientStorageMapperFuse.sol`

Routes data between fuses with type and decimal conversion.

```solidity
enter(TransientStorageMapperEnterData{items[]})
```

Each `TransientStorageMapperItem` specifies:
- Source: `paramType` (INPUT or OUTPUT), `dataFromAddress`, `dataFromIndex`, `dataFromType`, `dataFromDecimals`
- Destination: `dataToAddress`, `dataToIndex`, `dataToType`, `dataToDecimals`

Supports all numeric types (uint8-256, int8-256), address, bool, bytes32. Handles decimal scaling (up to 10^77 difference). Writes to destination via `setInput()` (destination must be pre-initialized).

Max 256 items per call to prevent gas DoS.

## Typical Chain Flow

`SetInputsFuse -> FuseA (writes outputs) -> MapperFuse (A.outputs -> B.inputs) -> ChainReaderFuse (reads external data) -> MapperFuse (reader.outputs -> C.inputs) -> FuseC`

## Key Invariants

- All data is in transient storage: automatically cleared at end of transaction.
- `setInput()` requires the target array to be pre-initialized with sufficient length (via `setInputs()`).
- ChainReaderFuse only makes `staticcall`s -- cannot modify external state.
- MapperFuse validates all type conversions and reverts on overflow or invalid conversion paths (e.g., signed -> address).
- Fuses are identified by their `VERSION` (immutable deployment address), not `address(this)` during delegatecall.

## Related Files

- `contracts/transient_storage/TransientStorageLib.sol` -- core library
- `contracts/fuses/transient_storage/` -- all transient storage fuses
- `contracts/libraries/TypeConversionLib.sol` -- type conversion helpers
- `contracts/libraries/IporFusionMarkets.sol` -- market ID constants
