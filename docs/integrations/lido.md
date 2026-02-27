# Lido Integration

## Quick Reference
- Fuse contracts: `StEthWrapperFuse.sol`
- External interface: `IWstETH.sol`
- Chain: Ethereum mainnet only
- No market ID (chain-specific utility fuse)

## Overview

Lido is an Ethereum liquid staking protocol. This fuse wraps stETH into wstETH and unwraps wstETH back to stETH. It is a simple utility fuse for managing the stETH/wstETH conversion within the PlasmaVault.

## Fuse Interface

```
StEthWrapperFuse:
  ST_ETH  = 0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84
  WST_ETH = 0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0

  constructor(marketId)

  enter(stEthAmount) -> (finalAmount, wstEthAmount):
    1. Return early if amount is 0
    2. Validate substrates (stETH and wstETH must be granted or underlying)
    3. Cap amount to available stETH balance
    4. Approve wstETH contract, call IWstETH.wrap(amount)
    5. Reset approval
    6. Return amounts

  exit(wstEthAmount) -> (finalAmount, stEthAmount):
    1. Return early if amount is 0
    2. Validate substrates
    3. Cap amount to available wstETH balance
    4. Call IWstETH.unwrap(amount)
    5. Return amounts

  enterTransient() / exitTransient():
    -> Same logic using transient storage
```

## Substrate Configuration

Both stETH and wstETH must be granted as asset substrates for the market, unless they are the vault's underlying asset. Validation is done via `isSubstrateAsAssetGranted()`.

## Balance Tracking

No dedicated balance fuse -- the wrapped/unwrapped tokens are tracked by the vault's normal asset accounting.

## Key Invariants

- Hardcoded to Ethereum mainnet addresses (stETH and wstETH)
- Both stETH and wstETH must be granted substrates (or be the underlying asset)
- Amounts are capped to available vault balance before operation
- Approval is reset to zero after wrapping

## Related Files
- Source: `contracts/fuses/chains/ethereum/lido/StEthWrapperFuse.sol`
- Source: `contracts/fuses/chains/ethereum/lido/ext/IWstETH.sol`
