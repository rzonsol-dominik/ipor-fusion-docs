# Fluid Instadapp Integration

## Quick Reference
- Fuse contracts: `FluidInstadappStakingSupplyFuse.sol`, `FluidInstadappStakingBalanceFuse.sol`
- Market IDs: `FLUID_INSTADAPP_POOL = 5`, `FLUID_INSTADAPP_STAKING = 6`
- Chains: Ethereum, Arbitrum
- External contracts: FluidLendingStakingRewards (staking pool), fToken (ERC4626 vault)

## Architecture
Similar to Gearbox V3, Fluid Instadapp uses a two-layer model:
1. **fToken (stakingToken)** -- an ERC4626 vault representing a lending position in Fluid
2. **stakingPool** -- a staking rewards contract where fTokens are staked to earn additional rewards (e.g., ARB)

The fuse handles staking/unstaking fTokens in the staking rewards pool.

## Fuse Interface

### FluidInstadappStakingSupplyFuse
Stakes and unstakes fTokens in Fluid Instadapp staking pools. Supports instant withdraw.

```
enter(fluidTokenAmount, stakingPool):
  -- Validates stakingPool is granted substrate (as asset)
  -- stakingToken = stakingPool.stakingToken()
  -- deposit = min(fluidTokenAmount, stakingToken.balanceOf(self))
  -- approve(stakingToken, deposit) to stakingPool
  -- stakingPool.stake(deposit)
  => returns (stakingPool, stakingToken, deposit)

exit(fluidTokenAmount, stakingPool):
  -- Validates stakingPool is granted substrate
  -- balance = stakingPool.balanceOf(self)
  -- withdrawAmount = min(fluidTokenAmount, balance)
  -- stakingPool.withdraw(withdrawAmount)
  => returns (stakingPool, withdrawAmount)

instantWithdraw(params[underlyingAmount, stakingPool]):
  -- Converts underlying amount to fToken shares via fToken.previewWithdraw
  -- Same as exit but catches exceptions
```

## Balance Tracking

Uses only the first substrate for balance calculation.

```
balanceOf():
  stakingPool = substrates[0]
  stakingToken = stakingPool.stakingToken()  // fToken (ERC4626)
  underlyingAsset = stakingToken.asset()

  stakedBalance = stakingPool.balanceOf(self)
  underlyingAssets = stakingToken.convertToAssets(stakedBalance)

  if underlyingAssets == 0: return 0

  (price, priceDecimals) = priceOracleMiddleware.getAssetPrice(underlyingAsset)
  return convertToWad(underlyingAssets * price, assetDecimals + priceDecimals)
```

## Key Invariants
- Substrates are staking pool addresses
- Staking token (fToken) is an ERC4626 vault
- Balance fuse uses ONLY the first substrate
- Instant withdraw uses `previewWithdraw` to convert underlying amounts to fToken shares (rounds up)
- Early return if underlying assets are zero

## Risks & Edge Cases
- Only the first substrate is used for balance -- additional substrates ignored
- Staking rewards (e.g., ARB) are not included in the balance calculation
- `previewWithdraw` rounds up for withdrawal (correct for burning shares)
- Fluid lending positions may have withdrawal limits depending on utilization

## Related Files
- Source: `contracts/fuses/fluid_instadapp/`
- Tests: `test/fuses/fluid_instadapp/`
- External interfaces: `ext/IFluidLendingStakingRewards.sol`
