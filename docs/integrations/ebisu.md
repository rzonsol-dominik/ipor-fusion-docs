# Ebisu Integration

## Quick Reference
- Fuse contracts: `EbisuZapperCreateFuse.sol`, `EbisuAdjustTroveFuse.sol`, `EbisuAdjustInterestRateFuse.sol`, `EbisuZapperLeverModifyFuse.sol`, `EbisuZapperBalanceFuse.sol`
- Adapters: `WethEthAdapter.sol`
- Libraries: `EbisuZapperSubstrateLib.sol`, `EbisuMathLib.sol`
- Market ID: `EBISU = 39`

## Overview

Ebisu is a Liquity v2-based lending protocol where users open "Troves" (collateralized debt positions) and mint ebUSD stablecoin. The integration supports leveraged Trove creation/closure via Zapper contracts, plus direct adjustment of collateral, debt, and interest rates. A WethEthAdapter handles ETH gas compensation required by the protocol.

## Fuse Interface

### EbisuZapperCreateFuse (open/close Troves)

```
enter(data: {zapper, registry, collAmount, ebusdAmount, upperHint,
             lowerHint, flashLoanAmount, annualInterestRate, maxUpfrontFee}):
  1. Validate zapper and registry are granted substrates
  2. Validate interest rate in [0.5%, 250%]
  3. Check upfront fee, min debt (2000 ebUSD), and ICR >= MCR
  4. Create WethEthAdapter if not exists
  5. Transfer ETH gas compensation (0.0375 ETH) and collateral to adapter
  6. Call adapter.openTroveByZapper()
  7. Store troveId in FuseStorageLib (keyed by zapper address)

exit(data: {zapper, flashLoanAmount, minExpectedCollateral, exitFromCollateral}):
  1. Validate zapper substrate
  2. Retrieve troveId (revert if not open)
  3. If exitFromCollateral=false, transfer ebUSD to adapter for repayment
  4. Call adapter.closeTroveByZapper()
  5. Delete troveId from storage
```

### EbisuAdjustTroveFuse

```
enter(data: {zapper, registry, collChange, debtChange,
             isCollIncrease, isDebtIncrease, maxUpfrontFee}):
  -> Adjusts collateral and/or debt of an existing Trove
  -> Approves tokens only in the direction of change
```

### EbisuAdjustInterestRateFuse

```
enter(data: {zapper, registry, newAnnualInterestRate,
             maxUpfrontFee, upperHint, lowerHint}):
  -> Changes the annual interest rate on an existing Trove
```

### EbisuZapperLeverModifyFuse

```
enter(data: {zapper, flashLoanAmount, ebusdAmount, maxUpfrontFee}):
  -> Lever-up: increases both collateral and debt via flash loan

exit(data: {zapper, flashLoanAmount, minBoldAmount}):
  -> Lever-down: decreases both collateral and debt via flash loan
```

## Substrate Configuration

Typed substrates (type in upper 96 bits):

| Type | Value | Description |
|------|-------|-------------|
| ZAPPER | 1 | Allowed Ebisu Zapper contract address |
| REGISTRY | 2 | Allowed AddressesRegistry contract address |

## Balance Tracking

`EbisuZapperBalanceFuse.balanceOf()`:
- Iterates all ZAPPER-type substrates
- For each zapper with an open troveId, reads `getLatestTroveData(troveId)`
- Calculates `sum(collateral * collPrice) - sum(debt * ebUSDPrice)`
- Returns `max(result, 0)` in USD (18 decimals)
- Interest fees are included automatically in `entireDebt`

## Key Invariants

- Each zapper can have at most one open Trove at a time
- Minimum debt is 2000 ebUSD
- ICR must be >= MCR (Minimum Collateral Ratio) at open time
- Annual interest rate must be between 0.5% and 250%
- 0.0375 ETH gas compensation is required via WethEthAdapter
- WethEthAdapter is created once and stored in ERC-7201 storage
- TroveIds are stored per-zapper in FuseStorageLib

## Related Files
- Source: `contracts/fuses/ebisu/EbisuZapperCreateFuse.sol`
- Source: `contracts/fuses/ebisu/EbisuAdjustTroveFuse.sol`
- Source: `contracts/fuses/ebisu/EbisuAdjustInterestRateFuse.sol`
- Source: `contracts/fuses/ebisu/EbisuZapperLeverModifyFuse.sol`
- Source: `contracts/fuses/ebisu/EbisuZapperBalanceFuse.sol`
- Source: `contracts/fuses/ebisu/lib/EbisuZapperSubstrateLib.sol`
