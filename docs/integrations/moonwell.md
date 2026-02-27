# Moonwell Integration

## Quick Reference
- Fuse contracts: `MoonwellSupplyFuse.sol`, `MoonwellBorrowFuse.sol`, `MoonwellEnableMarketFuse.sol`, `MoonwellBalanceFuse.sol`, `MoonwellHelperLib.sol`
- Market ID: `MOONWELL = 21`
- Chains: Base, Moonbeam
- External contracts: mToken contracts (Compound V2 fork), Comptroller

## Substrate Model
Substrates are **mToken addresses** (similar to Compound V2). The underlying asset is resolved via `mToken.underlying()`. A helper library `MoonwellHelperLib` iterates substrates to find the mToken for a given asset.

## Fuse Interface

### MoonwellSupplyFuse
Supplies and withdraws assets via mToken mint/redeemUnderlying. Supports instant withdraw.

```
enter(asset, amount):
  -- Finds mToken via MoonwellHelperLib.getMToken(substrates, asset)
  -- finalAmount = min(amount, asset.balanceOf(self))
  -- approve(mToken, finalAmount)
  -- mToken.mint(finalAmount)  -- reverts if returns non-zero
  => returns (asset, mToken, finalAmount)

exit(asset, amount):
  -- Finds mToken
  -- balance = mToken.balanceOfUnderlying(self)
  -- amountToWithdraw = min(amount, balance)
  -- mToken.redeemUnderlying(amountToWithdraw)
  => returns (asset, mToken, amount)

instantWithdraw(params[amount, asset]):
  -- Same as exit but catches exceptions
```

### MoonwellBorrowFuse
Borrows and repays assets via mToken borrow/repayBorrow.

```
enter(asset, amount):
  -- Finds mToken
  -- mToken.borrow(amount)  -- reverts if returns non-zero
  => returns (asset, mToken, amount)

exit(asset, amount):
  -- Finds mToken
  -- approve(mToken, amount)
  -- mToken.repayBorrow(amount)  -- reverts if returns non-zero
  -- Reset approval to 0
  => returns (asset, mToken, amount)
```

### MoonwellEnableMarketFuse
Enables/disables mTokens as collateral via Comptroller.

```
enter(mTokens[]):
  -- Validates each mToken is in substrates
  -- COMPTROLLER.enterMarkets(mTokens)
  -- Checks error codes

exit(mTokens[]):
  -- For each mToken: COMPTROLLER.exitMarket(mToken)
  -- Checks error codes per token
```

## Balance Tracking

Uses PriceOracleMiddleware. Note: `balanceOf()` is NOT `view` (mToken balance calls mutate state).

```
balanceOf():
  for each mToken substrate:
    underlying = mToken.underlying()
    (price, priceDecimals) = priceOracleMiddleware.getAssetPrice(underlying)
    supplied = mToken.balanceOfUnderlying(self)  // accrues interest
    borrowed = mToken.borrowBalanceStored(self)
    balance += convertToWad(supplied * price, decimals + priceDecimals)
    balance -= convertToWad(borrowed * price, decimals + priceDecimals)
  return max(0, balance)  // clamps to zero if negative
```

## Key Invariants
- Substrates are mToken addresses, not underlying assets
- Markets must be entered (via EnableMarketFuse) before borrowing against collateral
- mToken operations return 0 on success, non-zero error codes on failure
- Balance fuse clamps negative balance to zero (unlike Aave which reverts)
- Supply fuse caps deposit at vault's available balance

## Risks & Edge Cases
- `balanceOfUnderlying` triggers interest accrual -- not a view function
- Comptroller.exitMarket may fail if there is outstanding debt
- Error codes from Moonwell are checked but may not provide detailed failure info
- Moonwell is a Compound V2 fork -- similar risks apply (utilization, liquidation)

## Related Files
- Source: `contracts/fuses/moonwell/`
- Tests: `test/fuses/moonwell/`
- External interfaces: `ext/MErc20.sol`, `ext/MComptroller.sol`, `ext/MultiRewardDistributor.sol`
