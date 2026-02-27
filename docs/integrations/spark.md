# Spark Integration

## Quick Reference
- Fuse contracts: `SparkSupplyFuse.sol`, `SparkBalanceFuse.sol`
- External interface: `ISavingsDai.sol`
- Market ID: `SPARK = 15`
- Chain: Ethereum mainnet only

## Overview

Spark (formerly Savings DAI) is an ERC-4626 vault by MakerDAO that accepts DAI deposits and returns sDAI shares that accrue yield from the DAI Savings Rate (DSR). The integration provides supply/withdraw operations and balance tracking.

## Fuse Interface

### SparkSupplyFuse

```
SparkSupplyFuse:
  DAI  = 0x6B175474E89094C44Da98b954EedeAC495271d0F
  SDAI = 0x83F20F44975D03b1b09e64809B757c47f942BEeA

  constructor(marketId)

  enter(data: {amount}) -> shares:
    1. Return 0 if amount is 0
    2. Approve sDAI contract for DAI amount
    3. Call ISavingsDai.deposit(amount, address(this))
    4. Return shares received

  exit(data: {amount}) -> shares:
    1. Return 0 if amount is 0
    2. Cap amount to ISavingsDai.maxWithdraw(address(this))
    3. Call ISavingsDai.withdraw(amount, self, self)
    4. Return shares burned

  instantWithdraw(params):
    -> Same as exit but catches exceptions for graceful failure

  enterTransient() / exitTransient():
    -> Same logic using transient storage
```

## Substrate Configuration

No substrate validation in any fuse (DAI and sDAI are hardcoded). The balance fuse reads sDAI balance directly without substrate checks.

## Balance Tracking

`SparkBalanceFuse.balanceOf()`:
- Reads sDAI balance from `ISavingsDai.balanceOf(address(this))`
- Converts to USD: `sDAI_balance * price / (10^(decimals + priceDecimals))` normalized to WAD
- Uses PriceOracleMiddleware for sDAI pricing

## Key Invariants

- Hardcoded to Ethereum mainnet DAI and sDAI addresses
- Withdrawal is capped to `maxWithdraw()` to prevent reverts
- Instant withdraw catches exceptions to allow graceful partial withdrawals
- sDAI is an ERC-4626 vault, so shares accrue value over time via DSR
- Balance fuse reports sDAI value (not underlying DAI)

## Related Files
- Source: `contracts/fuses/chains/ethereum/spark/SparkSupplyFuse.sol`
- Source: `contracts/fuses/chains/ethereum/spark/SparkBalanceFuse.sol`
- Source: `contracts/fuses/chains/ethereum/spark/ext/ISavingsDai.sol`
