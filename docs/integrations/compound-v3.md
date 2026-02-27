# Compound V3 Integration

## Quick Reference
- Fuse contracts: `CompoundV3SupplyFuse.sol`, `CompoundV3BalanceFuse.sol`
- Market IDs: `COMPOUND_V3_USDC = 2`, `COMPOUND_V3_USDT = 13`, `COMPOUND_V3_WETH = 26`
- Chains: Ethereum, Arbitrum, others where Compound V3 (Comet) is deployed
- External contracts: Comet (market-specific, set at construction)

## Fuse Interface

### CompoundV3SupplyFuse
Supplies and withdraws assets to/from Compound V3 (Comet). Supports instant withdraw.

Constructor takes `cometAddress_` which determines the specific Comet market. The base token is read from `COMET.baseToken()`.

```
enter(asset, amount):
  -- Validates asset is granted substrate for MARKET_ID
  -- approve(COMET, amount)
  -- COMET.supply(asset, amount)
  => returns (asset, COMET address, amount)

exit(asset, amount):
  -- Validates asset is granted substrate
  -- If asset == baseToken: balance = COMET.balanceOf(self)
  -- Else: balance = COMET.collateralBalanceOf(self, asset)
  -- finalAmount = min(amount, balance)
  -- COMET.withdraw(asset, finalAmount)
  => returns (asset, COMET address, finalAmount)

instantWithdraw(params[amount, asset]):
  -- Same as exit but catches exceptions
```

## Balance Tracking

```
balanceOf():
  for each substrate asset:
    if asset == baseToken:
      balance = COMET.balanceOf(self)
      price = COMET.getPrice(baseTokenPriceFeed)
    else:
      balance = COMET.collateralBalanceOf(self, asset)
      price = COMET.getPrice(asset.priceFeed)
    total += convertToWad(balance * price, decimals + 8)

  borrowBalance = COMET.borrowBalanceOf(self)
  borrowValue = convertToWad(borrowBalance * baseTokenPrice, baseDecimals + 8)
  total -= borrowValue

  return total  // USD in 18 decimals
```

## Key Invariants
- Each Comet market has one base token (the lendable asset) and multiple collateral tokens
- Supplying the base token earns interest; supplying other tokens is collateral-only
- Borrow balance is always in base token terms
- Price feeds use Compound's internal oracle with 8 decimal precision
- No separate borrow fuse -- borrowing is implicit when collateral is supplied and base is withdrawn

## Risks & Edge Cases
- Compound V3 has supply caps per collateral asset
- Liquidation risk if collateral ratio drops below threshold
- Each Comet contract is a single market -- separate fuse instances needed per market
- Instant withdraw catches exceptions -- position may remain open

## Related Files
- Source: `contracts/fuses/compound_v3/`
- Tests: `test/fuses/compound_v3/`
- External interfaces: `ext/IComet.sol`
