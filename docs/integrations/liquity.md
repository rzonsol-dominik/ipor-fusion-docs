# Liquity V2 Integration

## Quick Reference
- Fuse contracts: `LiquityStabilityPoolFuse.sol`, `LiquityBalanceFuse.sol`
- Market ID: `LIQUITY_V2 = 29`
- Chains: Ethereum
- External contracts: AddressesRegistry, StabilityPool, BOLD token

## Architecture
Liquity V2 uses a Stability Pool mechanism where users deposit BOLD (stablecoin) and earn:
1. **Collateral gains** from liquidations (e.g., ETH)
2. **BOLD yield gains** from interest payments

The fuse manages deposits/withdrawals from the Stability Pool. Substrates are **AddressesRegistry** addresses (one per collateral type). Each registry provides access to its StabilityPool, BOLD token, and collateral token.

## Fuse Interface

### LiquityStabilityPoolFuse
Deposits BOLD into and withdraws from the Liquity Stability Pool.

```
enter(registry, amount):
  -- Validates registry is granted substrate (as asset)
  -- stabilityPool = registry.stabilityPool()
  -- boldToken = registry.boldToken()
  -- approve(boldToken, amount) to stabilityPool
  -- stabilityPool.provideToSP(amount, claimCollateral=false)
  -- Reset approval to 0
  => returns (stabilityPool, amount)

  NOTE: Does NOT claim collateral on enter (to avoid forced swaps)

exit(registry, amount):
  -- Validates registry is granted substrate
  -- If amount == 0 AND no deposits: stabilityPool.claimAllCollGains()
  -- Else: stabilityPool.withdrawFromSP(amount, claimCollateral=true)
  => returns (stabilityPool, amount)

  NOTE: ALWAYS claims collateral on exit (to close position cleanly)
```

## Balance Tracking

Accounts for four components across all registries:

```
balanceOf():
  boldToken = firstRegistry.boldToken()
  (boldPrice, boldPriceDecimals) = oracle.getAssetPrice(boldToken)

  for each registry substrate:
    stabilityPool = registry.stabilityPool()
    collToken = registry.collToken()

    // Collateral value (stashed + unrealized gains from liquidations)
    collValue += convertToWad(
      (stabilityPool.stashedColl(self) + stabilityPool.getDepositorCollGain(self)) * collPrice
    )

    // BOLD value (compounded deposits + yield gains from interest)
    boldValue += convertToWad(
      (stabilityPool.getCompoundedBoldDeposit(self) + stabilityPool.getDepositorYieldGainWithPending(self)) * boldPrice
    )

  return collateralValue + boldValue  // USD in 18 decimals
```

## Key Invariants
- Substrates are AddressesRegistry addresses (one per collateral type)
- BOLD token is assumed to be the same across all registries
- Enter does NOT claim collateral (avoids forced swaps at deposit time)
- Exit ALWAYS claims collateral (ensures clean position closure)
- Zero-amount exit with no deposits triggers `claimAllCollGains()` only

## Risks & Edge Cases
- Stability Pool deposits lose BOLD during liquidations but gain discounted collateral
- Compounded deposit decreases as liquidations absorb BOLD from the pool
- Collateral gains must be claimed (via exit) before they can be used elsewhere
- Stashed collateral is claimed but not yet sent to the depositor
- No instant withdraw support -- positions must be exited explicitly
- Multiple registries can exist (one per collateral type like ETH, wstETH, etc.)

## Related Files
- Source: `contracts/fuses/liquity/`
- Tests: `test/fuses/liquity/`
- External interfaces: `ext/IAddressesRegistry.sol`, `ext/IStabilityPool.sol`
