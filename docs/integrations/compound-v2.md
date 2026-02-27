# Compound V2 Integration

## Quick Reference
- Fuse contracts: `CompoundV2SupplyFuse.sol`, `CompoundV2BalanceFuse.sol`
- Market ID: Not in IporFusionMarkets (legacy integration)
- Chains: Ethereum mainnet
- External contracts: cToken contracts (substrates are cToken addresses)

## Substrate Model
Unlike most fuses where substrates are underlying assets, Compound V2 substrates are **cToken addresses**. The fuse resolves the underlying asset by calling `cToken.underlying()`.

## Fuse Interface

### CompoundV2SupplyFuse
Supplies and withdraws assets via cToken mint/redeem. Supports instant withdraw.

```
enter(asset, amount):
  -- Finds cToken by iterating substrates: cToken.underlying() == asset
  -- approve(cToken, amount)
  -- cToken.mint(amount)
  => returns (asset, cToken, amount)

exit(asset, amount):
  -- Finds cToken by iterating substrates
  -- balance = cToken.balanceOfUnderlying(self)
  -- amountToWithdraw = min(amount, balance)
  -- successFlag = cToken.redeemUnderlying(amountToWithdraw)
  -- successFlag == 0 means success
  => returns (asset, cToken, amountToWithdraw)

instantWithdraw(params[amount, asset]):
  -- Same as exit but catches exceptions
```

## Balance Tracking

Uses PriceOracleMiddleware (not Compound's oracle).

```
balanceOf():
  for each cToken substrate:
    underlying = cToken.underlying()
    (price, priceDecimals) = priceOracleMiddleware.getAssetPrice(underlying)
    supplied = cToken.balanceOfUnderlying(self)
    borrowed = cToken.borrowBalanceCurrent(self)
    balance += convertToWad(supplied * price, decimals + priceDecimals)
    balance -= convertToWad(borrowed * price, decimals + priceDecimals)
  return balance  // USD in 18 decimals
```

Note: `balanceOf()` is NOT `view` -- `borrowBalanceCurrent` mutates state.

## Key Invariants
- Substrates are cToken addresses, not underlying assets
- cToken.redeemUnderlying returns 0 on success, non-zero on error
- Balance fuse uses PriceOracleMiddleware from PlasmaVault storage
- Supply fuse iterates all substrates to find matching cToken (O(n) lookup)

## Risks & Edge Cases
- Compound V2 is legacy; new supply may be restricted on some markets
- `borrowBalanceCurrent` triggers interest accrual (state mutation) -- balance fuse is not view-only
- If no cToken matches the asset, the fuse reverts with `CompoundV2SupplyFuseUnsupportedAsset`
- cToken exchange rates can cause rounding differences on redeem

## Related Files
- Source: `contracts/fuses/compound_v2/`
- Tests: `test/fuses/compound_v2/`
- External interfaces: `ext/CErc20.sol`
