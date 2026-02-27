# Aave V2 Integration

## Quick Reference
- Fuse contracts: `AaveV2SupplyFuse.sol`, `AaveV2BalanceFuse.sol`, `AaveConstantsEthereum.sol`
- Market ID: Not in IporFusionMarkets (legacy integration)
- Chains: Ethereum mainnet only (hardcoded addresses)
- External contracts:
  - Aave Lending Pool V2: `0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9`
  - Aave Price Oracle: `0x54586bE62E3c3580375aE3723C145253060Ca0C2`

## Fuse Interface

### AaveV2SupplyFuse
Supply-only fuse (no borrow fuse exists for V2). Supports instant withdraw.

```
enter(asset, amount):
  -- Validates asset is granted substrate for MARKET_ID
  -- approve(AAVE_POOL, amount)
  -- AAVE_POOL.deposit(asset, amount, self, 0)
  => returns (asset, amount)

exit(asset, amount):
  -- Validates asset is granted substrate
  -- Gets aToken from AAVE_LENDING_POOL_V2.getReserveData(asset)
  -- amountToWithdraw = min(aToken.balanceOf(self), amount)
  -- AAVE_POOL.withdraw(asset, amountToWithdraw, self)
  => returns (asset, withdrawnAmount)

instantWithdraw(params[amount, asset]):
  -- Same as exit but catches exceptions (emits ExitFailed on error)
```

## Balance Tracking

```
balanceOf():
  for each substrate asset:
    reserveData = AAVE_LENDING_POOL_V2.getReserveData(asset)
    net = aToken.balanceOf - stableDebt.balanceOf - variableDebt.balanceOf
    price = AAVE_PRICE_ORACLE_MAINNET.getAssetPrice(asset)  // 8 decimals
    balance += convertToWad(net * price, decimals + 8)
  return balance  // USD in 18 decimals
```

## Key Invariants
- Assets must be registered as substrates
- Withdrawal capped at aToken balance
- Hardcoded to Ethereum mainnet addresses -- cannot be deployed to other chains
- Balance includes debt tokens (stable + variable) as negative values

## Risks & Edge Cases
- Aave V2 is in wind-down mode; new deposits may be restricted
- Hardcoded oracle address could become stale if Aave governance migrates
- No borrow fuse means debt positions cannot be opened through this integration
- Price oracle uses 8 decimal precision

## Related Files
- Source: `contracts/fuses/aave_v2/`
- Tests: `test/fuses/aave_v2/`
- External interfaces: `ext/AaveLendingPoolV2.sol`, `ext/IAavePriceOracle.sol`
