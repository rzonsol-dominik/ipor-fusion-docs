# Midas Integration

## Quick Reference
- Fuse contracts: `MidasSupplyFuse.sol`, `MidasRequestSupplyFuse.sol`, `MidasBalanceFuse.sol`
- Libraries: `MidasSubstrateLib.sol`, `MidasPendingRequestsStorageLib.sol`
- External interfaces: `IMidasDepositVault.sol`, `IMidasRedemptionVault.sol`, `IMidasDataFeed.sol`
- Market ID: `MIDAS = 45`

## Overview

Midas is a Real World Asset (RWA) protocol offering tokenized treasury bills (mTBILL) and basis trade tokens (mBASIS). The integration supports both instant and asynchronous (request-based) deposit and redemption flows. Pending async requests are tracked in ERC-7201 namespaced storage for accurate NAV reporting.

## Fuse Interface

### MidasSupplyFuse (instant operations)

```
enter(data: {mToken, tokenIn, amount, minMTokenAmountOut, depositVault}):
  1. Validate mToken, depositVault, tokenIn are granted substrates
  2. Cap amount to available balance
  3. Approve depositVault, call depositInstant(tokenIn, amountWad, minAmount)
  4. Verify mTokens received >= minMTokenAmountOut
  5. Reset approval

exit(data: {mToken, amount, minTokenOutAmount, tokenOut, instantRedemptionVault}):
  1. Validate mToken, instantRedemptionVault, tokenOut are granted substrates
  2. Cap amount to available mToken balance
  3. Approve redemption vault, call redeemInstant(tokenOut, amount, minAmount)
  4. Verify tokenOut received >= minTokenOutAmount

instantWithdraw(params):
  -> Same as exit() but catches exceptions for graceful failure
```

### MidasRequestSupplyFuse (async operations)

```
enter(data: {mToken, tokenIn, amount, depositVault}):
  1. Clean up completed/canceled pending deposits for the vault
  2. Validate substrates
  3. Call depositRequest() -> returns requestId
  4. Store requestId in pending deposits storage
  5. Reset approval

exit(data: {mToken, amount, tokenOut, standardRedemptionVault}):
  1. Clean up completed/canceled pending redemptions
  2. Validate substrates
  3. Call redeemRequest() -> returns requestId
  4. Store requestId in pending redemptions storage

cleanupPendingDeposits(depositVault, maxIterations):
  -> Remove non-pending requests from storage

cleanupPendingRedemptions(redemptionVault, maxIterations):
  -> Remove non-pending requests from storage
```

## Substrate Configuration

Typed substrates (type encoded in upper 96 bits):

| Type | Value | Description |
|------|-------|-------------|
| M_TOKEN | 1 | mTBILL/mBASIS token address |
| DEPOSIT_VAULT | 2 | Midas Deposit Vault address |
| REDEMPTION_VAULT | 3 | Standard Redemption Vault address |
| INSTANT_REDEMPTION_VAULT | 4 | Instant Redemption Vault address |
| ASSET | 5 | Allowed deposit/withdrawal asset (e.g., USDC) |

## Balance Tracking

`MidasBalanceFuse.balanceOf()` sums three components:

1. **Held mTokens**: For each deposit vault substrate, read `mToken()` address and price from `mTokenDataFeed().getDataInBase18()`. Multiply balance by price. Deduplicate mTokens across vaults.

2. **Pending deposits**: For each deposit vault, iterate pending request IDs (from storage). Sum `depositedUsdAmount` of requests with status == Pending.

3. **Pending redemptions**: For each redemption vault, iterate pending request IDs. Convert `amountMToken * mTokenPrice` for requests with status == Pending.

All values are returned in USD with 18 decimals.

## Key Invariants

- Every address (mToken, vault, asset) must be registered as its correct substrate type
- Pending request tracking uses ERC-7201 storage, auto-cleaned on each enter/exit
- Async deposits: USDC leaves immediately, mTokens arrive when admin approves (push model)
- Async redemptions: mTokens leave immediately, USDC arrives when admin approves
- Balance fuse correctly reports in-transit value for accurate NAV
- Request IDs must be unique per vault (duplicates revert)

## Related Files
- Source: `contracts/fuses/midas/MidasSupplyFuse.sol`
- Source: `contracts/fuses/midas/MidasRequestSupplyFuse.sol`
- Source: `contracts/fuses/midas/MidasBalanceFuse.sol`
- Source: `contracts/fuses/midas/lib/MidasSubstrateLib.sol`
- Source: `contracts/fuses/midas/lib/MidasPendingRequestsStorageLib.sol`
