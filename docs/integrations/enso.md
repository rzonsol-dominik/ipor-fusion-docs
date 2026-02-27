# Enso Integration

## Quick Reference
- Fuse contracts: `EnsoFuse.sol`, `EnsoBalanceFuse.sol`, `EnsoExecutor.sol`, `EnsoInitExecutorFuse.sol`
- Libraries: `EnsoSubstrateLib.sol`, `EnsoStorageLib.sol`
- Market ID: `ENSO = 38`

## Overview

Enso is a DeFi routing protocol that executes complex multi-step "shortcut" operations via delegatecall. The integration uses a persistent EnsoExecutor contract (stored in ERC-7201 namespaced storage) that receives tokens, executes Enso commands, and tracks remaining balances. Supports cross-chain scenarios where tokens may not be immediately available.

## Fuse Interface

```
EnsoFuse:
  constructor(marketId, weth, delegateEnsoShortcuts)

  enter(data: {tokenOut, amountOut, wEthAmount, accountId, requestId,
               commands[], state[], tokensToReturn[]}):
    1. Validate tokenOut != address(0)
    2. Validate enter substrates (tokenOut, WETH if used, tokensToReturn)
    3. Validate commands (skip STATICCALL, check CALL/VALUECALL targets)
    4. Create executor if not exists (stored in ERC-7201 slot)
    5. Transfer tokenOut and/or WETH to executor
    6. Call executor.execute() which delegatecalls DelegateEnsoShortcuts
    7. Executor tracks balance for NAV reporting

  exit(data: {tokens[]}):
    1. Validate exit substrates (all tokens must have transfer granted)
    2. Call executor.withdrawAll(tokens)
    3. Executor transfers all balances to PlasmaVault and resets balance
```

```
EnsoInitExecutorFuse:
  enter():
    -> Creates EnsoExecutor and stores address if not exists

EnsoExecutor:
  execute(data):
    1. Require balance not already set (prevents double-entry)
    2. Unwrap WETH to ETH if needed
    3. Delegatecall DelegateEnsoShortcuts.executeShortcut()
    4. Wrap remaining ETH back to WETH, transfer to vault
    5. Track remaining tokenOut balance for NAV

  withdrawAll(tokens):
    -> Transfer all token balances to PlasmaVault
    -> Reset tracked balance to zero

  getBalance() -> (assetAddress, assetBalance)
    -> Returns tracked asset for balance fuse

  recovery(target, data):
    -> Emergency delegatecall (only when balance is zero)
```

## Substrate Configuration

Substrates encode (target address, function selector) pairs:

| Layout | Description |
|--------|-------------|
| `[address (20B) | selector (4B) | padding (8B)]` | Allowed target + function |

Tokens require `ERC20.transfer` selector as their substrate entry.

## Balance Tracking

`EnsoBalanceFuse.balanceOf()`:
- Reads executor address from ERC-7201 storage
- Calls `executor.getBalance()` to get tracked (asset, amount)
- Converts to USD via PriceOracleMiddleware
- Handles cross-chain scenarios where balance may be pending

The executor stores balance in a packed struct (address + uint96) fitting one storage slot.

## Key Invariants

- Executor balance must be zero before a new `enter()` (prevents state conflicts)
- Commands with STATICCALL flag skip substrate validation (read-only)
- Commands with EXTENDED_COMMAND flag cause the next command to be skipped
- All token transfers require an explicit substrate grant (target + transfer selector)
- The executor is only callable by the PlasmaVault
- Recovery function only works when executor balance is zero

## Related Files
- Source: `contracts/fuses/enso/EnsoFuse.sol`
- Source: `contracts/fuses/enso/EnsoBalanceFuse.sol`
- Source: `contracts/fuses/enso/EnsoExecutor.sol`
- Source: `contracts/fuses/enso/EnsoInitExecutorFuse.sol`
- Source: `contracts/fuses/enso/lib/EnsoSubstrateLib.sol`
- Source: `contracts/fuses/enso/lib/EnsoStorageLib.sol`
