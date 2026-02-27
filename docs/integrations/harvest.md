# Harvest Integration

## Quick Reference
- Fuse contracts: `HarvestDoHardWorkFuse.sol`
- External interfaces: `IHarvestController.sol`, `IHarvestVault.sol`
- Market ID: `HARVEST_HARD_WORK = 27`

## Overview

Harvest Finance is a yield farming protocol. This fuse triggers "hard work" operations on Harvest vaults, which harvests accumulated rewards and rebalances vault strategies. It is an operational fuse -- it does not supply or withdraw assets, only triggers maintenance actions.

## Fuse Interface

```
HarvestDoHardWorkFuse:
  constructor(marketId)

  enter(data: {vaults[]}):
    for each vault in vaults:
      1. Validate vault is granted as substrate (isSubstrateAsAssetGranted)
      2. Read comptroller = IHarvestVault(vault).controller()
      3. Validate comptroller != address(0)
      4. Call IHarvestController(comptroller).doHardWork(vault)
      5. Emit HarvestDoHardWorkFuseEnter event
    return (vaults[], comptrollers[])

  enterTransient():
    -> Same logic using transient storage for input/output
```

## Substrate Configuration

Vaults are registered as simple asset-type substrates (address encoded as bytes32). The fuse checks `isSubstrateAsAssetGranted()` for each vault address.

## Balance Tracking

No balance fuse -- this is a maintenance-only integration. The fuse does not hold or manage assets.

## Key Invariants

- Every vault address must be registered as a substrate for the market
- Each vault must have a non-zero controller (comptroller) address
- The fuse only triggers doHardWork; it does not transfer any tokens
- Multiple vaults can be processed in a single transaction

## Related Files
- Source: `contracts/fuses/harvest/HarvestDoHardWorkFuse.sol`
- Source: `contracts/fuses/harvest/ext/IHarvestController.sol`
- Source: `contracts/fuses/harvest/ext/IHarvestVault.sol`
