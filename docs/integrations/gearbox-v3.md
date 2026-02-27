# Gearbox V3 Integration

## Quick Reference
- Fuse contracts: `GearboxV3FarmSupplyFuse.sol`, `GearboxV3FarmBalanceFuse.sol`
- Market IDs: `GEARBOX_POOL_V3 = 3`, `GEARBOX_FARM_DTOKEN_V3 = 4`
- Chains: Ethereum, Arbitrum
- External contracts: FarmingPool (staking), dToken (ERC4626 vault)

## Architecture
The Gearbox V3 farm integration works with a two-layer token model:
1. **dToken** -- an ERC4626 vault representing a deposit position in a Gearbox pool
2. **farmDToken** -- a farming/staking wrapper around dToken that earns additional rewards (e.g., ARB)

The fuse handles staking/unstaking dTokens into the farming pool. The exchange rate between farmDToken and dToken is 1:1 (farming pool just tracks staking balance).

## Fuse Interface

### GearboxV3FarmSupplyFuse
Stakes and unstakes dTokens in Gearbox V3 farming pools. Supports instant withdraw.

```
enter(dTokenAmount, farmdToken):
  -- Validates farmdToken is granted substrate (as asset)
  -- dToken = farmdToken.stakingToken()
  -- depositAmount = min(dTokenAmount, dToken.balanceOf(self))
  -- approve(dToken, depositAmount) to farmdToken
  -- farmdToken.deposit(depositAmount)
  => returns (farmdToken, dToken, depositAmount)

exit(dTokenAmount, farmdToken):
  -- Validates farmdToken is granted substrate
  -- withdrawAmount = min(dTokenAmount, farmdToken.balanceOf(self))
  -- farmdToken.withdraw(withdrawAmount)
  => returns (farmdToken, withdrawAmount)

instantWithdraw(params[underlyingAmount, farmdToken]):
  -- Converts underlying amount to dToken shares via dToken.previewWithdraw
  -- Same as exit but catches exceptions
```

## Balance Tracking

Uses only the first substrate for balance calculation.

```
balanceOf():
  farmDToken = substrates[0]
  dToken = farmDToken.stakingToken()
  underlyingAsset = dToken.asset()  // ERC4626

  stakedBalance = farmDToken.balanceOf(self)
  // farmDToken:dToken = 1:1 exchange rate
  underlyingAssets = dToken.convertToAssets(stakedBalance)

  (price, priceDecimals) = priceOracleMiddleware.getAssetPrice(underlyingAsset)
  return convertToWad(underlyingAssets * price, assetDecimals + priceDecimals)
```

## Key Invariants
- Substrates are farmDToken addresses (farming pool addresses)
- farmDToken to dToken is 1:1 exchange rate
- dToken to underlying uses ERC4626 convertToAssets
- Balance fuse uses ONLY the first substrate
- Instant withdraw converts underlying amount to dToken shares via `previewWithdraw`

## Risks & Edge Cases
- Only the first substrate is used for balance -- multiple substrates would be partially ignored
- Farming rewards (e.g., ARB) are not accounted for in the balance calculation
- dToken withdrawal from the Gearbox pool may have fees or delays
- `previewWithdraw` rounds up for withdrawals (correct behavior for burning shares)

## Related Files
- Source: `contracts/fuses/gearbox_v3/`
- Tests: `test/fuses/gearbox_v3/`
- External interfaces: `ext/IFarmingPool.sol`
