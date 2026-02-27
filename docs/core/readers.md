# Readers

## Overview

Readers are stateless contracts that extract configuration and state data from PlasmaVault instances. They use the **UniversalReader pattern**: the vault `delegatecall`s into the reader's code, giving the reader access to the vault's storage without the reader holding any state itself. Each reader exposes both a "raw" function (designed for delegatecall) and a convenience wrapper that routes through UniversalReader automatically.

## UniversalReader (Abstract Base)

Inherited by every PlasmaVault. Provides a two-step secure read:

```
read(target, data)            // external view -- validates target, calls readInternal via staticcall
  -> readInternal(target, data)  // onlyThis -- delegatecalls target with data
```

This guarantees reads cannot modify state. The `ReadResult` struct wraps returned `bytes`.

**Source:** `contracts/universal_reader/UniversalReader.sol`

## Reader Contracts

| Reader | Purpose | Key Function(s) |
|--------|---------|-----------------|
| `BalanceFusesReader` | Returns parallel arrays of `marketIds` and their balance `fuseAddresses` | `getBalanceFuseInfo(vault)` |
| `CallbackHandlerReader` | Looks up callback handler address by `(sender, sig)` key pair | `getCallbackHandler(vault, sender, sig)`, `getCallbackHandlers(vault, senders[], sigs[])` |
| `PreHooksInfoReader` | Returns all pre-hook configurations (selector, implementation, substrates) | `getPreHooksInfo(vault)` |
| `ReadAsyncExecutor` | Returns the AsyncExecutor address from vault storage | `readAsyncExecutorAddress()` (delegatecall only) |
| `EbisuWethEthAdapterAddressReader` | Returns WethEthAdapter address; includes Bech32 address conversion helpers | `getEbisuWethEthAdapterAddress(vault)` |
| `TacStakingDelegatorAddressReader` | Returns TAC staking delegator address; includes Bech32 conversion helpers | `getTacStakingDelegatorAddress(vault)` |
| `ReadBalanceFuses` | Returns balance fuse addresses for all active fuses (deduplicates market IDs) | `getBalanceFusesForActiveFuses()`, `getBalanceFuse(marketId)` |

## ReadBalanceFuses

Located in `contracts/universal_reader/ReadBalanceFuses.sol`. Unlike the other readers, this one discovers market IDs dynamically by iterating all registered fuses, calling `MARKET_ID()` on each, deduplicating, and always including `ERC20_VAULT_BALANCE`. It must be called via delegatecall.

## Usage Pattern

```
// Off-chain or from another contract:
BalanceFusesReader reader = BalanceFusesReader(readerAddress);
(uint256[] memory ids, address[] memory fuses) = reader.getBalanceFuseInfo(vaultAddress);
```

Internally this calls `UniversalReader(vault).read(reader, calldata)`, which delegatecalls the reader in the vault's storage context.

## Key Invariants

- Reader functions that access vault storage directly (no `plasmaVault_` parameter) MUST only be called via delegatecall through UniversalReader. Direct external calls will read from the wrong storage context.
- `UniversalReader.read()` is a `view` function -- it cannot modify state.
- `readInternal()` is restricted to `onlyThis` (self-call via staticcall).

## Related Files

- `contracts/readers/` -- all reader contracts
- `contracts/universal_reader/UniversalReader.sol` -- base read pattern
- `contracts/universal_reader/ReadBalanceFuses.sol` -- active fuse discovery
- `contracts/libraries/PlasmaVaultStorageLib.sol` -- storage layout readers access
