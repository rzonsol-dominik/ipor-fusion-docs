# Context Manager

## Overview

The Context Manager system enables batch execution of calls to approved contracts while preserving the original sender's identity. It supports two modes: direct execution (where `msg.sender` is the context sender) and signature-based execution (where an ECDSA signature proves the sender's intent, allowing a relayer to submit the transaction).

## Architecture

```
ContextManager                         ContextClient (target contract)
  |                                       |
  | 1. setupContext(sender)  ---------->  | stores sender in isolated storage slot
  | 2. functionCall(data)    ---------->  | reads sender via _getSenderFromContext()
  | 3. clearContext()        ---------->  | clears sender storage
```

## ContextManager

**Source:** `contracts/managers/context/ContextManager.sol`

| Function | Access | Description |
|----------|--------|-------------|
| `runWithContext(ExecuteData)` | Public | Batch-executes calls with `msg.sender` as context sender |
| `runWithContextAndSignature(ContextDataWithSender[])` | Public | Executes calls with ECDSA-verified sender impersonation |
| `addApprovedTargets(address[])` | ATOMIST_ROLE | Adds targets to whitelist |
| `removeApprovedTargets(address[])` | ATOMIST_ROLE | Removes targets from whitelist |
| `getApprovedTargets()` | View | Returns array of all approved target addresses |
| `getNonce(address)` | View | Returns current nonce for replay protection |
| `isTargetApproved(address)` | View | Checks whitelist status |
| `proxyInitialize(address, address[])` | Initializer | Proxy init with access manager and initial approved targets |

**Signature scheme** (non-EIP-712):
```
digest = keccak256(abi.encodePacked(
    contextManagerAddress, expirationTime, nonce, chainId, target, data
))
```
Signature must recover to `sender`. Nonces are strictly increasing per sender. `CHAIN_ID` is set immutably at deployment.

## ContextClient

**Source:** `contracts/managers/context/ContextClient.sol`

Abstract contract that target contracts inherit. Provides:

```solidity
setupContext(sender)         // restricted to TECH_CONTEXT_MANAGER_ROLE
clearContext()               // restricted to TECH_CONTEXT_MANAGER_ROLE
_getSenderFromContext()      // internal -- returns context sender or msg.sender if no context set
```

Single-context enforcement: `setupContext` reverts if a context is already active (`ContextAlreadySet`).

## Storage Libraries

**ContextManagerStorageLib** (`contracts/managers/context/ContextManagerStorageLib.sol`):
- Manages approved targets (mapping + array for enumeration)
- Manages per-sender nonces with `verifyAndUpdateNonce(sender, newNonce)`
- Uses ERC-7201 namespaced storage slots

**ContextClientStorageLib** (`contracts/managers/context/ContextClientStorageLib.sol`):
- Single storage slot holding `contextSender` address
- `getSenderFromContext()`: returns `contextSender` if set, otherwise `msg.sender`
- Uses ERC-7201 namespaced storage slot (`0x68262f...8b00`)

## Key Invariants

- Only approved targets can be called through `runWithContext` / `runWithContextAndSignature`.
- Context is always set before the call and cleared after. There is no try/catch — if a call reverts, the entire transaction reverts and transient state is discarded.
- Signature nonces must strictly increase -- replay of old nonces reverts with `NonceTooLow`.
- Signatures expire after `expirationTime` (block.timestamp check).
- `_getSenderFromContext()` falls back to `msg.sender` when no context is active, maintaining backward compatibility.
- `setupContext` / `clearContext` are restricted to `TECH_CONTEXT_MANAGER_ROLE`.

## Related Files

- `contracts/managers/context/` -- all context manager contracts
- `contracts/managers/context/IContextClient.sol` -- interface definition
- `contracts/managers/access/AccessManagedUpgradeable.sol` -- role-based access control base
