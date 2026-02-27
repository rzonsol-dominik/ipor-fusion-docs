# Morpho (Blue) Integration

## Quick Reference
- Fuse contracts: `MorphoSupplyFuse.sol`, `MorphoBorrowFuse.sol`, `MorphoCollateralFuse.sol`, `MorphoFlashLoanFuse.sol`, `MorphoSupplyWithCallBackDataFuse.sol`, `MorphoBalanceFuse.sol`, `MorphoOnlyLiquidityBalanceFuse.sol`
- Market IDs: `MORPHO = 14`, `MORPHO_FLASH_LOAN = 19`, `MORPHO_LIQUIDITY_IN_MARKETS = 41`
- Chains: Ethereum, Base
- External contracts: Morpho Blue singleton (`IMorpho`)

## Substrate Model
Substrates are **Morpho Market IDs** (bytes32), not asset addresses. Each Morpho market has a unique ID derived from its parameters (loanToken, collateralToken, oracle, irm, lltv).

## Fuse Interface

### MorphoSupplyFuse
Supplies loan tokens to and withdraws from Morpho markets. Supports instant withdraw.

```
enter(morphoMarketId, amount):
  -- Validates morphoMarketId is granted substrate
  -- marketParams = MORPHO.idToMarketParams(morphoMarketId)
  -- approve(marketParams.loanToken, amount) to MORPHO
  -- MORPHO.supply(marketParams, amount, 0, self, "")
  => returns (loanToken, morphoMarketId, assetsSupplied)

exit(morphoMarketId, amount):
  -- Validates morphoMarketId is granted substrate
  -- Accrues interest, calculates max withdrawable from shares
  -- If amount >= max: withdraws all shares
  -- Else: withdraws specified amount
  => returns (loanToken, morphoMarketId, assetsWithdrawn)
```

### MorphoBorrowFuse
Borrows and repays loan tokens. Supports both amount-based and shares-based operations.

```
enter(morphoMarketId, amountToBorrow, sharesToBorrow):
  -- MORPHO.borrow(marketParams, amountToBorrow, sharesToBorrow, self, self)
  => returns (marketId, morphoMarket, assetsBorrowed, sharesBorrowed)

exit(morphoMarketId, amountToRepay, sharesToRepay):
  -- approve(loanToken, max) to MORPHO
  -- MORPHO.repay(marketParams, amountToRepay, sharesToRepay, self, "")
  -- Reset approval to 0
  => returns (marketId, morphoMarket, assetsRepaid, sharesRepaid)
```

### MorphoCollateralFuse
Supplies and withdraws collateral tokens to/from Morpho markets.

```
enter(morphoMarketId, collateralAmount):
  -- transferAmount = min(collateralAmount, collateralToken.balance)
  -- approve(collateralToken) to MORPHO
  -- MORPHO.supplyCollateral(marketParams, transferAmount, self, "")
  => returns (collateralToken, morphoMarketId, transferAmount)

exit(morphoMarketId, collateralAmount):
  -- MORPHO.withdrawCollateral(marketParams, collateralAmount, self, self)
  => returns (collateralToken, morphoMarketId, collateralAmount)
```

### MorphoFlashLoanFuse
Executes flash loans from Morpho Blue. Enter-only (no exit). Uses callback to execute nested FuseActions.

```
enter(token, tokenAmount, callbackFuseActionsData):
  -- Validates token is granted substrate (as asset)
  -- MORPHO.flashLoan(token, tokenAmount, encodedCallbackData)
  -- Reset approval to 0 after callback
  => returns (token, tokenAmount)
```

### MorphoSupplyWithCallBackDataFuse
Like MorphoSupplyFuse but with callback support during supply. Hardcoded Morpho address: `0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb`.

## Balance Tracking

**MorphoBalanceFuse** (full balance including collateral and debt):
```
balanceOf():
  for each morphoMarketId substrate:
    collateral = MORPHO.extSloads(collateralSlot)  // via storage slot
    supply = MORPHO.expectedSupplyAssets(marketParams, self)
    borrow = MORPHO.expectedBorrowAssets(marketParams, self)
    balance += convertToUsd(collateralToken, collateral)
    balance += convertToUsd(loanToken, supply - borrow)  // can be negative
  return balance  // USD in 18 decimals
```

**MorphoOnlyLiquidityBalanceFuse** (supply-only, no collateral/debt):
```
balanceOf():
  for each morphoMarketId substrate:
    supply = MORPHO.expectedSupplyAssets(marketParams, self)
    balance += convertToUsd(loanToken, supply)
  return balance  // USD in 18 decimals
```

## Key Invariants
- Substrates are Morpho market IDs, validated via `isMarketSubstrateGranted`
- Supply exit accrues interest before calculating withdrawable amount
- Borrow repay uses max approval then resets to 0 for safety
- Collateral withdrawal reverts if it would make position unhealthy
- Flash loan must be repaid within same transaction via callback

## Risks & Edge Cases
- Morpho markets are permissionless -- oracle/IRM configuration matters
- Dust shares can accumulate; exit handles this by burning all shares when amount >= max
- Flash loan callback executes arbitrary FuseActions -- governance must vet carefully
- `extSloads` for collateral reading relies on Morpho storage layout stability

## Related Files
- Source: `contracts/fuses/morpho/`
- Tests: `test/fuses/morpho/`
- External: Morpho Blue interfaces from `@morpho-org/morpho-blue`
